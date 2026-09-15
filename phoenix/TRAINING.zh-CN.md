> **说明**：本文件是 [TRAINING.md](TRAINING.md) 的中文译本。如与英文原文有出入，以英文原文为准；代码、命令、路径与标识符均保持原样。

# 训练 Phoenix 推荐系统模型

本导出包含一条可运行的、单主机的训练路径，适用于排序模型和召回模型。[`QUICKSTART.md`](QUICKSTART.md) 是经测试的生成合成数据、训练两个 nano 配置、写出检查点、恢复训练并为其提供服务的途径。

已发布的路径是一个离线验证工具。它不包含生产环境数据源、生产环境检查点、多主机编排，或生产规模的训练配方。这些部分需由你自行提供，用于自己的部署。

## 优化器与训练步骤

本导出包含两类稠密参数优化器族：

- **Muon**（`xrex/optimizers/recsys/muon.py`）：生产环境 home-ranker（主页时间线排序器）所用的配方——在矩阵与嵌入分区上采用一致的 RMS 缩放，并配合解耦权重衰减。旗舰排序配置与 `home_direct_packed_nano` 选用它（`optim="muon"`），因此 nano 训练时使用与生产运行相同的稠密优化器配方。
- **标准 Optax AdamW**：其余已发布配置（双塔召回、gen-recs，以及遗留排序预设）所使用的槽位。内部部署在该槽位使用经过调优的 RMS 归一化 Adam 派生版本；AdamW 是这些已发布配方的经验证等价物，该槽位上的每个配置都端到端地用它训练。

嵌入表使用一个独立的稀疏行级 AdaGrad 优化器；排序旗舰配方（以及 nano）还额外为其启用累加器半衰期衰减、惰性逐行衰减与解耦权重衰减。大体上，每一步：

1. 查找并对 batch 用到的嵌入行去重；
2. 计算稠密参数与嵌入的损失和梯度；
3. 将配置的稠密优化器应用于稠密参数，将行级 AdaGrad 应用于被引用的嵌入行；以及
4. 当梯度非有限时跳过更新。

可运行的参考实现为 [`reference/train_step.py`](reference/train_step.py)。模型损失定义在 `xrex/models/` 下，优化器定义在 `xrex/optimizers/` 下。

## 数据

公开启动器读取由 `reference/dump_gen.py` 生成的一份有限的离线 Parquet dump。排序与双塔召回使用同一份 dump。Semantic-ID 快照由 `reference/world_snapshots.py` 生成。

使用 [`QUICKSTART.md`](QUICKSTART.md) 中的数据处理命令；尤其要首先生成快照，并将生成的 SID parquet 传给 `dump_gen.py`。启动器从 `.valid_batches.json` 读取分区数，并将所选配置指向该 dump。

合成数据在固定种子下具有确定性，且不含生产数据。它用于验证机制，而非模型质量。

## 运行训练

启动器默认使用 nano 排序配置：

```bash
uv run python reference/train_synth.py \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints" --metrics
```

对于双塔召回：

```bash
uv run python reference/train_synth.py \
  --config xrecsys_two_tower_nano_offline_kafka_dump \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints"
```

运行 `uv run python reference/train_synth.py --help` 查看启动器选项。该启动器为单 GPU，会将请求的步数限制在有限 dump 的容量范围内，并写出周期性检查点与最终检查点。

加上 `--metrics` 后，包括 `loss` 在内的逐步骤指标会写入：

```text
<out>/run/metrics.jsonl
```

合成排序的预演显示 loss 下降，但该结果仅用于确认训练能够运行。它并非关于质量或收敛性的声明。

## 检查点与恢复

检查点写入于：

```text
<out>/<config_name>/elapsed_samples_<n>/<run_id>/
```

要恢复训练，用相同的 `--out` 和更大的 `--steps` 值再次运行启动器。训练器会从最新的检查点恢复模型状态、优化器状态与数据位置。由于 dump 是有限的，在更长的训练计划之前需要先生成更多行。

[`QUICKSTART.md`](QUICKSTART.md) 中的服务示例加载由本启动器写出的检查点。不包含任何预训练检查点。

## 生产环境使用

对于生产规模的部署，请自行提供：

- 数据源与数据生命周期；
- 多主机调度与编排；
- 模型配置、训练计划、评测与监控；以及
- 检查点存储与服务集成。

nano 合成工具并不能确立生产环境的准确性、吞吐、可靠性或规模。
