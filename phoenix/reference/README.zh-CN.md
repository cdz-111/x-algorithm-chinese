> **说明**：本文件是 [README.md](README.md) 的中文译本。如与英文原文有出入，以英文原文为准；代码、命令、路径与标识符均保持原样。

# reference/ — 合成数据、参考提供器，以及可读的训练路径

生产环境的 Phoenix 由无法对外发布的基础设施驱动：一条训练会话的 Kafka 流、一个 MM 嵌入服务、一个 Semantic-ID（SID）服务，以及它们产生的检查点。本目录是这一切的公开替代方案——一个带种子的合成世界、一组能产出已发布服务/训练栈所读取的全部产物的生成器、两个提供器服务的参考实现，以及可读的单一设备训练步骤。这里的一切都可以在零生产数据的情况下端到端运行。

⚠ **参考数据，非生产数据。** 这些合成产物与对应的生产产物具有完全一致的 schema、布局与接口契约（wire contract），但其*取值*是确定性的占位替身。下方的目录映射对每个文件说明它是生成占位数据，还是算法移植 / 真实镜像源码——该映射为准（已发布的源码不带注释或 docstring 发布，因此文件内不会有横幅说明这一点）。

## 目录映射

**合成世界及其生成器**

| 文件 | 说明 |
| --- | --- |
| `world.py` | 所有生成器所依赖的带种子世界：主题、作者、用户、帖子，以及一个内置的互动模型。一个种子 → 一个世界 → 彼此一致的产物。 |
| `dump_gen.py` | 离线训练 dump 生成器——Kafka 的替代。以已发布的 `aggregated_kafka` 读取器所消费的完全相同的布局 + schema，将用户会话快照写为 parquet。 |
| `world_snapshots.py` | 从同一个世界产出三个提供器快照（MM、SID、帖子创建），全部以相同的帖子 id 作为键。其输出目录即为现成的 `PHOENIX_INDEX_BASE`。 |
| `gen_recs_artifacts_gen.py` | 从同一个世界产出 gen-recs 离线分支的配套产物：内存映射的 MM 查找表（`post_ids.npy` + `embeddings.npy`）、评测打分表（`inference_posts.parquet`）以及全局负样本池（`global_ids.parquet`）。 |
| `oss_recsys_synth.py` | 合成路径底层的确定性取值生成器：随机傅里叶特征 MM 嵌入与随机码本 RQ 码，同时支持服务端接口契约与模型缓冲区两种约定。 |

**MM 嵌入（帖子 → 1024 维向量）**

| 文件 | 说明 |
| --- | --- |
| `mm_encoder.py` | 忠实的 v5 嵌入算法：渲染器 + ChatML 封装 + MRL 截断 / L2 归一化，逐字移植。仅含代码——由读取方提供官方公开的 `Qwen/Qwen3-VL-Embedding-8B` 权重（或 SGLang 端点）。 |
| `mm_snapshot_gen.py` | 写出 MM 嵌入快照 parquet。默认：合成占位向量。可选开启：通过 `embedder=mm_encoder.embed_post` 使用真实 v5 向量。 |
| `example_data/` | 极小的自包含帖子样例（文本 / 图片 / 引用 / 视频），用于在无网络或真实媒体的情况下驱动渲染器。 |

**Semantic ID（帖子 → 6 级 RQ 码）**

| 文件 | 说明 |
| --- | --- |
| `sid_codebook.py` | RQ-KMeans / RQ-VAE 码本**训练器**（JAX），从内部流水线源码镜像而来。CLI：`train` / `evaluate` / VAE 变体。 |
| `sid_assign.py` | SID **分配器**，从内部流水线源码镜像而来：MLP 编码 → 对训练后的码本做残差量化。 |
| `sid_io.py` | 上述两个文件的共享 IO（码本 `.npz` 格式、MLP 前向），从内部流水线源码镜像而来。 |
| `sid_snapshot_gen.py` | 写出帖子 SID 快照 parquet。默认：合成随机码本码。可选开启：使用由 `sid_codebook.py` 训练得到的码本产出的码。 |
| `sid_index_server.py` | **真实** SID 查找服务器：通过生产环境的 `SidLookupService` gRPC 接口契约，从一个快照 parquet 提供码。 |
| `sid_mock_server.py` | 独立的 SID 模拟服务器：相同的接口契约，码即时生成（无需快照）。 |
| `_sid_proto/` | `sid_lookup.proto` 的已生成 gRPC 桩（来源：`crates/serving/xai-recsys-sid-proto/proto/`）。 |

