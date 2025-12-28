# Twitter (X) Algorithm: Comprehensive Technical Study

## Executive Summary

This document provides a deep technical analysis of Twitter's open-sourced recommendation algorithm. The system is a sophisticated, production-grade recommendation engine that serves billions of users with personalized content across multiple surfaces including the For You timeline, Following timeline, Search, and Notifications.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Home Mixer - Timeline Construction](#2-home-mixer---timeline-construction)
3. [Candidate Generation Sources](#3-candidate-generation-sources)
4. [Ranking Systems](#4-ranking-systems)
5. [SimClusters - Community Embeddings](#5-simclusters---community-embeddings)
6. [Earlybird - Real-Time Search](#6-earlybird---real-time-search)
7. [Graph-Based Recommendations](#7-graph-based-recommendations)
8. [Trust & Safety Systems](#8-trust--safety-systems)
9. [Push Notification Service](#9-push-notification-service)
10. [Follow Recommendations](#10-follow-recommendations)
11. [Key Signals & Features](#11-key-signals--features)
12. [Technology Stack](#12-technology-stack)

---

## 1. Architecture Overview

### System Design Philosophy

Twitter's algorithm follows a **multi-stage funnel architecture**:

```
Billions of Tweets
       ↓
Candidate Generation (1000s of candidates)
       ↓
Light Ranking (Fast filtering)
       ↓
Heavy Ranking (ML-based scoring)
       ↓
Filtering & Safety
       ↓
Mixing & Blending
       ↓
~50 Tweets Served
```

### Key Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Home Mixer** | Timeline construction orchestrator | Scala, Product Mixer |
| **Earlybird** | Real-time search index | Java, Lucene |
| **SimClusters** | Community-based embeddings | Scala, Scalding |
| **Navi** | ML model serving | Rust, TensorFlow/ONNX |
| **UTEG** | User-Tweet Entity Graph | Scala, GraphJet |
| **Real-Graph** | User interaction strength | Scala, Scio |
| **Pushservice** | Notification recommendations | Scala |
| **VisibilityLib** | Safety filtering | Scala |

### Data Flow

```
User Request → Home Mixer
                   ↓
    ┌──────────────┼──────────────┐
    ↓              ↓              ↓
Earlybird      TweetMixer      UTEG
(In-Network)   (Out-Network)   (Graph)
    ↓              ↓              ↓
    └──────────────┼──────────────┘
                   ↓
           Feature Hydration
           (~6000 features)
                   ↓
           Heavy Ranker (Navi)
                   ↓
           Safety Filtering
                   ↓
           Content Mixing
           (Ads, WTF, etc.)
                   ↓
           Response
```

---

## 2. Home Mixer - Timeline Construction

### Pipeline Architecture

Home Mixer uses the **Product Mixer** framework with nested pipelines:

```
ForYouProductPipeline
    └── ForYouScoredTweetsMixerPipeline
        ├── ScoredTweetsRecommendationPipeline
        │   ├── Candidate Fetching (9 sources)
        │   ├── Feature Hydration (50+ hydrators)
        │   ├── NaviModelScorer
        │   └── Filtering & Selection
        ├── ForYouAdsCandidatePipeline
        ├── ForYouWhoToFollowPipeline
        └── ForYouWhoToSubscribePipeline
```

### Candidate Sources (Priority Order)

1. **Static/Pinned Tweets** - Manually curated content
2. **Cached Scored Tweets** - Previously computed scores
3. **Earlybird In-Network** - Tweets from followed users (~50%)
4. **UTEG Direct** - Collaborative filtering recommendations
5. **Tweet Mixer** - Out-of-network ML recommendations
6. **Content Exploration** - Topic/community based
7. **Lists** - User's list tweets
8. **Backfill** - Reverse-chronological fallback
9. **Communities** - Community tweets

### In-Network vs Out-of-Network Balance

The system uses **DebunchCandidates** to ensure content variety:
- Maximum 2 consecutive out-of-network tweets
- Debunching algorithm swaps positions to break up recommendation clusters
- Ensures natural mix: "in, in, out, in, out, in..."

### Feature Hydration

~6000 features organized in phases:

**Query Features (Early):**
- User state, device info, language
- Real Graph scores, followed users
- Tweet impressions, feedback history

**Async Features (Parallel):**
- TWHIN embeddings, topic signals
- User engagement aggregates
- Large embeddings (author, tweet)

**Candidate Features:**
- Tweet metadata, engagement counts
- Author info, content classification
- Conversation structure, embeddings

---

## 3. Candidate Generation Sources

### Earlybird (In-Network)

Real-time search index for tweets from followed users:
- **Cluster Architecture**: Realtime, Protected, Archive
- **Indexing Latency**: ~1 second from tweet creation
- **Query Latency**: <100ms p99
- **Partitioning**: By tweet ID or user ID

### UTEG (User Tweet Entity Graph)

GraphJet-based collaborative filtering:
- **Bipartite Graph**: Users ↔ Tweets
- **Edge Types**: Click, Favorite, Retweet, Reply, Quote
- **Algorithm**: TopSecondDegreeByCount (SALSA variant)
- **Real-time Updates**: Kafka ingestion pipeline

### Tweet Mixer

Coordination layer for out-of-network candidates:
- Combines multiple ML-based sources
- SimClusters ANN for embedding similarity
- Topic-based recommendations

### CR-Mixer

Content Recommender Mixer:
- Lightweight candidate generation
- Multiple recommendation types
- Configurable source blending

---

## 4. Ranking Systems

### Two-Stage Ranking

#### Light Ranker (Earlybird/Deepbird)

**Architecture**: Linear model with discretized features

```
Score = BASE + Σ(feature_weight × bucketed_value)
```

**Features (~65)**:
- Text relevance (BM25)
- Author reputation
- Engagement counts (log-scaled)
- Content signals (media, links, replies)
- Safety labels

**Multi-Task Outputs (10)**:
- Main engagement prediction
- Click, Like, Retweet, Reply predictions
- Video playback, Profile click predictions

#### Heavy Ranker (Navi/Phoenix)

**Architecture**: Multi-task neural network

**15 Prediction Heads**:
1. `PredictedFavoriteScore` - P(liked)
2. `PredictedReplyScore` - P(replied)
3. `PredictedRetweetScore` - P(retweeted)
4. `PredictedReplyEngagedByAuthorScore` - P(author engaged)
5. `PredictedGoodClickScore` - P(quality click)
6. `PredictedGoodProfileClickScore` - P(profile click)
7. `PredictedVideoQualityViewScore` - P(video quality view)
8. `PredictedBookmarkScore` - P(bookmarked)
9. `PredictedShareScore` - P(shared)
10. `PredictedDwellScore` - P(dwell time)
11. `PredictedNegativeFeedbackV2Score` - P(negative feedback)
12. + Additional specialized heads

**Score Aggregation**:
```scala
final_score = Σ(normalized_score_i × weight_i)
// With special handling for negative weights
```

### Ranking Flow

```
Candidates
    ↓
NaviModelScorer (batch ML inference)
    ↓
15 engagement head predictions
    ↓
WeighedModelRerankingScorer (combine scores)
    ↓
PhoenixModelRerankingScorer (optional refinement)
    ↓
Heuristic adjustments (diversity, fatigue)
    ↓
Final ranked list
```

---

## 5. SimClusters - Community Embeddings

### Core Concept

SimClusters discovers ~145,000 communities from Twitter's follow graph and generates sparse, interpretable embeddings.

### Community Detection Pipeline

```
Follow Graph
    ↓
Producer-Producer Similarity (cosine on followers)
    ↓
Louvain Clustering (Metropolis-Hastings)
    ↓
KnownFor Matrix (20M producers × 145K clusters)
```

### Embedding Types

| Type | Description | Computation |
|------|-------------|-------------|
| **KnownFor** | What producers are known for | Cluster assignment |
| **InterestedIn** | What users are interested in | Follow graph × KnownFor |
| **Producer** | Multi-topic influence | Cosine(Followers, InterestedIn) |
| **Tweet** | Tweet topic representation | Aggregate engager InterestedIn |

### Tweet Embedding (Real-Time)

```
Tweet Created → Empty Vector
    ↓
User Favorites → Add user's InterestedIn
    ↓
Aggregate → Tweet Embedding
```

### Similarity Metrics

1. **Dot Product** - Raw similarity
2. **Cosine Similarity** - L2-normalized
3. **Log-Norm Cosine** - Reduced magnitude bias
4. **Jaccard/Fuzzy Jaccard** - Set-based

### ANN Candidate Generation

Two-stage process:
1. **Fast Scan**: Top clusters → top tweets per cluster → dot product
2. **Heavy Ranking**: Full embedding cosine similarity

---

## 6. Earlybird - Real-Time Search

### Architecture

```
SuperRoot (entry point)
    ↓
┌───────────┬───────────┬───────────┐
Realtime    Protected   Archive
Root        Root        Root
    ↓           ↓           ↓
Partitions  Partitions  Partitions
    ↓           ↓           ↓
└───────────┴───────────┴───────────┘
    ↓
Response Merger
```

### Index Structure

```
Term Dictionary (FST/MPH)
    ↓
Posting Lists (packed integers + skip lists)
    ↓
Positions (term positions in documents)
    ↓
Column-Stride Fields (engagement metrics)
```

### Scoring Functions

1. **LinearScoringFunction** (primary):
   ```
   score = base + lucene_weight × text_score
         + reputation_weight × user_rep
         + retweet_weight × log2(retweets)
         + fav_weight × log2(favorites)
         + ...
   ```

2. **TensorFlowBasedScoringFunction** (advanced):
   - Neural network scoring
   - Feature tensor construction
   - Real-time inference

### Key Characteristics

- **Indexing Latency**: ~1 second
- **Query Latency**: <100ms p99
- **Segments**: Time-sliced, optimized periodically
- **Partitioning**: Horizontal by tweet ID

---

## 7. Graph-Based Recommendations

### UTEG (User Tweet Entity Graph)

**Structure**: Bipartite graph (Users ↔ Tweets)

**Edge Types**:
- Click, Favorite, Retweet, Reply
- Tweet (authored), Mention, Quote

**Multi-Segment Design**:
- 8 segments, 134M edges each
- Power-law distribution for memory efficiency
- Circular buffer drops oldest

**SALSA Algorithm**:
```
Input: User + seed weights
    ↓
First Degree: Tweets engaged by seeds
    ↓
Second Degree: Users who engaged same tweets
    ↓
Ranking: By engagement weight
    ↓
Social Proof: Connecting users
```

### Real-Graph (Interaction Strength)

**Features Tracked (50+)**:
- Interaction counts (retweets, favorites, mentions, DMs)
- Engagement signals (clicks, dwell time, profile views)
- Status (follows, unfollows, blocks, mutes)
- Address book (email, phone contacts)

**Time-Series Statistics**:
- Mean, EWMA, Variance
- Elapsed days, non-zero days
- Days since last interaction

**Decay Formula**:
```
ewma_t = α × x_t + (1 - α) × ewma_{t-1}
```

### Graph Feature Service

Exposes features for ML models:
- Continuous: engagement means, EWMAs
- Discrete: day counts
- Binary: missingness indicators

---

## 8. Trust & Safety Systems

### ML Safety Models

| Model | Purpose | Architecture |
|-------|---------|--------------|
| **pNSFWMedia** | NSFW image detection | CNN + BERT |
| **pNSFWText** | NSFW text detection | Twitter BERT |
| **pToxicity** | Toxic content | Multilingual BERT |
| **pAbuse** | Abuse/harassment | Multi-label classifier |

### Safety Labels

**Tweet Labels (100+ types)**:
- Abuse: Abusive, AbusiveHighRecall, HatefulConduct
- NSFW: NsfwHighPrecision, NsfwHighRecall, NsfwVideo
- Toxicity: HighToxicityScore (language-specific thresholds)
- Spam: HighSpammyTweetContentScore, Automation
- Misinformation: MisinfoCivic, MisinfoMedical

**User Labels (40+ types)**:
- Abusive, DoNotAmplify, Compromised
- EngagementSpammer, LowQuality
- GoreAndViolenceHighPrecision

### Visibility Actions

| Action | Severity | Effect |
|--------|----------|--------|
| **Drop** | 16 | Remove from feed |
| **Tombstone** | 15 | Show placeholder |
| **Interstitial** | 10 | Warning before viewing |
| **LimitedEngagements** | 6 | Disable interactions |
| **Downrank** | 0 | Reduce visibility |
| **Allow** | -1 | No action |

### Rule Engine

40+ configurable rules:
- TweetLabelRules, UserLabelRules
- ToxicityReplyFilterRules
- DownrankingRules
- SensitiveMediaSettingsRules

---

## 9. Push Notification Service

### Pipeline Stages

```
1. Target Building → Check user eligibility
2. Candidate Fetching → Multiple sources
3. Hydration → Enrich with metadata
4. Pre-Rank Filtering → Quick predicates
5. Light Ranking → MLP pre-filter
6. Heavy Ranking → ClemNet neural network
7. Re-Ranking → Boosting/down-ranking
8. Take Step → Final validation
```

### Candidate Sources

- EarlyBird First Degree (followed tweets)
- FRS Content Recommender
- Trip Geo (location-based)
- Trends, Magic Fanout
- Topic Proof, Discover Twitter

### Ranking Models

**Light Ranker**: MLP with DeepNorm
- Fast pre-filtering
- Top-K selection (~60 candidates)

**Heavy Ranker (ClemNet)**: ResNet-style
- Multi-task: open + engagement prediction
- Producer quality boosting
- Quality upranking

### Producer Quality Boosting

- **High-quality** (3M+ followers OR 1M+ with good health): boost
- **Low-quality** (few followers, high NSFW, abuse): downboost
- Factors: NSFW scores, nudity rate, abuse strikes, engagement metrics

---

## 10. Follow Recommendations

### Candidate Sources (16 Types)

**Graph-Based**:
1. Sims (similar users by follow graph)
2. RealGraph (engagement strength)
3. STP (strong tie prediction)
4. SALSA (PYMK random walks)
5. Sims Expansion (2nd-hop)
6. TwoHopRandomWalk
7. User-User Graph (recent follows)

**Social/Contact**:
8. Address Book (phone/email)
9. Triangular Loops

**Engagement**:
10. Recent Engagement
11. Repeated Profile Visits

**Location/Demographics**:
12. Pop Geo Quality Follow
13. PPMI Locale Follow
14. Crowd Search Accounts
15. Top Organic Follows

### Ranking Pipeline

```
Candidate Generation
    ↓
Pre-Ranker Filters (inactive, excluded, competitors)
    ↓
Weighted Source Blending
    ↓
Feature Hydration (100+ ML features)
    ↓
ML Ranking (DeepbirdScorer)
    ↓
Interleave (A/B experiments)
    ↓
Fatigue Handling
    ↓
Social Proof ("followed by X")
    ↓
Truncation (top 3-5)
```

### Display Locations (40+)

- Sidebar (WTF module)
- Home Timeline
- Explore Tab
- NUX flows (new users)
- Search, Profile, etc.

---

## 11. Key Signals & Features

### User Behavior Signals

**Explicit**:
- Follows, Unfollows, Mutes, Blocks
- Favorites, Retweets, Quote Tweets
- Replies, Shares, Bookmarks, Reports

**Implicit**:
- Clicks, Video watches
- Dwell time, Profile visits
- Notification opens, NTab clicks

### Signal Usage by Component

| Component | Signals Used |
|-----------|--------------|
| SimClusters | Follows, Favorites, Video watches |
| UTEG | All engagement types |
| Real-Graph | Favorites, Replies, Retweets, Clicks |
| FRS | Follows, Engagement, Address book |
| Light Ranker | Clicks, Videos, Engagement |

### Feature Categories

1. **User Features**: Age, followers, verification, health
2. **Tweet Features**: Content, media, engagement counts
3. **Author Features**: Reputation, activity, creator status
4. **Relationship Features**: Follow status, Real-Graph scores
5. **Context Features**: Device, location, time

---

## 12. Technology Stack

### Languages

| Language | Usage |
|----------|-------|
| **Scala** | Core services, batch/streaming jobs |
| **Java** | Earlybird search (Lucene-based) |
| **Python** | ML model training |
| **Rust** | Navi ML serving (high-performance) |

### Frameworks & Technologies

| Category | Technology |
|----------|------------|
| **Batch Processing** | Scalding, Scio (Apache Beam) |
| **Stream Processing** | Summingbird, Heron, Kafka |
| **ML Training** | TensorFlow v1, TWML |
| **ML Serving** | Navi (Rust), supports TF/ONNX/PyTorch |
| **Search** | Apache Lucene |
| **Graphs** | GraphJet (in-memory) |
| **RPC** | Thrift |
| **Build** | Bazel |

### Infrastructure

| Component | Technology |
|-----------|------------|
| **Database** | Manhattan (distributed) |
| **Cache** | Twemcache |
| **Storage** | HDFS |
| **Analytics** | BigQuery |
| **Messaging** | Kafka |

---

## Key Architectural Insights

### 1. Multi-Stage Funnel
Every recommendation flow uses progressive filtering:
- Cheap operations first (heuristics, simple predicates)
- Expensive operations last (ML models, complex hydration)

### 2. Hybrid In-Network + Out-of-Network
~50% tweets from follows, ~50% recommendations:
- In-network: Earlybird search index
- Out-of-network: SimClusters, UTEG, ML models

### 3. Real-Time + Batch Processing
- Real-time: Tweet embeddings, graph updates, search index
- Batch: SimClusters communities, Real-Graph scores, model training

### 4. Safety-First Design
Multiple layers of content filtering:
- ML safety models (NSFW, toxicity, abuse)
- Rule-based visibility engine
- Geographic and legal compliance

### 5. Experimentation Infrastructure
Every component supports A/B testing:
- Feature flags (Deciders)
- Dynamic weight tuning
- Producer-side experiments

### 6. Scale Optimization
Designed for billions of tweets and users:
- Partitioning (horizontal scaling)
- Caching (reduce redundant computation)
- Batching (efficient ML inference)
- Quality factors (graceful degradation)

---

## File Reference Summary

```
/home-mixer/           - Timeline construction service
/product-mixer/        - Recommendation framework
/src/scala/com/twitter/
  ├── simclusters_v2/  - Community embeddings
  ├── recos/           - GraphJet recommendations
  ├── interaction_graph/ - Real-Graph
  └── timelines/       - Timeline ML utilities
/src/java/com/twitter/search/
  └── earlybird/       - Real-time search
/src/python/twitter/deepbird/
  └── projects/timelines/ - Light ranker training
/trust_and_safety_models/ - Safety ML models
/visibilitylib/        - Visibility filtering
/pushservice/          - Push notifications
/follow-recommendations-service/ - Follow suggestions
/navi/                 - ML serving (Rust)
```

---

## Conclusion

Twitter's recommendation algorithm is a sophisticated, production-grade system that:

1. **Balances Relevance & Discovery**: Mixes followed content with personalized recommendations
2. **Prioritizes Safety**: Multiple layers of content filtering and moderation
3. **Optimizes for Engagement**: Multi-head ML models predicting various engagement types
4. **Scales Massively**: Handles billions of tweets and users with low latency
5. **Enables Experimentation**: Every component supports A/B testing and dynamic tuning

The architecture represents years of iteration and optimization, combining traditional information retrieval techniques with modern ML approaches to deliver personalized content at scale.
