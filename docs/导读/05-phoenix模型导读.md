# 05 — Phoenix 模型导读（召回与排序）

上一篇（02）讲到 `PhoenixCandidatePipeline` 里 `PhoenixSource` / `PhoenixMoeSource` 是 For You 网络外内容的**主力召回与排序模型**。`home-mixer` 只是调用方，真正的模型代码在 `phoenix/` 这个独立仓库里。本篇钻进 `phoenix/xrex/`（JAX 训练/推理栈）与 `phoenix/crates/`（Rust gRPC 服务引擎），把"模型到底长什么样、怎么训、怎么跑"讲清楚。

> 一切结论均来自实际读到的代码（`phoenix/xrex/...`），文件名已在括号标注。未深读的部分会明确说明。

## 1. 一句话定位

**Phoenix 是 For You 的两阶段推荐模型族：召回用双塔把数百万候选压到数百，排序用"候选互相不能注意"的 transformer 对召回结果打分排序。** 整套栈是生产代码的直接导出（`README.zh-CN.md:5` 与 `README.zh-CN.md:32`），模型定义、训练器、检查点、Rust 服务引擎都在仓库里；唯一被替换掉的是 xAI 私有基础设施（生产数据源、集群编排、遥测），改用本地等价实现 + 合成数据生成器（`README.zh-CN.md:7`）。

承接 02 篇的位置关系：

```
home-mixer (Rust 调用方)
   └─ PhoenixSource ──► [召回双塔] ──► [排序 Transformer] ──► 多目标概率
                        phoenix/ 提供这两个模型 + 服务引擎
```

## 2. 目录结构与文件职责

`phoenix/` 顶层是 Rust 工程（`Cargo.toml`、`crates/`），Python 训练/推理栈全在 `xrex/` 下。关键文件：

| 路径 | 职责 |
|---|---|
| `xrex/models/recsys_model.py` | **排序模型** `RecsysAggregatedModel`（transformer 主干 + 多目标 head），3557 行，最核心 |
| `xrex/models/recsys_two_tower_model.py` | **召回双塔** `RecsysTwoTowerModel`：用户塔 + 候选塔 + 对比损失，1778 行 |
| `xrex/models/recsys_attention.py` | 排序专用注意力核，**实现候选隔离掩码**（block-sparse 布局） |
| `xrex/models/recsys_embedding.py` | **哈希嵌入表**：多哈希、无字典服务、确定性 |
| `xrex/models/recsys_feature_prep.py` | 特征准备：ID/SID/上下文 → token 嵌入 |
| `xrex/models/loss_recsys.py` | 多标签 BCE、连续值（停留）MSE/MAE/Huber、Tweedie 损失 |
| `xrex/models/recsys_sid.py`、`recsys_sid_retrieval_model.py` | 语义 ID（SID）相关模型（本篇未深读） |
| `xrex/models/recsys_gen_recs_model.py` | 生成式推荐（gen-recs），与双塔/排序并列的另一条线 |
| `xrex/configs/xrecsys.py` | 排序生产/ nano 配置（`home_direct_packed`、`home_direct_packed_nano`） |
| `xrex/configs/xrecsys_two_tower.py` | 双塔召回配置 |
| `xrex/data/parquet_recsys.py` | 离线 Parquet 数据加载 `PhoenixDataset` |
| `xrex/data/conversion_labels.py` | 延迟反馈 → 多热标签（转化窗口） |
| `xrex/data/retrieval_dataset.py` | 召回候选语料（SID 快照）加载 |
| `xrex/train/trainer_recsys.py`、`trainer_sid_retrieval.py`、`trainer_gen_recs.py` | 各模型训练器（`RecsysTrainer` 等） |
| `xrex/inference/` | 服务侧：`model_runner.py`、`launch_inference.py`、`serving_services.py`、`gen_recs_runner.py` |
| `reference/` | 合成世界生成（`world_snapshots.py`、`dump_gen.py`）+ 训练启动器（`train_synth.py`）+ 端到端客户端（`retrieve_then_rank.py`） |
| `crates/` | Rust 服务引擎（gRPC），通过 `uv sync --extra engine` 构建 |

## 3. 模型架构拆解

### 3.1 排序模型：带"候选隔离"的 Transformer

