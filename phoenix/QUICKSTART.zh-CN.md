> **说明**：本文件是 [QUICKSTART.md](QUICKSTART.md) 的中文译本。如与英文原文有出入，以英文原文为准；代码、命令、路径与标识符均保持原样。

# 快速上手

本指南使用合成数据验证已发布的 nano 排序与召回路径：训练、保存检查点、恢复训练、提供服务，然后发送一次召回 → 排序请求。它并非生产质量的模型，也不是生产规模的部署。生产数据、检查点、编排与规模均不包含在内。

所有命令均从此目录（导出根目录）运行。

## 环境要求

- 装有 NVIDIA GPU 且具备 CUDA 12 的 Linux
- `uv` 以及 Python 3.11 或更新版本
- Rust 工具链与 `protoc` 3.15 或更新版本

系统软件包的安装说明请参阅 [`README.md`](README.md)。

```bash
uv sync --extra engine
export PYTHONPATH=$PWD
```

训练与服务提供（serving）需要 engine 这一 extra。首次模型运行在 JAX 编译期间可能需要几分钟。

使用随机权重验证安装：

```bash
uv run python xrex/inference/oss_bench/bench.py --smoke --service_type ranking
```

## 1. 生成确定性的合成数据

```bash
uv run python reference/world_snapshots.py --out ./synth_index --seed 20260721
export PHOENIX_INDEX_BASE=./synth_index

uv run python reference/dump_gen.py --out ./synth_dump --seed 20260721 \
  --num-rows 12288 --partitions 4 --rows-per-file 1024 \
  --sid ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --self-check
```

相同的种子会生成相同的合成数据。后续命令请保持 `PHOENIX_INDEX_BASE=./synth_index` 已设置。

## 2. 训练排序模型

```bash
uv run python reference/train_synth.py \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints" --metrics

uv run python -c 'import json; print(*((m["step"], m["loss"]) for m in map(json.loads, open("checkpoints/run/metrics.jsonl"))), sep="\n")'
```

第一条命令会在 `./checkpoints/home_direct_packed_nano_offline_kafka_dump/` 下写入检查点。第二条命令打印已记录的 step 与 loss 值。这次演练在该合成数据运行上观察到 loss 下降；但请勿将六步训练视为模型质量的佐证。

## 3. 恢复训练

使用相同的输出目录并增大总 step 上限：

```bash
uv run python reference/train_synth.py \
  --data ./synth_dump --steps 12 --out "$PWD/checkpoints"
```

训练器会自动发现最新的检查点。其日志包含：

```text
Discovered checkpoint to load:
CheckpointMeta(...)
Restored from checkpoint: <count> tensors ... checkpoint step is <N>
Checkpoint checksums match
create_dataset: resume_position={'last_batch_id': ..., ...}
```

模型状态、优化器状态与数据位置都会被恢复。合成 dump 是有限的；在请求更长时间的运行前，请先生成更多行。

## 4. 提供排序检查点服务

```bash
RANK_CKPT=$(ls -d "$PWD"/checkpoints/home_direct_packed_nano_offline_kafka_dump/elapsed_samples_*/*/ | sort | tail -1)

uv run python xrex/inference/oss_bench/bench.py \
  --checkpoint_path "$RANK_CKPT" \
  --service_type ranking \
  --config_name home_direct_packed_nano_offline_kafka_dump
```

`bench.py` 会恢复检查点、启动 gRPC 服务器，在端口可用时发送一次合成请求，然后停止服务器。恢复与预热的日志包含：

```text
Restored from checkpoint: <count> tensors ... checkpoint step is <N>
Checkpoint checksums match
gRPC server ready.
Model warm up finished.
```

在启动下文完整栈之前，请等待 9988 端口被释放。

## 5. 训练召回模型并运行「召回 → 排序」

在同一份 dump 上训练双塔召回模型：

```bash
uv run python reference/train_synth.py \
  --config xrecsys_two_tower_nano_offline_kafka_dump \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints"
```

召回检查点包含由生成的合成快照构建的候选索引。

启动 SID 服务与两个模型服务器：

```bash
RANK_CKPT=$(ls -d "$PWD"/checkpoints/home_direct_packed_nano_offline_kafka_dump/elapsed_samples_*/*/ | sort | tail -1)
RETR_CKPT=$(ls -d "$PWD"/checkpoints/xrecsys_two_tower_nano_offline_kafka_dump/elapsed_samples_*/*/ | sort | tail -1)

uv run python reference/sid_index_server.py \
  --parquet ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --port 50061 &

XLA_PYTHON_CLIENT_MEM_FRACTION=0.30 uv run python xrex/inference/launch_inference.py \
  --driver local --service_type retrieval \
  --config_name xrecsys_two_tower_nano_offline_kafka_dump \
  --checkpoint_path "$RETR_CKPT" --grpc_port 9990 \
  --sid_endpoint localhost:50061 \
  --num_devices_per_process 1 --bs_per_device 1 \
  --history_seq_len 128 --candidate_seq_len 8 \
  --max_inflight_requests 16 --allow_random_init false --fake_mm_embeddings true \
  attn_impl=pallas_ranker_attn use_seqpack=False right_anchored_rope=True \
  bs_per_device=1 parallel_config.num_devices_per_process=1 num_devices_per_process=1 \
  ep=1 dp=1 training_ep=1 &

XLA_PYTHON_CLIENT_MEM_FRACTION=0.30 uv run python xrex/inference/launch_inference.py \
  --driver local --service_type ranking \
  --config_name home_direct_packed_nano_offline_kafka_dump \
  --checkpoint_path "$RANK_CKPT" --grpc_port 9988 --metrics_port 9091 \
  --num_devices_per_process 1 --bs_per_device 1 \
  --history_seq_len 128 --candidate_seq_len 16 \
  --max_inflight_requests 16 --allow_random_init false --fake_mm_embeddings true \
  attn_impl=pallas_ranker_attn_infer use_seqpack=False right_anchored_rope=True \
  bs_per_device=1 parallel_config.num_devices_per_process=1 num_devices_per_process=1 \
  ep=1 dp=1 training_ep=1 \
  model_config.model_config.sequence_len=146 &
```

等待两个模型服务器均输出 `Server ready to serve` 日志，然后通过两个真实的 gRPC 服务发送三个合成会话：

```bash
uv run python reference/retrieve_then_rank.py \
  --data ./synth_dump --sessions 3 --topk 16 \
  --retrieval-port 9990 --ranking-port 9988
```

客户端将每个会话传给召回，再请求排序对相同用户序列返回的候选（内容）进行打分。一次成功运行会以如下日志结束：

```text
retrieve_then_rank: 3 session(s) completed the full loop.
```

这仅验证了已发布的集成。nano 模型与合成数据并不能体现推荐质量、生产性能或规模。

## 后续

- [`TRAINING.md`](TRAINING.md) 说明了已发布的训练路径。
- [`reference/README.md`](reference/README.md) 描述了合成数据与快照工具。
