# How Tweets Get Ranked and Go Viral: Technical Deep Dive

This document explains exactly how Twitter's algorithm ranks tweets and what causes content to spread beyond followers.

---

## Table of Contents
1. [The Ranking Formula](#1-the-ranking-formula)
2. [Engagement Weights](#2-engagement-weights)
3. [Author Quality Boosts](#3-author-quality-boosts)
4. [Content Type Preferences](#4-content-type-preferences)
5. [The Viral Spread Mechanism](#5-the-viral-spread-mechanism)
6. [What Gets Penalized](#6-what-gets-penalized)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. The Ranking Formula

### Heavy Ranker: 15 Prediction Heads

The ML model predicts 15 different engagement probabilities:

| Prediction | What It Measures |
|------------|------------------|
| `PredictedFavoriteScore` | P(user will like) |
| `PredictedRetweetScore` | P(user will retweet) |
| `PredictedReplyScore` | P(user will reply) |
| `PredictedReplyEngagedByAuthorScore` | P(author will engage with reply) |
| `PredictedGoodClickV1Score` | P(user clicks → then likes/replies) |
| `PredictedGoodClickV2Score` | P(user clicks → dwells >2 seconds) |
| `PredictedGoodProfileClickScore` | P(user clicks profile) |
| `PredictedVideoQualityViewScore` | P(watches video >10s) |
| `PredictedVideoQualityViewImmersiveScore` | P(immersive video view) |
| `PredictedBookmarkScore` | P(user bookmarks) |
| `PredictedShareScore` | P(user shares externally) |
| `PredictedDwellScore` | P(user spends time reading) |
| `PredictedVideoWatchTimeScore` | Expected watch time in ms |
| `PredictedVideoQualityWatchScore` | P(quality video watch) |
| `PredictedNegativeFeedbackV2Score` | P(user gives negative feedback) |

### Final Score Calculation

```
Final Score = Σ(predicted_score[i] × weight[i]) + 0.001

Where weights can range from -10,000 to +10,000
```

Each engagement type has a configurable weight that can be tuned via A/B testing.

---

## 2. Engagement Weights

### What Matters Most (By Weight Potential)

All weights default to 0.0 but can range from **-10,000 to +10,000**:

| Engagement Type | Weight Range | Notes |
|-----------------|--------------|-------|
| **Video Quality View** | -10K to +10K | Videos ≥10 seconds get special tracking |
| **Video Watch Time (ms)** | -10K to +10K | Raw watch duration |
| **Dwell Time** | -10K to +10K | Time spent reading/viewing |
| **Good Clicks** | -10K to +10K | Clicks that lead to engagement |
| **Favorites (Likes)** | -10K to +10K | Core engagement signal |
| **Retweets** | -10K to +10K | Amplification signal |
| **Replies** | -10K to +10K | Conversation signal |
| **Bookmarks** | -10K to +10K | Save-for-later signal |
| **Shares** | -10K to +10K | External sharing |
| **Profile Clicks** | -10K to +10K | Interest in author |

### Earlybird (Light Ranker) Weights

From actual production config for Communities feed:

```scala
replyCountParams = weight: 10000.0   // Replies weighted HIGHEST
favCountParams = weight: 1000.0      // Likes
quotedCountParams = weight: 1000.0   // Quote tweets
```

**Key Insight**: Replies are weighted **10x higher** than likes - the algorithm heavily prioritizes conversation-generating content.

### Log2 Normalization

Engagement counts are log-normalized to prevent gaming:

```java
score += retweetWeight × log2(retweetCount)
score += favWeight × log2(favCount)
score += replyWeight × log2(replyCount)
```

This means:
- 1 like = 0 points (log2(1) = 0)
- 2 likes = 1 point
- 4 likes = 2 points
- 1000 likes = ~10 points
- 1M likes = ~20 points

Diminishing returns prevent raw engagement farming.

---

## 3. Author Quality Boosts

### Multiplicative Boost Factors

These are **multiplied** to the base score, amplifying high-quality content:

```java
// From FeatureBasedScoringFunction.java
if (isFromVerifiedAccount) {
    boostedScore *= tweetFromVerifiedAccountBoost;  // Legacy verified
}

if (isFromBlueVerifiedAccount) {
    boostedScore *= tweetFromBlueVerifiedAccountBoost;  // Twitter Blue
}
```

### Author Reputation Score

Each author has a reputation score (0-255 scale):

```java
data.reputationContrib = params.reputationWeight × data.userRep;
```

Higher reputation = higher score contribution.

### Relationship-Based Boosts

```java
if (isFollow) {
    // User follows the author - HIGHEST priority
    boostedScore *= directFollowBoost;
} else if (isTrusted) {
    // Author is in trusted network
    boostedScore *= trustedCircleBoost;
}

if (isSelfTweet) {
    // Your own tweets get boosted in your timeline
    boostedScore *= selfTweetBoost;
}
```

### Real-Graph Weighting

Engagement from high-quality users matters more:

```scala
InNetworkFavoritesAvgRealGraphWeight   // Average quality of likers
InNetworkFavoritesMaxRealGraphWeight   // Best liker's quality
```

A like from someone with 1M followers weighs more than a like from a new account.

---

## 4. Content Type Preferences

### Media Boosts

```java
if (hasImageUrl || hasVideoUrl) {
    boostedScore *= tweetHasMediaUrlBoost;  // Images/videos boosted
}

if (hasNewsUrl) {
    boostedScore *= tweetHasNewsUrlBoost;   // News links boosted
}
```

### Video Quality View (VQV) Special Handling

Videos ≥10 seconds get special tracking:

```scala
// VQV eligibility
if (hasVideo && videoDuration >= 10.seconds) {
    enableVQVTracking = true
}

// VQV thresholds
if (vqvScore < ScoreThresholdForVQV) {
    score = 0.0  // Below threshold = no boost
}
```

### Trending Content

```java
if (hasTrend) {
    boostedScore *= tweetHasTrendBoost;  // Single trend = boost
}

if (hasMultipleHashtagsOrTrends) {
    boostedScore *= multipleHashtagsOrTrendsDamping;  // Multiple = PENALTY
}
```

**Key Insight**: Using one relevant trend helps; hashtag spam hurts.

---

## 5. The Viral Spread Mechanism

### How Tweets Escape Your Follower Network

Twitter uses three main systems to spread content beyond followers:

#### A. UTEG (User Tweet Entity Graph)

This powers the "Liked by [Person]" recommendations:

```
Your followers engage with a tweet
    ↓
UTEG finds users with similar follow graphs
    ↓
Tweet shown to those users with social proof
    ↓
"Liked by [Person you follow]"
    ↓
More engagement → More distribution
```

**The Exponential Effect**:
- 1 like → Shown to liker's followers who have similar interests
- 5 likes → Union of all 5 likers' similar followers
- Each liker might have 1000+ similar followers
- 5 likes = potential reach of 5,000+ users

**Real-time window**: UTEG only tracks last 24-48 hours, so fresh engagement matters.

#### B. SimClusters (Community Embeddings)

When users engage with your tweet, it gets embedded into their community clusters:

```
Tweet Created → Empty embedding vector
    ↓
User A (in tech cluster) likes → Tech cluster added
    ↓
User B (in sports cluster) likes → Sports cluster added
    ↓
Tweet now discoverable by ANYONE interested in tech OR sports
```

~145,000 communities exist. Each engagement from a new cluster expands reach to that entire community.

#### C. Social Proof Loop

Existing engagement is shown to new users, driving more engagement:

```
Tweet has 5 likes
    ↓
Shown to User X as "Liked by A, B, C"
    ↓
User X more likely to engage (social proof)
    ↓
Now has 6 likes
    ↓
Shown to more users with stronger social proof
    ↓
Positive feedback loop
```

### The 50/50 Split

Timeline composition:
- **~50% In-network**: Tweets from people you follow
- **~50% Out-of-network**: Algorithmic recommendations (UTEG, SimClusters)

Out-of-network tweets get a **0.75x scale factor** (25% penalty), but high-engagement content overcomes this.

### Out-of-Network Rescoring

```scala
OutOfNetworkScaleFactorParam = 0.75  // Default 25% reduction

// But engagement scores can easily overcome this:
// A tweet with 10x the engagement of in-network tweets
// will still rank higher despite the 0.75x penalty
```

---

## 6. What Gets Penalized

### Negative Engagement Signals

```scala
NegativeFeedbackV2Param    // "See less" / "Not interested"
ReportParam                // Reports (range: -20K to 0 only)
WeakNegativeFeedbackParam  // Mild negative signals
StrongNegativeFeedbackParam // Strong negative signals
```

Reports have **only negative weight** - they can never help, only hurt.

### Author Quality Damping

```java
if (isUserSpam) {
    boostedScore *= spamUserDamping;     // Spam account penalty
}
if (isUserNSFW) {
    boostedScore *= nsfwUserDamping;     // NSFW account penalty
}
if (isUserBot) {
    boostedScore *= botUserDamping;      // Bot account penalty
}
if (isOffensive) {
    boostedScore *= offensiveDamping;    // Offensive content penalty
}
```

### Out-of-Network Reply Penalty

```java
if (!isFollow && !isTrusted && isReply) {
    boostedScore -= outOfNetworkReplyPenalty;  // Subtraction, not multiplication
}
```

Replies from strangers are penalized.

### Author Diversity Decay

Seeing multiple tweets from the same author causes decay:

```scala
decay_factor = (1 - floor) × (0.5 ^ position) + floor

Position 0: 1.0 (full score)
Position 1: 0.625
Position 2: 0.4375
Position 3: 0.34375
Position ∞: 0.25 (minimum 25% of original score)
```

The algorithm prevents any single author from dominating your feed.

### Feedback Fatigue

If you marked "See Fewer" for an author or topic:
- Their tweets get exponentially decayed
- Effect diminishes over time but persists

---

## 7. Key Takeaways

### What Makes Tweets Rank Higher

1. **Replies matter most** - 10x weight vs likes in some contexts
2. **Video watch time** - Especially videos ≥10 seconds
3. **Dwell time** - Time spent reading/viewing
4. **Quality clicks** - Clicks that lead to further engagement
5. **Bookmarks and shares** - Strong intent signals
6. **Author reputation** - Verified, Blue, high follower count
7. **Relationship** - Tweets from follows/trusted circle boosted

### What Makes Tweets Go Viral

1. **Early engagement velocity** - First few engagements matter (24-48 hour UTEG window)
2. **Diverse engagers** - Engagement from different SimClusters communities
3. **Social proof** - "Liked by [Person]" drives more engagement
4. **Quality of engagers** - High Real-Graph weight users boost more
5. **Media content** - Images and videos get boosted
6. **Single relevant trend** - One hashtag helps, many hurt

### What Hurts Distribution

1. **Reports** - Only negative weight possible
2. **"See less" feedback** - Creates fatigue decay
3. **Spam/bot signals** - Author damping applied
4. **Multiple hashtags** - Treated as spam
5. **Out-of-network replies** - Penalized
6. **Low engagement velocity** - Falls out of UTEG window

### The Viral Formula

```
High Early Engagement (especially replies)
    +
Diverse Community Engagement (SimClusters)
    +
High-Quality Engagers (Real-Graph weight)
    +
Social Proof ("Liked by..." display)
    +
Positive Feedback Loop
    =
Viral Spread
```

---

## File References

| Topic | Key Files |
|-------|-----------|
| **Heavy Ranker Weights** | `home-mixer/param/HomeGlobalParams.scala` (lines 786-1028) |
| **Score Aggregation** | `home-mixer/util/RerankerUtil.scala` |
| **15 Prediction Heads** | `home-mixer/model/PredictedScoreFeature.scala` |
| **Earlybird Scoring** | `src/java/com/twitter/search/earlybird/search/relevance/scoring/LinearScoringFunction.java` |
| **Boost Factors** | `src/java/com/twitter/search/earlybird/search/relevance/scoring/FeatureBasedScoringFunction.java` |
| **UTEG Graph** | `src/scala/com/twitter/recos/user_tweet_entity_graph/` |
| **SimClusters** | `src/scala/com/twitter/simclusters_v2/` |
| **Real-Graph** | `src/scala/com/twitter/interaction_graph/` |
| **Rescoring** | `home-mixer/product/scored_tweets/scorer/RescoringFactorProvider.scala` |