排序模型就是 `RecsysAggregatedModel`（`recsys_model.py:1434`，`hk.Module`）。一次前向（`__call__`，`recsys_model.py:2779`）的流程：

```
input_embeddings [B, T, D]   ← build_inputs 拼出（见 3.3）
        │
   transformer（自定义注意力核，带 segment_ids / positions）
        │
   final_layer_norm（rms_norm）
        │
   取候选位置输出（candidate_start_offset 或 seqpack 的 candidate_positions）
        │
   decode → logits [B, C, num_actions]      ← 每个候选、每个动作一个 logit
   decode_continuous → continuous_predictions ← 连续值（停留）回归 head
```

关键事实（均来自代码）：
- 输出 `logits` 形状为 `[B, num_candidates, num_actions]`（`recsys_model.py:2846` + `output_vocab_size`），另有一个独立的 `continuous_predictions` 回归头（`recsys_model.py:2855`）。即**多标签多目标 + 连续目标**。
- 候选输出是"在用户+历史之后、按位置截取"得到的（`recsys_model.py:2843`：`out_embeddings[:, candidate_start_offset:, :]`），说明用户/历史在前、候选在后，候选位置的结果就是它们的打分。

**候选隔离注意力掩码**（`recsys_attention.py`）是整个排序模型最精巧的地方。注意力核 `PallasRankerAttention`（`recsys_attention.py:23`）把序列切成三段，用 `bound` 描述块稀疏布局：

```text
history_upper   = num_user_prefix_tokens + history_seq_len      (recsys_attention.py:43)
candidate_lower = history_upper
candidate_upper = q.shape[1]
bound = (0, history_upper, candidate_lower, candidate_upper)
```

对应 README 的注意力表（README.zh-CN.md:96）：

| Query ↓ ＼ Key → | 用户前缀 | 历史（S） | 候选（C） |
| --- | --- | --- | --- |
| **用户前缀** | ✓ | ✓ | ✗ |
| **历史（S）** | ✓ | ✓ | ✗ |
| **候选（C）** | ✓ | ✓ | 仅对角线（自注意） |

也就是说：**候选可以注意用户与历史，但候选之间只能注意自己**。`bound` 这个四元组正是把这个"候选互不看"的约束编码进 FlashAttention 的块稀疏调度里（而非用一张显式 0/1 mask 矩阵）。这样做的目的（README.zh-CN.md:71）是**批不变性**：一个候选的分数不依赖同批里还有哪些候选。

代码里还看到几个面向硬件的注意力实现变体：`PallasRankerAttentionInference`（推理，H100 走 FA3）、`PallasRankerVarlenAttention`（序列打包变长）、`CutedslRankerAttention` / `CutedslRankerVarlenAttention`（FA4，要求 `qk_norm=True`、A100/H100/GB200/GB300，`recsys_attention.py:197` 起）。注意它们都 `assert not self.config.causal`：**排序模型用非因果注意力**。

### 3.2 召回模型：双塔

召回是 `RecsysTwoTowerModel`（`recsys_two_tower_model.py:857`），由两个塔组成：

- **用户塔** `user_tower: RecsysAggregatedModel`（`recsys_two_tower_model.py:859`）——**和排序模型用的是同一个类**，只是最后不接多目标 head，而是把用户序列池化成一段向量（`__call__` 里用 `segment_sum` 对打包序列做池化，`recsys_two_tower_model.py:1247`）。
- **候选塔** `RecsysCandidateTower`（`recsys_two_tower_model.py:90`）——把"帖子哈希嵌入 + 作者哈希嵌入 + SID 嵌入"组合成候选向量，并 L2 归一化（`_l2_normalize_candidates`，`recsys_two_tower_model.py:155`）。

候选塔有三种组合方式（`__call__`，`recsys_two_tower_model.py:232`）：
1. `enable_linear_proj` → `_concat_then_mlp`：拼接后接两层 MLP（SiLU 激活，`recsys_two_tower_model.py:152`）。
2. `use_project_then_sum`（排序共享的"先投影再求和"）→ `_project_then_sum`：每个哈希/SID token 各自线性投影后相加（`recsys_two_tower_model.py:218`）。
3. 否则 → `_mean_pool` 简单平均。