**训练、检查点与服务**

| 文件 | 说明 |
| --- | --- |
| `train_step.py` | 标准的单一设备训练步骤——已发布的组合（稠密优化器 + 稀疏行级 AdaGrad；本参考实现组合了 AdamW 分支，而排序旗舰 / nano 配置训练的是已发布的 Muon 配方），并去掉了分片基础设施。参见 [`TRAINING.md`](../TRAINING.md)。 |
| `train_synth.py` | 公开启动器：将已发布的训练器配置指向一个 `dump_gen.py` 生成的 dump，并端到端训练 nano 模型。 |
| `repack_checkpoint.py` | 将一个训练好的检查点重新打包为可发布的产物：保留加载 / 推理张量，丢弃优化器状态，清除内部元数据，重新生成校验和。 |
| `retrieve_then_rank.py` | QUICKSTART 第 5 节的驱动程序：通过生产环境的 gRPC 接口契约，将真实 dump 会话依次送入两个在线服务器——召回阶段用 `RetrieveTopKCandidates`，排序阶段用 `PredictNextActions`。 |

出处，逐条说明：`sid_codebook.py`、`sid_assign.py` 与 `sid_io.py` 是真实的内部 SID 流水线源码，在导出时仅剥离了基础设施；`train_step.py` 与 `mm_encoder.py` 是生产算法的忠实移植；其余皆为本次发布编写的参考工具。

## 各组件如何关联

```
                       world.py  (one seeded world)
                          │
             ┌────────────┴──────────────┐
             ▼                           ▼
        dump_gen.py ◄─── --sid ──  world_snapshots.py
             │          (SID codes)     │
             ▼                           ▼
       offline dump              PHOENIX_INDEX_BASE/
        partition=*/...            mm_snapshot/post_mm_v5.parquet
        .valid_batches.json        sid_snapshot/post_sid_v5_256x6.parquet
        world/*.parquet            sid_snapshot/codebook_v5_256x6.npz
             │                     post_creation_snapshots/post_creation_1day.parquet
             ▼                           │
      train_synth.py                     ├──► serving stack (reads the snapshots)
      (train_step.py inside)             └──► sid_index_server.py --parquet ...
             │                                 ▲ gRPC (SidLookupService)
             ▼                                 │
        checkpoint ──► repack_checkpoint.py    │
             │                                 │
             ▼                                 │
        inference (loads checkpoint, hydrates SIDs from the live server)
```

`world_snapshots.py` 组合了其他生成器——通过 `mm_snapshot_gen.write_mm_snapshot` 产出 MM 向量，通过由 `sid_codebook.py` 训练、由 `sid_assign.py` 分配的码本产出 SID 码——因此其三个产物描述的是同一批帖子，且 SID 码确实对与之相邻的 MM 向量做了量化。随后 `dump_gen.py --sid` 将这些相同的码读入 dump，这正是快照要先生成的原因：模型是在 SID 服务器之后将为同一批帖子提供服务的那些 semantic ID 上训练的。

## 三个提供器的来龙去脉

**训练数据。** 生产环境的训练从 Kafka 流式获取用户会话快照的 Arrow record batch。`dump_gen.py` 将这些相同的 batch 写为 parquet（`partition={p}/<bucket>/batch_{b}.parquet`，外加一份 `.valid_batches.json` 清单），使已发布的 `aggregated_kafka` 读取器原样消费。会话从世界的互动模型中采样，因此内置的结构（主题亲和度、作者质量）是可学习的——在 dump 上训练的模型会产出有意义的排序推荐，而非噪声。dump 还带有 `world/*.parquet` 附带表（用户、帖子、作者），便于你解码模型所见的内容。

**MM 嵌入。** 生产环境运行一个服务，将每个帖子（文本 / 图片 / 视频）转化为一个 1024 维单位向量（"v5"）。发布版本提供两条路径：

- *忠实路径：* `mm_encoder.py` 是真实的流水线——将帖子渲染为一个带有位置感知 image-pad 标记的字符串，用 v5 system prompt 做 ChatML 封装，用官方公开的 `Qwen/Qwen3-VL-Embedding-8B` 编码，截断 4096 → 1024，再做 L2 归一化。重型依赖为惰性加载；通过 `mm-encoder` extra 安装（参见仓库根目录的 `pyproject.toml`）。
- *合成路径：* `oss_recsys_synth.synth_mm_embeddings` 用带种子的随机傅里叶特征 sin 映射将帖子 id 映射为单位向量。schema 相同、单位范数约定相同、零重型依赖——这正是 `world_snapshots.py` 与 CI 所用的。

