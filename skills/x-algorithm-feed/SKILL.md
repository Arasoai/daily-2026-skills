```markdown
---
name: x-algorithm-feed
description: Rust implementation of X's For You feed ranking algorithm by xai-org
triggers:
  - implement x for you feed algorithm
  - twitter recommendation algorithm rust
  - x feed ranking system
  - build social media feed ranking
  - xai recommendation algorithm
  - for you feed implementation
  - x algorithm ranking rust
  - social feed personalization algorithm
---

# X Algorithm (For You Feed)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

The X Algorithm (`xai-org/x-algorithm`) is the open-source Rust implementation powering the "For You" feed on X (formerly Twitter). It provides candidate sourcing, feature extraction, scoring, and ranking logic for personalizing social media feeds at scale.

---

## What It Does

- **Candidate sourcing**: Fetches tweet candidates from multiple sources (social graph, interest graph, trending)
- **Feature extraction**: Extracts user, tweet, and engagement features
- **Scoring & ranking**: Scores candidates using ML models and heuristics
- **Filtering**: Applies safety, diversity, and relevance filters
- **Serving**: Exposes ranked feed results via a gRPC/REST API

---

## Installation & Setup

### Prerequisites

- Rust 1.75+ (`rustup install stable`)
- Cargo
- (Optional) Docker for containerized deployment

### Clone & Build

```bash
git clone https://github.com/xai-org/x-algorithm.git
cd x-algorithm
cargo build --release
```

### Run Tests

```bash
cargo test
cargo test --lib          # unit tests only
cargo test --integration  # integration tests
```

### Run the Service

```bash
cargo run --release -- --config config/default.toml
```

---

## Configuration

The algorithm is configured via TOML files and environment variables.

### `config/default.toml` (example structure)

```toml
[server]
host = "0.0.0.0"
port = 8080
workers = 8

[ranking]
max_candidates = 1500
final_feed_size = 150
diversity_factor = 0.3
recency_weight = 0.25

[scoring]
model_path = "./models/ranker_v1.bin"
engagement_weight = 0.4
relevance_weight = 0.35
recency_weight = 0.25

[sources]
social_graph_weight = 0.5
interest_graph_weight = 0.3
trending_weight = 0.2

[filters]
safe_search = true
min_quality_score = 0.1
max_age_hours = 48
```

### Environment Variables

```bash
export X_ALGO_CONFIG_PATH=/etc/x-algorithm/config.toml
export X_ALGO_MODEL_PATH=/models/ranker_v1.bin
export X_ALGO_LOG_LEVEL=info
export X_ALGO_METRICS_PORT=9090
export X_ALGO_REDIS_URL=redis://localhost:6379
export X_ALGO_DB_URL=postgres://user:pass@localhost/xalgo
```

---

## Key Modules & API

### Core Data Types

```rust
use x_algorithm::types::{Tweet, User, FeedRequest, FeedResponse, CandidateScore};

// Tweet candidate
let tweet = Tweet {
    id: 1234567890,
    author_id: 987654321,
    text: "Hello world".to_string(),
    created_at: chrono::Utc::now(),
    engagement: EngagementStats {
        likes: 42,
        retweets: 10,
        replies: 5,
        views: 1000,
    },
    ..Default::default()
};

// Feed request
let request = FeedRequest {
    user_id: 111222333,
    cursor: None,
    count: 50,
    context: RequestContext {
        device: DeviceType::Mobile,
        language: "en".to_string(),
    },
};
```

### Pipeline Usage

```rust
use x_algorithm::{
    pipeline::FeedPipeline,
    config::AlgorithmConfig,
    sources::{SocialGraphSource, InterestGraphSource, TrendingSource},
    ranker::NeuralRanker,
    filters::{SafetyFilter, DiversityFilter},
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = AlgorithmConfig::from_file("config/default.toml")?;

    // Build the pipeline
    let pipeline = FeedPipeline::builder()
        .with_source(SocialGraphSource::new(&config))
        .with_source(InterestGraphSource::new(&config))
        .with_source(TrendingSource::new(&config))
        .with_ranker(NeuralRanker::load(&config.scoring.model_path)?)
        .with_filter(SafetyFilter::default())
        .with_filter(DiversityFilter::new(config.ranking.diversity_factor))
        .build();

    // Generate feed for a user
    let request = FeedRequest::for_user(111222333);
    let feed = pipeline.generate_feed(request).await?;

    for item in &feed.items {
        println!("Tweet {} | Score: {:.4}", item.tweet_id, item.score);
    }

    Ok(())
}
```

### Candidate Sourcing

```rust
use x_algorithm::sources::{CandidateSource, SocialGraphSource};

