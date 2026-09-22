# 02 — home-mixer 核心推荐链路

上一篇讲了框架与 For You 的外层。这一篇钻进真正的推荐主链路：`PhoenixCandidatePipeline`（`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`，1094 行）。

它是 For You 里 `ScoredPostsSource` 内部调用的对象 —— **你在 For You 里看到的普通帖子，都是从这里出来的**。

## 1. 链路全貌

```
ScoredPostsQuery（已由 QueryBuilder 组装好）
   │
   ├─① query_hydrators × 17      补齐「查询」：社交关系、已看历史、用户序列…
   │
   ├─② sources × 7（并行召回）    thunder / tweet_mixer / simclusters / phoenix
   │                              phoenix_topics / phoenix_moe / cached_posts
   │
   ├─③ hydrators × 12            给每个候选补字段：作者、媒体、互动数、语言…
   │
   ├─④ filters × 18              批量剔除：重复、过期、已看、拉黑、低质…
   │
   ├─⑤ scorers × 3               打分：Phoenix 模型分 → 加权聚合 → VMRanker
   │
   ├─⑥ selector: TopKScoreSelector   按总分取 Top-K
   │
   ├─⑦ post_selection_hydrators × 7  只给入选的少量候选补昂贵字段
   ├─⑧ post_selection_filters × 3    合规/质量兜底
   │
   └─⑨ side_effects                 写 Kafka / 缓存 / 实验埋点
```

**这条链路的性能设计核心是"分级"**：粗筛（④，便宜、规则化）放在打分前，昂贵操作（⑦，需要 RC 校验、品牌安全标签）只对最终入选的几十条做。

## 2. ①查询补水：17 个 QueryHydrator

查询补水解决的是"这条请求还需要什么上下文"。代表组件：

| 补水器 | 作用 |
|---|---|
| `ScoringSequenceQueryHydrator` / `RetrievalSequenceQueryHydrator` | 取用户的**行为序列**（打分序列 / 召回序列）——喂给 Phoenix 模型的用户侧输入 |
| `BlockedUserIdsQueryHydrator` / `MutedUserIdsQueryHydrator` | 拉黑 / 静音名单 |
| `FollowedUserIdsQueryHydrator` / `SubscribedUserIdsQueryHydrator` | 关注 / 订阅关系 |
| `MutualFollowQueryHydrator` | 互关关系（用于互关加权等） |
| `CachedPostsQueryHydrator` | 从 Redis 取缓存过的帖子（避免重复回源） |
| `UserDemographicsQueryHydrator` / `UserInferredGenderQueryHydrator` / `UserInstalledAppsQueryHydrator` | 用户画像类信号 |
| `ImpressedPostsQueryHydrator` / `ImpressionBloomFilterQueryHydrator` | 曝光历史（含布隆过滤器，**用概率数据结构省内存**） |
| `FollowedGrokTopicsQueryHydrator` | 用户关注的 Grok 主题 |
| `PastRequestTimestampsQueryHydrator` | 上次请求时间（控制请求节奏/限频） |

> 注意 `IPQueryHydrator`、`ExplicitEngagementSignalsQueryHydrator` 等也存在于 `home-mixer/query_hydrators/`（该目录共 21 个文件），不同流水线按需装配。

## 3. ②召回：7 路候选源

| 源 | 召回逻辑（按名称与调用点归纳） |
|---|---|
| `ThunderSource` | **网络内**：你关注账号的近期帖子（`thunder` 内存存储，延迟最低） |
| `TweetMixerSource` | 推文混合召回 |
| `SimClustersSource` | 基于 **社群聚类**（`simclusters` 模块）找相似兴趣社区的帖子 |
| `PhoenixSource` | **网络外主力**：Phoenix 模型的向量召回 |
| `PhoenixTopicsSource` | Phoenix 的**主题**维度召回 |
| `PhoenixMoeSource` | Phoenix 的 **MoE**（专家混合）分支召回 |
| `CachedPostsSource` | 缓存命中直接复用 |

