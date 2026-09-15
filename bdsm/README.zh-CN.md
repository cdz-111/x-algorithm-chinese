> **说明**：本文件是 [README.md](README.md) 的中文译本。如与英文原文有出入，以英文原文为准；代码、命令、路径与标识符均保持原样。

# 行为检测序列模型（BDSM）

一个动作序列 transformer，从行为事件流中检测非真实行为（机器人 / 垃圾信息 / 协同）账号，同时附带任务头（task-head）训练栈与一套参考用的流式打分（scoring）流水线。

## 目录结构

```
bdsm/
├── runtime/     主干、任务头、打分流水线、结果汇聚
├── training/    在缓存的主干激活值上训练任务头
├── proto/       打分事件的 Protobuf 定义
└── rust/        Rust 组件（Kafka 累加器、PyO3 特征提取器）
```

## 模型

`runtime/model.py` —— 一个作用于用户近期动作序列的双向 transformer 编码器，包含：

- **时间感知 RoPE**：由归一化动作时间戳而非 token 索引驱动的旋转位置嵌入（rotary position embeddings），使模型能够原生表示动作间时序（突发性、机械式节奏）。
- **分组查询注意力**（Grouped-query attention）、RMSNorm 与 SwiGLU。
- **逐动作特征**（动作类型、产品界面、停留时长、设备/客户端信号、互动目标哈希等），由 Rust `abuse-v3-features` 提取器从原始 Arrow IPC 序列字节中提取，并经 `runtime/feature_norm.py` 改写为训练期输入格式。
- **八个任务头**（`runtime/heads.py`）：FollowBot、LikeBot、EngagementAmplifier、ReplySpamBot、TweetSpamBot、RTBot、MultiActionBot 与 LegitimateUser。采用类别平衡 BCE 加上对称的 reverse-CE 抗噪项进行训练，focal 权重接入损失函数（`runtime/loss.py`）。

主干（backbone）在服务（serving）时冻结。任务头（`runtime/task_heads.py`）是基于主干 CLS 嵌入的 MLP，在缓存的激活值上单独训练（`training/train_head.py`），并通过 sha256 与主干导出一并加载（`runtime/load_backbone.py`）。权重以 npz + 每数组 sha256 的形式随 `MANIFEST.json` 一同提供——不存在 pickle 反序列化面。

## 训练

1. **主干**：在未经标注的动作序列上的自监督编码器（每步进行掩码属性预测；每第 6 步进行一次双视图 InfoNCE 对比目标）。公开产物为导出的 `backbone.npz`。
2. **任务头**（`training/train_head.py`）：在八个头上的掩码多标签 SCE，在缓存的 CLS 嵌入上训练。非有限的 loss 或梯度属于硬性中止（绝不置零后继续）。Focal gamma 是带输出统计的真实参数，在第 0 步即被断言存在。

## 运行时流水线

```
Kafka（用户动作事件）
  → rust/accumulator          去重 + 冷却，将 user_ids 入队到 Redis
  → runtime/batch_prefetcher  获取序列（缓存优先，存储回退），
                              Rust 特征提取，
                              将可送 GPU 的批次发布到 Kafka
  → runtime/gpu_scorer        feature_norm，在 GPU 上进行 JAX 推理，
                              发布 8 列的分值行
  → runtime/score_results_sink_focal
                              逐头阈值，去重/冷却账本，
                              分级处置（质询 vs. 停用），
                              BigQuery + protobuf 事件输出
```

`proto/abuse_inference.proto` 定义了各阶段之间交换的事件（`ScoreRequest`、`ScoreResult`、`FiredHead` 等）。

打分器（scorer）以 `heads.HEAD_NAMES` 的顺序发布一个 8 列的行。

## 依赖与注意事项

- Python ≥ 3.10；参见 `pyproject.toml`。GPU 推理/训练还需具备 CUDA 的 JAX、`optax` 以及 `haiku2` 神经网络库。
- Rust 特征提取器（`rust/abuse-v3-features`）构建为 PyO3 扩展模块（`abuse_v3_features`）。服务（serving）会固定使用生成训练期特征的提取器；请从本代码树重新构建，或提供匹配的 `.so`。
- 该流水线通过 gRPC sidecar 从内部键值存储读取动作序列，并通过 HTTP 读取账号元数据。为完整性起见，相关客户端已包含在内，但其后端服务不属于本次发布；请将取数层（`runtime/manhattan_scorer.py:ManhattanArrowReader`）适配到你的数据源。
- 预取器的序列获取以缓存优先（`runtime/sequence_cache.py`）：对历史序列缓存执行 Redis GET（默认启用，可用 `--no-seq-cache` 关闭），并回退到序列存储及尽力而为的读穿式填充；另含一个可选的实时动作合并（`--realtime-cache-enabled`），它会从 Redis 有序集合缓存追加尚未刷出的动作，并在特征提取前对序列重新排序。这两个缓存均假定可通过本地 Envoy redis-proxy 监听器访问；任何缓存失败都会降级到普通存储路径。
- 内部序列存储 schema 的生成式 protobuf 绑定未包含在内；导入 `proto_gen.recsys_pb2` 的代码路径会优雅降级，或需要为你的 schema 重新生成绑定。
- 默认值中的主机名、broker 地址与项目名称均为占位符（`localhost:9092`、`your-gcp-project`）——可通过 CLI 标志或环境变量覆盖。权重路径（`--backbone-dir`、`--head-checkpoint`）没有内嵌的文件系统默认值。
- 已发布配置的权威维度：256 种动作类型、8 个分类头、序列长度 512、嵌入宽度 1024。
- 结果汇聚（results-sink）强制执行的**操作点**（operating points）——即 `runtime/sink_policy.yaml` 中的各头决策阈值（以及 `score_results_sink_focal.py` 中匹配的回退默认值）——在本公开版本中已**脱敏**：它们以超出范围的 `9.99` 哨兵值随附（这些字段是 `[0, 1]` 区间内的概率，因此 `9.99` 永远不会触发，显然只是占位符，而非真实值）。最小动作数强制门（min-actions enforcement gate，是一个计数而非概率）以同样方式脱敏，使用不可能的 `999999` 哨兵值——远超任何可打分的序列长度。公开精确的操作点会将检测器的规避边界拱手交给对手——包括评分永不触发所依据的最小账号规模下限。策略的*结构*、头名称与门逻辑是真实的且未脱敏；仅调优后的数值被扣留。请通过 `--policy-file` / `BDSM_SINK_POLICY` 自行提供。
- 逐头的**申诉说明模板**（appeal-note templates）：生产环境的汇聚端会从主导的机器人头与选定的直方图计数中插值生成一段简短散文（`runtime/score_results_sink_focal.py` 中的 `build_enforcement_note`）。公开包保留了**门**（gates）（最小动作数门——数值已脱敏，由策略接入——以及主导头选择）和 `enforcement_note` proto 字段。模板*字符串*与逐头的 `key_actions` 插值器为哨兵值 `"<redacted>"` —— 与 `9.99` 操作点的思路相同。当本应触发一条说明时，它携带该哨兵值加上模型头后缀，而非内部申诉段落或各头所依据的动作类型。ActionName proto 枚举保持不变。