let source = SocialGraphSource::new(&config);

// Fetch candidates for a user
let candidates = source.fetch_candidates(
    user_id,
    FetchOptions {
        max_candidates: 500,
        since_timestamp: chrono::Utc::now() - chrono::Duration::hours(24),
        include_retweets: true,
    }
).await?;

println!("Fetched {} candidates", candidates.len());
```

### Feature Extraction

```rust
use x_algorithm::features::{FeatureExtractor, UserFeatures, TweetFeatures};

let extractor = FeatureExtractor::new(&config);

// Extract features for scoring
let features = extractor.extract(
    &tweet,
    &author,
    &viewer_user,
    &social_context,
).await?;

// Features include:
// - author_follower_count_log
// - tweet_age_seconds
// - engagement_rate
// - semantic_similarity_to_user_interests
// - author_quality_score
// - network_distance
println!("Feature vector: {:?}", features.as_slice());
```

### Scoring & Ranking

```rust
use x_algorithm::ranker::{Ranker, NeuralRanker, ScoredCandidate};

let ranker = NeuralRanker::load("models/ranker_v1.bin")?;

// Score a batch of candidates
let scored: Vec<ScoredCandidate> = ranker
    .score_batch(&candidates, &user_context)
    .await?;

// Sort by score descending
let mut ranked = scored;
ranked.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());

// Take top N
let top_feed: Vec<_> = ranked.into_iter().take(150).collect();
```

### Custom Ranker Implementation

```rust
use x_algorithm::ranker::{Ranker, ScoredCandidate};
use x_algorithm::types::{Candidate, UserContext};
use async_trait::async_trait;

pub struct CustomHeuristicRanker {
    recency_weight: f64,
    engagement_weight: f64,
}

#[async_trait]
impl Ranker for CustomHeuristicRanker {
    async fn score_batch(
        &self,
        candidates: &[Candidate],
        context: &UserContext,
    ) -> anyhow::Result<Vec<ScoredCandidate>> {
        let scored = candidates
            .iter()
            .map(|c| {
                let age_hours = c.tweet.age_hours();
                let recency_score = (-0.1 * age_hours).exp(); // exponential decay
                
                let engagement_rate = c.tweet.engagement.likes as f64
                    / (c.tweet.engagement.views as f64 + 1.0);

                let score = self.recency_weight * recency_score
                    + self.engagement_weight * engagement_rate;

                ScoredCandidate {
                    candidate: c.clone(),
                    score,
                    score_breakdown: ScoreBreakdown {
                        recency: recency_score,
                        engagement: engagement_rate,
                        relevance: 0.0,
                    },
                }
            })
            .collect();

        Ok(scored)
    }
}
```

### Filtering

```rust
use x_algorithm::filters::{Filter, DiversityFilter, FilterResult};

// Diversity filter ensures feed variety
let diversity_filter = DiversityFilter::new(DiversityConfig {
    max_per_author: 2,
    max_same_topic_ratio: 0.3,
    min_content_similarity_distance: 0.15,
});

let filtered = diversity_filter.apply(ranked_candidates)?;

// Safety filter
use x_algorithm::filters::SafetyFilter;

let safety_filter = SafetyFilter::with_config(SafetyConfig {
    block_nsfw: true,
    min_author_quality: 0.2,
    block_spam_signals: true,
});

let safe_feed = safety_filter.apply(filtered)?;
```

---

## Common Patterns

### Full Feed Generation Flow

```rust
use x_algorithm::prelude::*;

pub struct FeedService {
    pipeline: FeedPipeline,
}

impl FeedService {
    pub async fn new() -> anyhow::Result<Self> {
        let config = AlgorithmConfig::from_env()?;
        let pipeline = FeedPipeline::from_config(&config).await?;
        Ok(Self { pipeline })
    }

    pub async fn get_feed(
        &self,
        user_id: u64,
        page_size: usize,
    ) -> anyhow::Result<Vec<RankedTweet>> {
        let request = FeedRequest {
            user_id,
            count: page_size,
            cursor: None,
            context: RequestContext::default(),
        };

        let response = self.pipeline.generate_feed(request).await?;
        Ok(response.items)
    }
}
```

### Observability & Metrics

```rust
use x_algorithm::metrics::{MetricsCollector, FeedMetrics};