> 目录里还有 `popular_topics_source`、`following_night_owl_source`、`reverse_chron_posts_source`、`seed_candidates_source` 等（共 19 个文件），供其他流水线使用。

## 4. ③候选补水：12 个 Hydrator

召回只给出帖子 ID 和少量特征，需要补全才能过滤与打分：

`InNetworkCandidateHydrator`（标记网络内/外）、`BidirectionalFollowHydrator`（互关）、`core_data_hydrator`（正文/作者等核心数据）、`QuoteHydrator`（引用帖）、`MediaInfoHydrator`（媒体元信息）、`SubscriptionHydrator`（订阅状态）、`GizmoduckCandidateHydrator`（用户账号信息）、`BlockedByHydrator`（是否被作者拉黑）、`FilteredTopicsHydrator`（主题过滤）、`LanguageCodeHydrator`（语言）、`EngagementCountsHydrator`（互动数）、`SemanticIdHydrator`（语义 ID）。

## 5. ④过滤：18 个 Filter，五类

| 类别 | 过滤器 | 规则 |
|---|---|---|
| **去重** | `DropDuplicatesFilter`、`RetweetDeduplicationFilter`、`PreviouslySeenPostsFilter`、`PreviouslySeenPostsBackupFilter`、`PreviouslyServedPostsFilter` | 同帖去重、转推去重、已看过（有主备两道）、已下发过 |
| **时效** | `AgeFilter` | 超过 `params::MAX_POST_AGE` 的帖子直接丢弃 |
| **关系/权限** | `SelfTweetFilter`、`AuthorSocialgraphFilter`、`ViewerMutedKeywordFilter`、`IneligibleSubscriptionFilter` | 自己的帖、作者关系链、静音词、订阅资格 |
| **内容类型** | `OONRetweetReplyFilter`、`OONNsfwSimclustersFilter`、`VideoFilter`、`TopicIdsFilter`、`CoreDataHydrationFilter` | 网络外的转推/回复（含"回复无祖先"）不推、NSFW 社群内容、视频开关、主题开关、核心数据缺失 |
| **实验/质量** | `InventoryHoldoutFilter`、`NewUserMinEngagementFilter`、`Brazil2026ElectionFilter` | **库存保留实验**（按原创/回复/转推各留出一定百分比不进推荐，用于测量推荐的因果增量）、新用户最低互动门槛、巴西 2026 选举合规过滤 |

两个值得学习的实现：

- `OONRetweetReplyFilter`：`c.in_network == Some(false) && (is_retweet || is_reply)` 或 `is_reply && ancestors.is_empty()` 即剔除 —— **网络外的转推和回复不推荐**，因为脱离上下文看不懂。
- `InventoryHoldoutFilter`：把候选分成 Original / Reply / Retweet 三类，各按参数（`InventoryHoldoutOriginalsPercent` 等）留出 holdout 组。这是**推荐系统做因果评估的标准手法**，工程代码里能看到非常难得。

## 6. ⑤打分：3 个 Scorer + ⑥TopK 选择

| 打分器 | 职责 |
|---|---|
| `PhoenixScorer`（131 行） | 调 Phoenix 模型（`PredictionDispatch`，支持 xDS 动态路径 + 重试 + fallback 开关）拿到**多目标概率** |
| `RankingScorer`（1635 行，最核心） | 把多目标概率**加权聚合成单一分数**，并做冷启动/负反馈/停留惩罚等修正 |
| `VMRanker` | 价值模型重排（另一路 ranker，独立客户端 + xDS） |

`RankingScorer` 的权重结构（`scorers/ranking_scorer.rs:22` 起）包含 15+ 个正向信号与若干负向信号：

