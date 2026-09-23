# 04 — thunder 网内召回内存存储

`thunder/` 是 For You 的**第一路召回**：你关注的账号最近发了什么。对应 02 篇里的 `ThunderSource`。

它只有 24 个文件，但其中 3 个是生成的 schema（`schema/user.rs` 一个文件就 14750 行），**真正的逻辑约 3000 行**，是整套系统里最容易一次读完的模块。

## 1. 定位：为什么需要它

For You 要解决"你可能想看什么"，第一优先级永远是"你关注的人刚发了什么"。这条路径的要求只有两个：**要快**（不能拖慢整条流水线）、**要新**（刚发的帖必须立刻可见）。

通用方案（查数据库）做不到亚毫秒级；`thunder` 的做法是把关注关系上的帖子**全部放进内存**，用 Kafka 增量维护。

```
发帖事件（Kafka）
     │
     ▼
thunder（内存存储：post_id → CompactPost，作者 → 帖子队列）
     │  gRPC: GetInNetworkPosts(user_id, following_user_ids, exclude_tweet_ids)
     ▼
home-mixer: ThunderSource → 候选进入候选池
```

## 2. 为什么能快：4 个设计选择

### 2.1 无锁并发表

```rust
posts: Arc<DashMap<i64, Arc<CompactPost>>>,
original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
deleted_posts: Arc<DashMap<i64, bool>>,
```

（`thunder/posts/post_store.rs:88-97`）

用 `DashMap`（分片无锁哈希表）而非 `Mutex<HashMap>`：读多写多的场景下，分片锁避免了全局竞争。

### 2.2 两级数据结构：查全量 vs 查人

| 索引 | 用途 |
|---|---|
| `posts`（post_id → `Arc<CompactPost>`） | 按帖子 ID 直查；`Arc` 让多路引用不复制数据 |
| `*_posts_by_user`（作者 → `VecDeque<TinyPost>`） | 按用户查"他最近发的帖"，队列天然按时间有序 |

`TinyPost` 只有 `post_id + created_at` 两个字段（`post_store.rs:21`）—— **列表里只放指针级信息，重字段留给主表**。这是典型的"索引与数据分离"。

### 2.3 三级分类：原创 / 回复转推 / 视频

同一个作者被拆进三个队列（`original_posts_by_user` / `secondary_posts_by_user` / `video_posts_by_user`），各自有独立上限：

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_ORIGINAL_POSTS_PER_AUTHOR` | 50 | 每作者保留的原创帖上限 |
| `MAX_REPLY_POSTS_PER_AUTHOR` | 30 | 每作者保留的回复帖上限 |
| `MAX_VIDEO_POSTS_PER_AUTHOR` | 100 | 每作者保留的视频帖上限 |
| `MAX_POSTS_TO_RETURN` | 1200 | 单次返回上限 |
| `MAX_VIDEOS_TO_RETURN` | 600 | 单次视频返回上限 |
| `MAX_INPUT_LIST_SIZE` | 10000 | 请求里关注列表长度上限 |
| `MAX_TINY_POSTS_PER_USER_SCAN` | 500 | 单用户扫描上限 |
| `MAX_POSTING_LIST_SIZE` | 5000 | 发帖列表上限 |
| `TRIM_FRACTION` | 5 | 裁剪批次占比 |
| `DELETE_EVENT_KEY` | 0 | 删除事件的特殊 key |

（`thunder/config.rs`）

**这是内存系统必须做的事**：给每个维度设硬上限，使内存占用有上界（O(用户数 × 每用户上限)），不会因为某个大 V 突然爆发而击穿。

### 2.4 软删除墓碑

删除不直接消失，而是 `mark_as_deleted()`：从 `posts` 移除，同时在 `deleted_posts` 打标记（`post_store.rs:112`）。

**为什么需要墓碑**：Kafka 是乱序/可重放的，同一条帖的"发布"事件可能在"删除"事件之后到达。没有墓碑就会出现"已删除的帖复活"。这类乱序防护在任何事件驱动系统里都是必须的 —— 分片集群里见过太多"删了又回来"的诡异 bug。

## 3. 写入路径：Kafka → 内存

```
Kafka topic
  ↓ thunder/kafka/tweet_events_listener_v2.rs  start_tweet_event_processing_v2()
  ↓ thunder/deserializer.rs  deserialize_tweet_event_v2(payload) -> InNetworkEvent
  ↓ PostStore::insert_posts(Vec<LightPost>)      ← 批量插入
  ↓ insert_posts_internal()                       ← 分类 + 上限裁剪 + 入队
