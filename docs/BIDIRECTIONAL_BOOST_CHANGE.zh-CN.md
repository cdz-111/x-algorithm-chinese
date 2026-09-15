# 算法变更示例：双向关注提升

今后，未来的算法更新都会发布到这个开源仓库，你应该可以通过查看 diff 来理解改了什么。

为了让你对这件事的体验有个概念，下面是 2026 年 7 月逐步推开的、被广泛讨论的[双向关注算法变更](https://x.com/nikitabier/status/2076747704248758617)中，你可能会看到的内容。通俗地说，双向关注提升会提高「你会回复某位与你互相关注的作者所发帖子」这一预测概率的权重，从而提升你互相关注的人所发的原创帖。

2026 年 7 月 10 日，我们启动了一项 A/B 测试，用来实现双向关注回复提升：其中一小部分用户被随机分配到 5、10、15 或 20 的双向关注回复提升值（当时大多数用户的值被设为 0，相当于没有双向关注回复提升）。你还会看到一段双向关注停留权重提升的代码，它最初是作为同一个 A/B 测试的一部分进行测试的，但没有更广泛地发布。

2026 年 7 月 13 日，在看到 A/B 测试不错的初步结果后，我们向大量用户放开了 20 的提升值，同时继续以不同比例的用户对 0、5、10、15 等其他提升值做实验。

随后，2026 年 7 月 24 日，在拿回实验结果并听取了 X 上的反馈之后——例如世界杯正在举行，但有些人看到的讨论没有他们期望的那么多，因为许多帖子来自他们未关注的账号——我们把双向关注回复提升值从 20 调整为 15。


## 2026 年 7 月 13 日：首次大范围上线

#### `home-mixer/candidate_hydrators/bidirectional_follow_hydrator.rs`

```diff
diff --git a/home-mixer/candidate_hydrators/bidirectional_follow_hydrator.rs b/home-mixer/candidate_hydrators/bidirectional_follow_hydrator.rs
new file mode 100644
index 0000000000000..6488e8f1c1040
--- /dev/null
+++ b/home-mixer/candidate_hydrators/bidirectional_follow_hydrator.rs
@@ -0,0 +1,201 @@
+use crate::models::candidate::PostCandidate;
+use crate::models::query::ScoredPostsQuery;
+use crate::params::EnableBidirectionalFollowHydration;
+use std::collections::HashSet;
+use std::sync::Arc;
+use tonic::async_trait;
+use xai_candidate_pipeline::component_library::clients::SocialGraphClientOps;
+use xai_candidate_pipeline::hydrator::Hydrator;
+
+pub struct BidirectionalFollowHydrator {
+    pub socialgraph_client: Arc<dyn SocialGraphClientOps>,
+}
+
+#[async_trait]
+impl Hydrator<ScoredPostsQuery, PostCandidate> for BidirectionalFollowHydrator {
+    fn enable(&self, query: &ScoredPostsQuery) -> bool {
+        query.params.get(EnableBidirectionalFollowHydration)
+    }
+
+    async fn hydrate(
+        &self,
+        query: &ScoredPostsQuery,
+        candidates: &[PostCandidate],
+    ) -> Vec<Result<PostCandidate, String>> {
+        let following: HashSet<i64> = query
+            .user_features
+            .followed_user_ids
+            .iter()
+            .copied()
+            .collect();
+
+        let followed_authors: Vec<u64> = candidates
+            .iter()
+            .map(|c| c.author_id)
+            .filter(|a| following.contains(&(*a as i64)))
+            .collect::<HashSet<_>>()
+            .into_iter()
+            .collect();
+
+        let mutual = if followed_authors.is_empty() {
+            HashSet::new()
+        } else {
+            match self
+                .socialgraph_client
+                .check_followed_by(query.user_id, &followed_authors)
+                .await
+            {
+                Ok(m) => m,
+                Err(e) => return candidates.iter().map(|_| Err(e.to_string())).collect(),
+            }
+        };
+
+        candidates
+            .iter()
+            .map(|c| {
+                Ok(PostCandidate {
+                    is_mutual_follow_author: Some(mutual.contains(&c.author_id)),
+                    ..Default::default()
+                })
+            })
+            .collect()
+    }
+
+    fn update(&self, candidate: &mut PostCandidate, hydrated: PostCandidate) {
+        candidate.is_mutual_follow_author = hydrated.is_mutual_follow_author;
+    }
+}
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+    use crate::models::user_features::UserFeatures;
+    use tonic::Status;
+
+    struct MockSocialGraph {
+        followers: Vec<i64>,
+    }
+
+    #[async_trait]
+    impl SocialGraphClientOps for MockSocialGraph {
+        async fn get_following_list(&self, _user_id: u64) -> Result<Vec<u64>, Status> {
+            Ok(vec![])
+        }
+        async fn check_blocked_by(
+            &self,
+            _viewer_id: u64,
+            _author_ids: &[u64],
+        ) -> Result<HashSet<u64>, Status> {
+            Ok(HashSet::new())
+        }
+        async fn check_followed_by(
+            &self,
+            _viewer_id: u64,
+            user_ids: &[u64],
+        ) -> Result<HashSet<u64>, Status> {
+            let followers: HashSet<i64> = self.followers.iter().copied().collect();
+            Ok(user_ids
+                .iter()
+                .copied()
+                .filter(|id| followers.contains(&(*id as i64)))
+                .collect())
+        }
+        async fn get_blocked_user_ids(&self, _viewer_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_muted_user_ids(&self, _viewer_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_followed_user_ids(&self, _viewer_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_follower_ids(&self, _user_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_subscribed_user_ids(&self, _viewer_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_device_following_user_ids(&self, _viewer_id: u64) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+        async fn get_hide_recommendations_user_ids(
+            &self,
+            _viewer_id: u64,
+        ) -> Result<Vec<i64>, Status> {
+            Ok(vec![])
+        }
+    }
+
+    fn hydrator(followers: Vec<i64>) -> BidirectionalFollowHydrator {
+        BidirectionalFollowHydrator {
+            socialgraph_client: Arc::new(MockSocialGraph { followers }),
+        }
+    }
+
+    fn query(followed: Vec<i64>, enabled: bool) -> ScoredPostsQuery {
+        let mut query = ScoredPostsQuery {
+            user_id: 42,
+            user_features: UserFeatures {
+                followed_user_ids: followed,
+                ..Default::default()
+            },
+            ..Default::default()
+        };
+        let fs = xai_feature_switches::FeatureSwitches::new(vec![]).unwrap();
+        let mut results =
+            fs.match_recipient(&xai_feature_switches::RecipientBuilder::new().build());
+        results.override_fs(
+            "rust_home_mixer_enable_bidirectional_follow_hydration".to_string(),
+            if enabled { "true" } else { "false" },
+        );
+        query.params = results.into();
+        query
+    }
+
+    fn candidate(author_id: u64) -> PostCandidate {
+        PostCandidate {
+            author_id,
+            ..Default::default()
+        }
+    }
+
+    #[tokio::test]
+    async fn tags_only_mutual_follow_authors() {
+        let hydrator = hydrator(vec![2, 3, 5]);
+        let q = query(vec![1, 2, 3], true);
+        let candidates = vec![candidate(1), candidate(2), candidate(4)];
+
+        let out = hydrator.hydrate(&q, &candidates).await;
+
+        assert_eq!(
+            out[0].as_ref().unwrap().is_mutual_follow_author,
+            Some(false)
+        );
+        assert_eq!(out[1].as_ref().unwrap().is_mutual_follow_author, Some(true));
+        assert_eq!(
+            out[2].as_ref().unwrap().is_mutual_follow_author,
+            Some(false)
+        );
+    }
+
+    #[tokio::test]
+    async fn empty_following_tags_all_false() {
+        let hydrator = hydrator(vec![1, 2, 3]);
+        let q = query(vec![], true);
+        let candidates = vec![candidate(1), candidate(2)];
+
+        let out = hydrator.hydrate(&q, &candidates).await;
+
+        assert!(
+            out.iter()
+                .all(|c| c.as_ref().unwrap().is_mutual_follow_author == Some(false))
+        );
+    }
+
+    #[test]
+    fn disabled_by_default() {
+        let hydrator = hydrator(vec![]);
+        assert!(!hydrator.enable(&ScoredPostsQuery::default()));
+        assert!(!hydrator.enable(&query(vec![1], false)));
+    }
+}
```

#### `home-mixer/candidate_hydrators/mod.rs`

```diff
diff --git a/home-mixer/candidate_hydrators/mod.rs b/home-mixer/candidate_hydrators/mod.rs
index 406035620372b..113e895ac606a 100644
--- a/home-mixer/candidate_hydrators/mod.rs
+++ b/home-mixer/candidate_hydrators/mod.rs
@@ -1,5 +1,6 @@
 pub mod ads_brand_safety_hydrator;
 pub mod ads_brand_safety_vf_hydrator;
+pub mod bidirectional_follow_hydrator;
 pub mod blocked_by_hydrator;
 pub mod broadcast_liveness_hydrator;
 pub mod core_data_candidate_hydrator;
```

#### `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`

```diff
diff --git a/home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs b/home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs
index aa0318d1eeb88..296908336a547 100644
--- a/home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs
+++ b/home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs
@@ -1,5 +1,6 @@
 use crate::candidate_hydrators::ads_brand_safety_hydrator::AdsBrandSafetyHydrator;
 use crate::candidate_hydrators::ads_brand_safety_vf_hydrator::AdsBrandSafetyVfHydrator;
+use crate::candidate_hydrators::bidirectional_follow_hydrator::BidirectionalFollowHydrator;
 use crate::candidate_hydrators::blocked_by_hydrator::BlockedByHydrator;
 use crate::candidate_hydrators::broadcast_liveness_hydrator::BroadcastLivenessHydrator;
 use crate::candidate_hydrators::core_data_candidate_hydrator::CoreDataCandidateHydrator;
@@ -300,5 +301,8 @@ impl PhoenixCandidatePipeline {
         let hydrators: Vec<Box<dyn Hydrator<ScoredPostsQuery, PostCandidate>>> = vec![
             Box::new(InNetworkCandidateHydrator),
+            Box::new(BidirectionalFollowHydrator {
+                socialgraph_client: socialgraph_client.clone(),
+            }),
             Box::new(CoreDataCandidateHydrator::new(tes_client.clone()).await),
             Box::new(QuoteHydrator::new(tes_client.clone(), socialgraph_client.clone()).await),
             Box::new(VideoDurationCandidateHydrator::new(tes_client.clone()).await),
```

#### `home-mixer/models/candidate.rs`

```diff
diff --git a/home-mixer/models/candidate.rs b/home-mixer/models/candidate.rs
index 244754ef48f24..8dd28c32f898a 100644
--- a/home-mixer/models/candidate.rs
+++ b/home-mixer/models/candidate.rs
@@ -54,6 +54,7 @@ pub struct PostCandidate {
     pub view_count: Option<u64>,
     pub bookmark_count: Option<i64>,
     pub mutual_follow_jaccard: Option<f64>,
+    pub is_mutual_follow_author: Option<bool>,
     pub brand_safety_verdict: Option<BrandSafetyVerdict>,
     pub nsfw_author: Option<bool>,
     #[serde(default)]
```

#### `home-mixer/params/param.rs`

```diff
diff --git a/home-mixer/params/param.rs b/home-mixer/params/param.rs
index 6ca1088092628..bdd78044651dd 100644
--- a/home-mixer/params/param.rs
+++ b/home-mixer/params/param.rs
@@ -272,5 +272,17 @@ param!(
 param!(FavoriteWeight, f64, "rust_home_mixer_favorite_weight", 0.5);
 param!(ReplyWeight, f64, "rust_home_mixer_reply_weight", 5.0);
+param!(
+    BidirectionalFollowReplyWeightBoost,
+    f64,
+    "rust_home_mixer_bidirectional_follow_reply_weight_boost",
+    20.0
+);
+param!(
+    BidirectionalFollowDwellWeightBoost,
+    f64,
+    "rust_home_mixer_bidirectional_follow_dwell_weight_boost",
+    0.0
+);
 param!(RetweetWeight, f64, "rust_home_mixer_retweet_weight", 1.0);
 param!(
     PhotoExpandWeight,
@@ -608,6 +620,13 @@ param!(
     "rust_home_mixer_enable_mutual_follow_jaccard_hydration",
     false
 );
+
+param!(
+    EnableBidirectionalFollowHydration,
+    bool,
+    "rust_home_mixer_enable_bidirectional_follow_hydration",
+    true
+);
 param!(
     EnableHasMediaHydration,
     bool,
```

#### `home-mixer/scorers/ranking_scorer.rs`

```diff
diff --git a/home-mixer/scorers/ranking_scorer.rs b/home-mixer/scorers/ranking_scorer.rs
index f51bda993108e..02c01f2888c69 100644
--- a/home-mixer/scorers/ranking_scorer.rs
+++ b/home-mixer/scorers/ranking_scorer.rs
@@ -55,6 +55,8 @@ pub(crate) struct ScoringWeights {
     enable_quoted_vqv_duration_check: bool,
     open_link_in_network_only: bool,
     open_link_live_broadcast_only: bool,
+    bidirectional_follow_reply_weight_boost: f64,
+    bidirectional_follow_dwell_weight_boost: f64,
 }
 
 impl ScoringWeights {
@@ -87,6 +91,10 @@ impl ScoringWeights {
         let enable_quoted_vqv_duration_check = params.get(EnableQuotedVqvDurationCheck);
         let open_link_in_network_only = params.get(OpenLinkWeightInNetworkOnly);
         let open_link_live_broadcast_only = params.get(OpenLinkWeightLiveBroadcastOnly);
+        let bidirectional_follow_reply_weight_boost =
+            params.get(BidirectionalFollowReplyWeightBoost);
+        let bidirectional_follow_dwell_weight_boost =
+            params.get(BidirectionalFollowDwellWeightBoost);
 
         let positive_sum = favorite
             + reply
@@ -138,6 +146,8 @@ impl ScoringWeights {
             enable_quoted_vqv_duration_check,
             open_link_in_network_only,
             open_link_live_broadcast_only,
+            bidirectional_follow_reply_weight_boost,
+            bidirectional_follow_dwell_weight_boost,
         }
     }
 }
@@ -152,5 +162,29 @@ impl ScoringWeights {
         }
         self.open_link
     }
+
+    fn bidirectional_boost_eligible(candidate: &PostCandidate) -> bool {
+        candidate.in_reply_to_tweet_id.is_none()
+            && candidate.retweeted_tweet_id.is_none()
+            && candidate.is_mutual_follow_author == Some(true)
+    }
+
+    fn reply_weight_for(&self, candidate: &PostCandidate) -> f64 {
+        if self.bidirectional_follow_reply_weight_boost != 0.0
+            && Self::bidirectional_boost_eligible(candidate)
+        {
+            return self.reply + self.bidirectional_follow_reply_weight_boost;
+        }
+        self.reply
+    }
+
+    fn dwell_weight_for(&self, candidate: &PostCandidate) -> f64 {
+        if self.bidirectional_follow_dwell_weight_boost != 0.0
+            && Self::bidirectional_boost_eligible(candidate)
+        {
+            return self.dwell + self.bidirectional_follow_dwell_weight_boost;
+        }
+        self.dwell
+    }
 }
 
@@ -235,7 +275,7 @@ impl RankingScorer {
         );
 
         let combined_score = Self::apply(scores.favorite_score, weights.favorite)
-            + Self::apply(scores.reply_score, weights.reply)
+            + Self::apply(scores.reply_score, weights.reply_weight_for(candidate))
             + Self::apply(scores.retweet_score, weights.retweet)
             + Self::apply(scores.photo_expand_score, weights.photo_expand)
             + Self::apply(scores.click_score, weights.click)
@@ -248,7 +288,7 @@ impl RankingScorer {
                 scores.share_via_copy_link_score,
                 weights.share_via_copy_link,
             )
-            + Self::apply(scores.dwell_score, weights.dwell)
+            + Self::apply(scores.dwell_score, weights.dwell_weight_for(candidate))
             + Self::apply(scores.quote_score, weights.quote)
             + Self::apply(scores.quoted_click_score, weights.quoted_click)
             + Self::apply(scores.quoted_vqv_score, quoted_vqv_weight)
@@ -640,5 +680,63 @@ mod tests {
         assert!((oon_score - in_network_score * 0.75).abs() < 1e-9);
     }
 
+    #[test]
+    fn bidirectional_weight_boosts_only_mutual_original_posts() {
+        let query = query_with_flags(&[
+            (
+                "rust_home_mixer_bidirectional_follow_reply_weight_boost",
+                "3.0",
+            ),
+            (
+                "rust_home_mixer_bidirectional_follow_dwell_weight_boost",
+                "2.0",
+            ),
+        ]);
+        let weights = ScoringWeights::from_params(&query.params);
+        let base_reply = query.params.get(ReplyWeight);
+        let base_dwell = query.params.get(DwellWeight);
+
+        let mutual_original = PostCandidate {
+            is_mutual_follow_author: Some(true),
+            ..candidate(20, Some(true))
+        };
+        assert!((weights.reply_weight_for(&mutual_original) - (base_reply + 3.0)).abs() < 1e-9);
+        assert!((weights.dwell_weight_for(&mutual_original) - (base_dwell + 2.0)).abs() < 1e-9);
+
+        let mutual_reply = PostCandidate {
+            is_mutual_follow_author: Some(true),
+            ..candidate_with_reply(20, Some(true))
+        };
+        assert!((weights.reply_weight_for(&mutual_reply) - base_reply).abs() < 1e-9);
+        assert!((weights.dwell_weight_for(&mutual_reply) - base_dwell).abs() < 1e-9);
+        let mutual_retweet = PostCandidate {
+            is_mutual_follow_author: Some(true),
+            ..candidate_with_retweet(20, Some(true))
+        };
+        assert!((weights.reply_weight_for(&mutual_retweet) - base_reply).abs() < 1e-9);
+
+        let non_mutual = PostCandidate {
+            is_mutual_follow_author: Some(false),
+            ..candidate(999, Some(true))
+        };
+        assert!((weights.reply_weight_for(&non_mutual) - base_reply).abs() < 1e-9);
+        assert!((weights.dwell_weight_for(&non_mutual) - base_dwell).abs() < 1e-9);
+    }
+
+    #[test]
+    fn bidirectional_weight_default_boost_is_noop() {
+        let query = query_with_flags(&[]);
+        let weights = ScoringWeights::from_params(&query.params);
+        let base_reply = query.params.get(ReplyWeight);
+        let base_dwell = query.params.get(DwellWeight);
+
+        let mutual_original = PostCandidate {
+            is_mutual_follow_author: Some(true),
+            ..candidate(20, Some(true))
+        };
+        assert!((weights.reply_weight_for(&mutual_original) - base_reply).abs() < 1e-9);
+        assert!((weights.dwell_weight_for(&mutual_original) - base_dwell).abs() < 1e-9);
+    }
+
     #[tokio::test]
     async fn fs_off_does_not_discount_in_network_replies_or_retweets() {
 ```


## 2026 年 7 月 24 日：参数调整

2026 年 7 月 24 日，在拿回实验结果并听取了 X 上的反馈之后——例如世界杯正在举行，但有些人看到的讨论没有他们期望的那么多，因为许多帖子来自他们未关注的账号——我们把双向关注回复提升值从 20 调整为 15。

#### `home-mixer/params/param.rs`

```diff
diff --git a/home-mixer/params/param.rs b/home-mixer/params/param.rs
--- a/home-mixer/params/param.rs
+++ b/home-mixer/params/param.rs
@@ -286,7 +286,7 @@ param!(
     BidirectionalFollowReplyWeightBoost,
     f64,
     "rust_home_mixer_bidirectional_follow_reply_weight_boost",
-    20.0
+    15.0
 );
```