// Record feed generation metrics
let metrics = MetricsCollector::global();

let timer = metrics.start_timer("feed_generation_latency");
let feed = pipeline.generate_feed(request).await?;
timer.observe();

metrics.record_feed_stats(FeedMetrics {
    user_id,
    candidates_fetched: feed.debug.candidates_fetched,
    candidates_after_filter: feed.debug.candidates_after_filter,
    final_feed_size: feed.items.len(),
    latency_ms: timer.elapsed_ms(),
});
```

### Batch Processing for Multiple Users

```rust
use futures::stream::{self, StreamExt};

async fn generate_feeds_batch(
    pipeline: &FeedPipeline,
    user_ids: Vec<u64>,
    concurrency: usize,
) -> Vec<anyhow::Result<FeedResponse>> {
    stream::iter(user_ids)
        .map(|user_id| {
            let req = FeedRequest::for_user(user_id);
            pipeline.generate_feed(req)
        })
        .buffer_unordered(concurrency)
        .collect()
        .await
}

// Usage
let results = generate_feeds_batch(&pipeline, user_ids, 16).await;
```

---

## gRPC Service

```rust
// Start the gRPC server
use x_algorithm::server::AlgorithmServer;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = AlgorithmConfig::from_env()?;
    
    let server = AlgorithmServer::new(config).await?;
    
    println!("Starting X Algorithm server on :50051");
    server.serve("[::]:50051".parse()?).await?;
    
    Ok(())
}
```

```proto
// proto/feed.proto (reference)
service FeedAlgorithm {
    rpc GetFeed(FeedRequest) returns (FeedResponse);
    rpc GetCandidates(CandidateRequest) returns (CandidateResponse);
    rpc ScoreTweets(ScoreRequest) returns (ScoreResponse);
}
```

---

## Troubleshooting

### Build Errors

```bash
# Missing system dependencies
sudo apt-get install -y pkg-config libssl-dev

# Update Rust toolchain
rustup update stable
rustup target add x86_64-unknown-linux-musl  # for static builds

# Clear build cache if strange errors
cargo clean && cargo build --release
```

### Model Loading Failures

```bash
# Ensure model file exists
ls -la models/ranker_v1.bin

# Check model path in config
export X_ALGO_MODEL_PATH=$(pwd)/models/ranker_v1.bin

# Verify model format compatibility
cargo run --bin validate-model -- --path models/ranker_v1.bin
```

### Performance Tuning

```toml
# config/production.toml
[server]
workers = 16  # match CPU cores

[ranking]
max_candidates = 1500
batch_size = 256  # tune for GPU/CPU

[cache]
candidate_ttl_seconds = 30
feature_cache_size = 10000
```

```bash
# Profile with flamegraph
cargo install flamegraph
cargo flamegraph --bin x-algorithm -- --config config/default.toml

# Benchmark
cargo bench
```

### Logging

```bash
# Set log level
export RUST_LOG=x_algorithm=debug,info

# Structured JSON logging for production
export X_ALGO_LOG_FORMAT=json
export X_ALGO_LOG_LEVEL=warn
```

---

## Testing Your Integration

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use x_algorithm::testing::{MockCandidateSource, TestFixtures};

    #[tokio::test]
    async fn test_feed_generation() {
        let fixtures = TestFixtures::load();
        
        let pipeline = FeedPipeline::builder()
            .with_source(MockCandidateSource::with_tweets(fixtures.sample_tweets()))
            .with_ranker(x_algorithm::ranker::SimpleRanker::default())
            .build();

        let request = FeedRequest::for_user(fixtures.test_user_id);
        let feed = pipeline.generate_feed(request).await.unwrap();

        assert!(!feed.items.is_empty());
        assert!(feed.items.len() <= 150);
        
        // Verify sorted by score
        let scores: Vec<f64> = feed.items.iter().map(|i| i.score).collect();
        assert!(scores.windows(2).all(|w| w[0] >= w[1]));
    }
}
```

---

## Resources

- **Repository**: https://github.com/xai-org/x-algorithm
- **License**: Apache-2.0
- **Language**: Rust
- **Issues**: Check GitHub Issues for known bugs and feature requests
```