**Semantic ID。** 生产环境运行一个服务，将每个帖子映射为一个 6 级残差量化码（每一级取值在 `[0, 256)` 内），该码由其 MM 嵌入计算得出。发布版本包含*完整*的闭环：

- *训练：* `sid_codebook.py train --training-data emb.npy --codebook-out cb.npz`（真实的镜像训练器，RQ-KMeans 或 RQ-VAE）。
- *分配：* `sid_assign.py` 对嵌入做编码，对照训练后的码本进行量化。
- *快照：* `sid_snapshot_gen.write_sid_snapshot` 写出 `post_id → post_sid` 的 parquet——默认合成码，通过 `codes=` 可使用训练后码本的码。
- *在其上训练：* `dump_gen.py --sid <snapshot>` 将这些码复制进 dump 的 `semanticIdSeq` 列。若没有这一步，该列就不存在，而由于启用 SID 的配置仅依据 `sid_num_levels` 来确定缓冲区大小，训练会把每个帖子都读作"无 SID"——不会报错，也不会向 SID 表回传梯度。
- *服务：* `sid_index_server.py --parquet <snapshot>` 从那份 parquet 应答生产环境的 `SidLookupService` gRPC 接口契约；`sid_mock_server.py` 用即时生成的合成码应答。服务引擎的 `PySemanticIdClient` 与二者都能通信，无需改动。

通信约定说明：服务器使用 0 起始的码（`[0, 256)`）；模型的输入缓冲区是 1 起始的 `uint16`（`0` = 缺失）。`oss_recsys_synth.py` 同时实现了二者（`sid_codes_for_posts` 与 `synth_post_sids`）并记录了该偏移。

## 端到端合成快速上手

在仓库根目录执行（需先 `uv sync --extra engine`——第 2 步经由已发布的训练器进行训练，后者会导入 Rust 引擎的 Python 模块；参见 [`QUICKSTART.md`](../QUICKSTART.md)）：

```bash
# 1. 一个带种子的世界 → 提供器快照，然后是训练 dump。
#    快照先生成：dump 会复用其中已分配的 SID 码，因此二者
#    以相同的方式描述每个帖子。
python reference/world_snapshots.py --out ./synth_index --seed 20260721 --self-check
export PHOENIX_INDEX_BASE=./synth_index
python reference/dump_gen.py  --out ./synth_dump  --seed 20260721 \
  --num-rows 12288 --partitions 4 --rows-per-file 1024 \
  --sid ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --self-check

# 预览几条已解码的会话（人类可读，使用世界的帖子文本）。
python reference/dump_gen.py --out ./preview_dump --seed 20260721 --preview 3

# 2. 在该 dump 上训练 nano 排序模型（写入 ./checkpoints/...）。
python reference/train_synth.py --data ./synth_dump --steps 500 --out ./checkpoints

# 3. 由世界产出的快照提供 SID 服务（独立终端）。
python reference/sid_index_server.py \
  --parquet ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --port 50061
```

[`TRAINING.md`](../TRAINING.md) 从第 2 步继续：加载回检查点并对其运行真实推理，外加逐组件的算法映射。没有可下载的预训练检查点；按此 dump 配方训练 nano 在一块 GPU 上只需数分钟。

这里的每个模块也都可以作为自身的自检查直接运行：工具会解析 `--help` / `--self-check`，而库（`world.py`、`mm_encoder.py`、`mm_snapshot_gen.py`、`sid_snapshot_gen.py`、`oss_recsys_synth.py`）在裸执行时会运行其断言，例如 `python reference/world.py`。

## 确定性

相同的种子 → 相同的世界 → 逐字节一致的产物，跨运行、跨机器均如此：生成器使用显式设定种子的 NumPy 生成器、固定的行顺序，以及固定的 parquet 写入器设置。唯一的例外：`world_snapshots.py` 在 GPU 上用 JAX 训练其 SID 码本，而 k-means 归约在逐次运行间可能在最后一个浮点位上抖动——`codebook_v5_256x6.npz` 的质心可能在约 1e-8 量级上有所不同，但包括已分配 SID 码在内的所有 parquet 产物都保持逐字节一致。各处的默认种子为 `20260721`；传入 `--seed` 可生成不同的世界。