```

（`thunder/main.rs:91` 起用 `tokio::sync::mpsc::channel::<i64>(args.kafka_num_threads)` 把消息分发给多个 worker 线程 —— 消费并发度可配。）

**冷启动**：进程重启后内存是空的，需要一个预热来源。`thunder/o2/mod.rs` 提供了 `O2Client`——基于 `object_store` crate 的 S3 兼容对象存储客户端（从环境变量读 `O2_ACCESS_KEY_ID` / `O2_SECRET_ACCESS_KEY` / `O2_ENDPOINT_URL`）。配合 `PostStore::finalize_init()`（`post_store.rs:143`）与 `sort_all_user_posts()`（`post_store.rs:606`），可以从快照批量装载后重建有序结构，再靠 Kafka 追增量。

> 这解释了一个常见疑问："内存系统重启不就空了？" 答案是**快照 + 增量**：对象存储兜底历史，Kafka 补最新。

## 4. 读取路径：一次 gRPC 调用

实现在 `impl InNetworkPostsService for ThunderServiceImpl::get_in_network_posts`（`thunder/thunder_service.rs:149`）：

1. **限流**：`rate_limiter.check()` 失败直接返回 `resource_exhausted`（`:153`）—— 防止上游异常把自己打穿。
2. **在途计数**：`IN_FLIGHT_REQUESTS` 用 `InFlightGuard` + `Drop` 自动增减（`:160-168`）—— 用 RAII 保证"无论从哪个分支返回都不漏减"，比在每个 return 前手动减可靠得多。
3. **分段计时**：`_total_timer` 与 `_processing_timer`（`GET_IN_NETWORK_POSTS_DURATION_WITHOUT_STRATO`，`:221`）分开统计，**把"取关注列表"和"取帖子"的耗时拆开**，否则无法判断慢在依赖还是慢在自己。
4. **关注列表兜底**：请求里没带关注列表时，回源到 Strato 拉取（`:190`，上限 `MAX_INPUT_LIST_SIZE`）。
5. **结果上限决策**：`max_results > 0` 用请求值；否则视频请求用 `MAX_VIDEOS_TO_RETURN`，普通请求用 `MAX_POSTS_TO_RETURN`（`:223`）。
6. **取数据**：`get_all_posts_by_users()` / `get_videos_by_users()` → 内部 `get_posts_from_map()`。
7. **排序**：`score_recent(light_posts, max_results)`（`:322`）—— 网内候选的排序原则就是**按新鲜度**，先给 home-mixer 一批"最近发的"，精排交给下游。
8. **排除已看**：请求携带 `exclude_tweet_ids`，在 `get_posts_from_map` 里过滤（单元测试 `test_exclude_tweet_ids_filtering` 覆盖了该行为）。

## 5. 内存不会无限涨：保留期 + 自动裁剪

- 构造时传入 `retention_seconds`（`PostStore::new`，`post_store.rs:100`）—— 超过保留期的帖子不再有意义（For You 侧的 `AgeFilter` 也会过期拦截）。
- `start_auto_trim(interval_minutes)` 起后台任务周期性 `trim_old_posts(trim_iter)`（`post_store.rs:477`、`:499`），每次按 `TRIM_FRACTION` 比例裁剪。
- `start_stats_logger()`（`:377`）周期输出存储水位。

## 6. 可观测性清单

`thunder/metrics.rs` 里的指标几乎是这类内存系统的标准模板：

| 类别 | 指标 |
|---|---|
| 消费端 | `KAFKA_MESSAGES_FAILED_PARSE`、`KAFKA_POLL_ERRORS`、`BATCH_PROCESSING_TIME`、**`KAFKA_PARTITION_LAG`**（最关键的背压信号） |
| 存储水位 | `POST_STORE_USER_COUNT`、`POST_STORE_TOTAL_POSTS`、`POST_STORE_MAX_POSTS_PER_USER`、`POST_STORE_ENTITY_COUNT`、`POST_STORE_USER_POSTS_DISTRIBUTION` |
| 质量 | `POST_STORE_DELETED_POSTS`、`POST_STORE_REPLIES_FILTERED`、`POST_STORE_POSTS_RETURNED` 及 `POST_STORE_POSTS_RETURNED_RATIO` |
| 依赖 | `STRATO_REQUESTS` / `STRATO_REQUEST_DURATION` |
| 接口 | `GET_IN_NETWORK_POSTS_{DURATION, COUNT, FOLLOWING_SIZE, EXCLUDED_SIZE, MAX_RESULTS}`、`REJECTED_REQUESTS`、`IN_FLIGHT_REQUESTS` |

注意 `POST_STORE_POSTS_RETURNED_RATIO`（返回/总量比）和 `REPLIES_FILTERED` 这类**业务语义指标** —— 它们能直接回答"是不是过滤得太狠了""候选池是不是在缩水"，比纯技术指标有用得多。

## 7. 可以带走的工程经验

1. **内存索引要"索引与数据分离"**：列表里只放 `(id, timestamp)`，重数据放主表，并统一用 `Arc` 共享 —— 复制成本直接归零。
2. **每个维度都要有硬上限**：单作者 50/30/100、列表 5000、返回 1200。内存系统的容量必须是**可计算的**。
3. **删除要打墓碑**：事件乱序是常态，没有墓碑就会出现"删掉的帖子复活"。
4. **冷启动 = 快照 + 增量**：对象存储装历史、Kafka 追增量，中间用 `finalize_init` 把结构重建好再对外服务。
5. **计时指标要分段**：把依赖调用与本地处理分开统计（`DURATION` vs `DURATION_WITHOUT_STRATO`），否则无法归因。
6. **在途请求用 RAII 计数**：`Drop` 里自减，避免早期 return 漏减导致指标永久漂移。

> 下一篇可读：`phoenix/`（模型侧）、`simclusters/`（社群聚类召回）。