召回的打分就是**用户向量与候选向量的点积**（带温度 `temperature`，默认 init 0.1，`recsys_two_tower_model.py:1511`），在 `compute_retrieval_loss` 里用 `shard_map` 做分片大矩阵乘（`recsys_two_tower_model.py:556` 的 `_sharded_full_matmul`）。

**共享架构**这一点值得点出：召回用户塔和排序主干是同一份 transformer 代码，差异只在 head（README.zh-CN.md:120 的说法在代码中成立——用户塔字段类型就是 `RecsysAggregatedModel`）。

### 3.3 输入是怎么来的：哈希嵌入 + 特征准备

两个模型都不用字典服务，而是**对原始 ID 做多哈希映射进一张共享嵌入表**（`recsys_embedding.py`）：

- `HashKeys`（`recsys_embedding.py:176`）为 user / item / author / ip 各定义多组 `scales`、`biases`、`modulus`（例如 user 用 2 个哈希：`user_hash_scales = [196742702, 1852108266]`）。
- `_hash_ids_batch`（`recsys_embedding.py:154`，numba JIT）计算 `raw = (id*scale + bias) % modulus`，再映射到桶；ID 为 0（缺失）时落桶 0。
- `get_user_hash` / `get_author_hash` / `get_item_hash` / `get_ip_hash` 各自加一个**偏移量**（`offset_user/offset_item/offset_author/offset_ip`，`recsys_embedding.py:235` 起），把不同实体的桶空间拼进同一张表，避免冲突。

这印证了 README 的"基于哈希的嵌入"设计（README.zh-CN.md:114）：**确定性、无字典、靠多哈希容忍冲突**。

特征准备 `FeaturePrepConfig`（`recsys_feature_prep.py:142`）则决定每条 token 带哪些信号。这里要**诚实**地说一句：它**不是"完全无特征工程"**——而是在"原始 ID 哈希成嵌入"为主的基础上，叠了一层薄薄的、可学习的上下文嵌入：用户画像（国家/语言/性别/年龄段/已装应用/位置，`enable_user_*`）、帖子年龄（`enable_post_age`，按分钟分桶）、产品界面（`enable_product_surface`）、时区、小时、停留时长（`enable_dwell_time`，`dwell_time_norm_scale=30.0`），以及语义 ID（`enable_post_sid`，`sid_num_levels=6`、`sid_codebook_size=256`）。其中时间特征用了循环核（box/triangle/cosine，`_cyclic_kernel_weights`，`recsys_feature_prep.py:90`），小时用了小基数嵌入（`hour_of_day_cardinality=25`）。这些都不是手调权重，而是**可学习的嵌入查表 + 桶化/循环编码**，比传统手工交叉特征轻得多，但也不能说"零特征工程"。

## 4. 训练管线（数据 → 特征 → 训练 → 评估 → 导出）

训练路径是单主机、离线验证用的（TRAINING.zh-CN.md:7 明确"不含生产数据源/多机编排"）。

**数据侧**：
- 离线 dump 由 `reference/dump_gen.py` 生成，排序与双塔共用同一份（`TRAINING.zh-CN.md:27`）。
- `PhoenixDataset`（`parquet_recsys.py:849`）负责按分区读取 Parquet，`InterleavingRecordBatchProvider` / `LazyRecordBatchIterator` 做多线程预取（`parquet_recsys.py:244`、`:136`）。
- 标签处理在 `conversion_labels.py`：**延迟反馈**被折叠成多热标签（`fold_action_delays_into_multihot`，`conversion_labels.py:85`；带 `window_ms` 转化窗口、`action_delay_columns` 取各动作的延迟列）。这正是多标签目标里"正样本可能迟到的延迟反馈"的工程解法。

