# X 推荐算法源码中文导读

本目录是对 X（Twitter）开源推荐算法仓库 [`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm) 的**中文源码导读**，面向想读懂这套工业级推荐系统的读者。

> **说明**：本导读完全基于本仓库公开代码的阅读整理，不包含任何非公开信息、内部数据或未开源实现。所有结论均标注了对应的源文件路径，可自行核对。
>
> **与「翻译」的区别**：上游仓库的文档极少（多数模块只有代码、没有 README），信息密度几乎全在代码里。所以这里不做逐行翻译，而是把**代码里的架构、数据流、设计取舍**讲清楚。

## 导读目录

| 篇目 | 内容 | 适合谁 |
|---|---|---|
| [01 — 请求生命周期与流水线框架](01-请求生命周期与流水线框架.md) | 一次 `For You` 请求从 gRPC 入口到返回的完整链路；`candidate-pipeline` 框架的 10 个阶段抽象与执行顺序 | 想先建立全局认知的读者 |
| [02 — home-mixer 核心推荐链路](02-home-mixer核心推荐链路.md) | 真正的召回→过滤→打分→选择链路：7 个候选源、12 个补水器、18 个过滤器、3 个打分器逐个拆解 | 想深入推荐链路细节的读者 |
| [03 — 关键设计模式与工程取舍](03-关键设计模式与工程取舍.md) | Prod/Mock 双实现的依赖注入、Feature Switch + Decider、分级过滤、副作用解耦、可观测性设计 | 关注工程架构与可测试性的读者 |

## 阅读前置知识

- **Rust**：本仓库主体（`home-mixer`、`candidate-pipeline`、`thunder`）为 Rust；`simclusters`、`botmaker` 为 Scala；`phoenix`（模型）为 Python + Rust。
- **推荐系统基本概念**：召回（retrieval/sourcing）、粗排、精排（ranking/scoring）、重排（re-ranking/selection）、混排（blending）。
- **gRPC / Protobuf**：服务入口与跨服务通信方式（`home-mixer/main.rs` 起 tonic 服务）。

## 模块速查

| 模块 | 语言 | 文件数 | 职责 |
|---|---|---|---|
| `home-mixer` | Rust | 224 | **推荐主服务**：编排召回、过滤、打分、混排，对外提供 For You / Following 等 Feed |
| `candidate-pipeline` | Rust | 11 | **流水线框架**：定义 `Source` / `Hydrator` / `Filter` / `Scorer` / `Selector` / `SideEffect` 抽象与统一执行器 |
| `thunder` | Rust | 24 | 内存候选存储：存放近期帖子，供超低延迟召回 |
| `phoenix` | Python + Rust | 323 | **推荐模型**：召回与排序的神经网络（含训练、推理、特征） |
| `simclusters` | Scala | 232 | 社群聚类：基于图聚类的兴趣社区，用于候选召回与相似内容 |
| `botmaker` / `botmaker-rules` | Scala | 492 | 内容治理：机器人/垃圾内容识别与规则引擎 |
| 其余模块 | 混合 | — | 广告、安全、可见性过滤、审核等（见仓库根目录） |

## 建议阅读顺序

1. 先读 `candidate-pipeline/`（11 个文件，全框架）——**这是理解全局的钥匙**，所有业务流水线都是它的实例。
2. 再读 `home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs`——最短的一条业务流水线（326 行），一眼看清 8 个 stage 怎么装配。
3. 然后进 `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`（1094 行）——真正的主推荐链路。
4. 想追模型细节，再进 `phoenix/`。