```
正向：favorite, reply, retweet, photo_expand, video_open, click, open_link,
      profile_click, vqv, share, share_via_dm, share_via_copy_link, dwell,
      quote, quoted_click, quoted_vqv, cont_dwell_time, follow_author,
      post_unexplored …
负向：not_interested, block_author, mute_author, report, not_dwelled, negative_sum …
修正：dwell_regret_sigmoid / gated_dwell_regret（停留后悔修正）
      click_dwell_low_fav_rate_penalty（低收藏率点击惩罚，含 baseline/alpha/floor/cap）
      bidirectional_follow_reply_weight_boost / dwell_weight_boost（互关加权）
```

**实际权重值**（`home-mixer/params/param.rs`，可通过 Feature Switch 在线调整）：

| 信号 | 参数名 | 默认值 |
|---|---|---|
| 回复 | `ReplyWeight` | **5.0** |
| 引用 | `QuoteWeight` | **5.0** |
| 分享 | `ShareWeight` | 2.0 |
| 转推 | `RetweetWeight` | 1.0 |
| 点赞 | `FavoriteWeight` | 0.5 |
| 点击 | `ClickWeight` | 0.4 |
| 打开链接 | `OpenLinkWeight` | 0.2 |
| 停留 | `DwellWeight` | 0.05 |
| 视频质量观看 | `VqvWeight` | 0.0（默认关闭） |
| 举报 | `ReportWeight` | **-234.0** |

两个可直接用于自己系统的结论：
1. **负反馈权重远大于正反馈**（举报 -234 vs 点赞 0.5），工程上必须让"一次伤害"抵得过几百次喜欢。
2. **权重不是硬编码**：全部走 `param!` 宏注册成 Feature Switch，既能灰度实验，也能应急调整（`params/param.rs` 里还留有大段关于"一次举报不等于抵消 468 个赞"的注释，说明团队对权重语义有过讨论）。

选择器 `TopKScoreSelector`（`home-mixer/selectors/`）按总分取前 K 条，返回值分离成 `selected` / `non_selected` 两堆（框架层 `SelectResult`），便于下游分析"差一点入选"的候选。

## 7. ⑦⑧入选后处理：7 补水 + 3 过滤

只对最终入选的少量候选执行：

- `VFCandidateHydrator` / `AdsBrandSafetyVfHydrator`：跑**可见性过滤**（visibility filtering），拿回 `Action`
- `TweetTypeMetricsHydrator`、`FollowingRepliedUsersHydrator`、`MutualFollowJaccardHydrator`（互关 Jaccard 相似度）、`TopicFeedbackContextHydrator`、`AiTrendFeedbackContextHydrator`：补齐反馈上下文
- `VFFilter` / `AncillaryVFFilter`：把 `Action::Drop / Tombstone / NotEvaluated` 的候选剔除，`Allow / Interstitial / Avoid / Downrank` 保留（`filters/vf_filter.rs`）
- `DedupConversationFilter`：同会话去重

## 8. ⑨副作用：写出去的东西

`PhoenixExperimentsSideEffect`（实验曝光）、`RerankingKafkaSideEffect`、`RedisPostCandidateCacheSideEffect`（回写候选缓存）、`ScoredStatsSideEffect` 等。它们全部**并行执行且不阻塞响应**，是"主链路只管算分、数据管道异步落地"的典型分工。

## 9. 这一章可以带走什么

1. **分级处理是性能与效果的双赢**：便宜规则在前、昂贵校验在后，且只在入选集上做。
2. **召回多源并行 + 统一抽象**：7 个源都是 `Source` trait 的实现，加一个召回源不需要改框架。
3. **过滤器要留"被丢弃原因"**：框架把 `filtered_candidates` 全量保留，线上排障才有据可查。
4. **推荐系统必须做 holdout**：`InventoryHoldoutFilter` 是评估"推荐到底带来多少增量"的基础设施，值得抄。
5. **权重参数化**：所有打分权重走 Feature Switch，而不是写在代码里。

> 进一步阅读：[03 — 关键设计模式与工程取舍](03-关键设计模式与工程取舍.md)