**损失侧**（来自代码，非 README 推断）：
- 排序：`loss`（`recsys_model.py:2860`）读 `batch["candidate_seq"]["actions"]`（多热目标）和 `continuous_actions`（停留）。实际损失函数 `multihot_loss_compute`（`loss_recsys.py:15`）就是 **sigmoid 二分类交叉熵**（每个动作独立），支持样本权重与 log-Q 校正（`recsys_model.py:2954` 的 `log_q_correction`）；连续头用 `continuous_loss_compute`（`loss_recsys.py:52`，MSE/MAE/Huber 可选）。
- 召回：`compute_retrieval_loss`（`recsys_two_tower_model.py:520`）组合**批内负样本**（`use_in_batch_negatives`）与**采样全局负样本**（`num_global_negatives_per_example`），做 log-Q 校正（`_apply_logq_correction`，`logq_correction_scale=2.0`，`recsys_two_tower_model.py:1583`），正样本信号来自 `positive_actions`（点赞一类）。支持按 head 拆分动作（`get_per_head_actions`，`recsys_two_tower_model.py:1682`）。

**优化器侧**（TRAINING.zh-CN.md:13）：
- 排序生产/nano 用 **Muon**（`optim="muon"`，`xrecsys.py:312`），双塔/gen-recs 用 **标准 AdamW**。
- 嵌入表单独用**行级 AdaGrad**（`RecsysRowwiseAdagradConfig`），且每步先对 batch 用到的嵌入行去重再查表（`trainer_recsys.py:_get_embedding_hash_leaves` / `_lookup`，`:684` / `:649`），梯度有限性检查后跳过坏步。

**评估侧**：指标函数就定义在 `recsys_model.py` 里——`metric_rce`（RCE）、`metric_prauc`、`metric_ndcg`、`metric_calib`（校准）、`compute_recsys_metrics`（`:1639`）等，模型前向同时算指标。召回侧另有 `_compute_retrieval_metrics` / `_compute_recall_at_k`（`recsys_two_tower_model.py:304`、`:520`）。

**导出侧**：召回检查点把候选语料嵌入成 `post_embeddings` **存进检查点内部**（`README.zh-CN.md:62`），服务时直接加载，省去启动时的语料嵌入；检查点写入 `<out>/<config>/elapsed_samples_<n>/<run_id>/`（`TRAINING.zh-CN.md:65`）。

## 5. 推理 / 服务侧

服务栈由 `launch_inference.py` 启动，按 `--service_type` 区分 `retrieval` / `ranking`（QUICKSTART.zh-CN.md:118 起）。关键抽象：

- `BaseModelRunner(RecsysTrainer)`（`model_runner.py:485`）——**训练器即服务运行器**：复用同一份 `RecsysTrainer` 来加载检查点、构建模型、做 JIT 前向。这意味训练与服务共用一套模型定义，部署时不另写推理图。
- `GenRecsModelRunner`（`gen_recs_runner.py:35`）是生成式推荐的对应 runner（`reply_request` / `gather_embeddings_and_forward`）。
- 排序推理走 `PallasRankerAttentionInference`（`recsys_attention.py:93`），候选隔离布局同样生效，保证**服务时分值与批次组成无关**（README.zh-CN.md:135）。

**调用契约**（承接 02 篇）：`home-mixer` 的 `PhoenixScorer` 通过 gRPC 调排序服务，拿到 `[B, C, num_actions]` 的多目标概率 + 停留回归值，再喂给 `RankingScorer` 加权聚合。召回端则另有 **SID 查询服务**（`reference/sid_index_server.py`，QUICKSTART.zh-CN.md:115）在推理时补全历史帖子的语义 ID——召回候选由 SID + 哈希作者 ID 表示，而非单纯帖子 ID（README.zh-CN.md:61），从而对新帖子有组合泛化能力。

端到端闭环由 `reference/retrieve_then_rank.py` 演示：把用户历史发给召回服务取 top-K，再让排序服务对同一批候选打分（QUICKSTART.zh-CN.md:146）。**这正是生产所组合的同一套 gRPC 契约**。

> 说明：`crates/` 下的 Rust gRPC 引擎本篇未逐文件深读，仅确认它是 `uv sync --extra engine` 构建并链接 `libibverbs`（生产嵌入传输层）的真实服务引擎（README.zh-CN.md:176）。

## 6. 本地可跑通的部分（对上手最有价值）

整条链可在**无 GPU 集群、无 Kafka、无生产数据**下端到端跑通，靠的是合成数据生成器（`README.zh-CN.md:34`）。完整流程（QUICKSTART.zh-CN.md）：

```bash
# 0. 装依赖：uv sync --extra engine；另需 Rust 工具链 + protoc>=3.15
# 1. 造"合成世界"：SID 快照 + 多模态快照 + 候选语料
uv run python reference/world_snapshots.py --out ./synth_index --seed 20260721
export PHOENIX_INDEX_BASE=./synth_index
# 2. 造训练 dump（排序与双塔共用）
uv run python reference/dump_gen.py --out ./synth_dump --seed 20260721 \
  --num-rows 12288 --partitions 4 --rows-per-file 1024 \
  --sid ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --self-check
# 3. 训练 nano 排序（默认配置 home_direct_packed_nano）
uv run python reference/train_synth.py --data ./synth_dump --steps 6 --out "$PWD/checkpoints" --metrics
# 4. 训练 nano 双塔召回
uv run python reference/train_synth.py --config xrecsys_two_tower_nano_offline_kafka_dump \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints"
# 5. 起 SID 服务 + 召回服务(9990) + 排序服务(9988)，再跑闭环
uv run python reference/retrieve_then_rank.py --data ./synth_dump \
  --sessions 3 --topk 16 --retrieval-port 9990 --ranking-port 9988
```

几个对读者有用的具体数字（来自代码/配置，非估算）：
- 排序 nano：`emb_size=512`、`num_layers=4`、`history_seq_len=1022`、`candidate_seq_len=64`、`optim="muon"`（`xrecsys.py:415` 起）；生产母版 `emb_size=2560`、`num_layers=8`、`use_seqpack=True`（`xrecsys.py:246` 起）。
- 召回 nano：`emb_size=512`、`max_posts=65536`（`RecsysCandidateModelConfig.max_posts`，`recsys_two_tower_model.py:249` 默认 10.24M，nano 覆盖为 65536）；生产 `max_posts=10_240_000`。
- 6 步合成训练仅用于"确认链路能跑通"，**不代表模型质量**（QUICKSTART.zh-CN.md:52、`:157`）。恢复训练只需同 `--out` + 更大 `--steps`，训练器自动发现最新检查点（QUICKSTART.zh-CN.md:63）。

想先验证安装不报错的最快路径：`uv run python xrex/inference/oss_bench/bench.py --smoke --service_type ranking`（QUICKSTART.zh-CN.md:27），用随机权重起真实服务栈发一个合成请求。

## 7. 可以带走的工程经验

1. **"批不变"是用注意力掩码硬做出来的**。排序候选互不注意（`recsys_attention.py` 的 `bound` 块稀疏布局），召回把候选索引塞进检查点（README.zh-CN.md:62）——两者合起来保证"同一候选的分数只取决于用户和该候选本身"。这是把"serving 与 training 一致性""batch 无关性"当成一等约束来设计的范例。

2. **无字典的哈希嵌入是大规模推荐的刚需**。多哈希 + 偏移量共享一张表（`recsys_embedding.py`），确定性、可容忍冲突、无需在线字典服务。自己系统要上亿级 ID 时，这套比 embedding 字典服务省心得多。

3. **训练器即服务运行器**（`BaseModelRunner(RecsysTrainer)`，`model_runner.py:485`）。训练和线上推理共用同一份模型定义与检查点格式，从根上消除"训推不一致"。

4. **延迟反馈用转化窗口折叠成多热标签**（`conversion_labels.py:85`）。多标签目标里正样本会迟到，这套把延迟信号转成带时间窗的标签，是工业推荐可抄的标注手法。

5. **同一份 transformer 主干，靠 head 区分任务**。召回用户塔和排序模型是同一个 `RecsysAggregatedModel`（`recsys_two_tower_model.py:859`），差异只在 head 与池化方式——共享主干能最大化复用与一致性。

6. **nano 孪生版是给读者的礼物**：保留生产的损失、检查点、服务契约，只缩宽度/深度/表规模（`README.zh-CN.md:33`）。想读透一个大模型，先拿 nano 在单 GPU 上跑通比硬啃生产配置高效得多。

> 未深读、需读者自行确认的部分：`recsys_sid.py` / `recsys_sid_retrieval_model.py`（语义 ID 量化与 SID 检索模型）、`recsys_gen_recs_model.py`（生成式推荐）、`crates/` 下 Rust 服务引擎的具体实现、以及 `xrex/eval/` 目录的完整评测管线。本篇结论均限于已读代码。
