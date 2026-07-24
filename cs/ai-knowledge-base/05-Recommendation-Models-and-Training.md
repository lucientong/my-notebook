# Recommendation Models and Training

Language: English | [中文](../AI知识库/05-推荐模型与训练.md)

> This document focuses on recommendation model architecture, embeddings, ranking, retrieval, sequence modeling, multi-task learning, distributed training, online learning, and the connection between LLMs and recommender systems.

---

## 0. Scope

This document is model-centric. It explains recommendation models and training behavior. For end-to-end recommender system architecture, traffic allocation, A/B testing, and degradation strategy, read the recommendation system specialty notes in the Chinese knowledge base.

---

## 1. Embedding Technology

### Concept

Embeddings convert sparse high-cardinality entities, such as user IDs, item IDs, categories, and queries, into dense vectors.

### Mechanism

```text
one-hot ID -> embedding lookup -> dense vector
```

Embedding value:

- reduces dimensionality,
- captures similarity,
- enables feature interaction through dot products or neural layers,
- supports retrieval and ranking.

### Trade-off

Large embedding tables dominate memory and training cost. Low-frequency IDs overfit easily, while new IDs suffer from cold start.

### Production Practice

- Use frequency thresholds and hash buckets.
- Combine ID embeddings with content embeddings.
- Store huge embeddings in parameter servers or sharded embedding services.
- Monitor embedding drift and stale features.
- Keep training and serving feature definitions consistent.

### Interview Self-Check

- Why are embeddings central to recommendation models?
- How do you handle unseen users or items?
- Why can embedding drift harm retrieval quality?

---

## 2. Classic Ranking Models

### Concept

Ranking models estimate the relevance or utility of candidate items for a user and context.

### Mechanism

Common models:

| Model | Core Idea |
|---|---|
| FM | learn second-order feature interactions through latent vectors |
| FFM | field-aware interactions with separate embeddings per field |
| Wide & Deep | combine memorization and generalization |
| DeepFM | share embeddings between FM and DNN components |
| DCN | explicitly model feature crosses with cross layers |

### Trade-off

Linear and FM-style models are interpretable and efficient. Deep models capture more complex interactions but require more data, careful regularization, and better feature pipelines.

### Production Practice

- Start with strong feature baselines.
- Validate feature crosses and leakage.
- Compare offline AUC/logloss with online CTR/CVR/business metrics.
- Track calibration, not just ranking accuracy.

### Interview Self-Check

- Why did Wide & Deep combine two parts?
- What advantage does DeepFM have over Wide & Deep?
- What is the difference between memorization and generalization?

---

## 3. Sequence Recommendation Models

### Concept

Sequence models use user behavior order to capture evolving interests.

### Mechanism

Examples:

| Model | Mechanism |
|---|---|
| DIN | attention over historical behaviors conditioned on target item |
| DIEN | models interest evolution through sequential structure |
| BST | uses Transformer attention over behavior sequences |
| SIM | retrieves long-term behavior subsets before precise modeling |

### Trade-off

Sequence models improve personalization but increase latency, feature complexity, and sensitivity to behavior noise. Long histories are expensive and often need retrieval or truncation.

### Production Practice

- Separate short-term and long-term interests.
- Use time decay and behavior type weights.
- Cap sequence length and select high-signal events.
- Monitor performance by user activity level.

### Interview Self-Check

- Why is average pooling weaker than attention for user history?
- How do you model short-term versus long-term interests?
- Why can long behavior sequences hurt latency and quality?

---

## 4. Multi-Task and Multi-Objective Learning

### Concept

Recommendation systems optimize multiple objectives: click, conversion, watch time, retention, revenue, quality, and safety.

### Mechanism

Common architectures:

- shared-bottom,
- MMOE,
- PLE,
- ESMM for full-space conversion modeling.

Multi-objective ranking may combine scores:

```text
final_score = CTR^a * CVR^b * value^c * quality_penalty
```

### Trade-off

Shared representation improves efficiency but tasks can conflict. Optimizing CTR can harm long-term retention or content quality.

### Production Practice

- Monitor task-specific and business-level metrics.
- Use task weighting, expert routing, or gradient conflict mitigation.
- Analyze metric trade-offs in A/B tests.
- Keep guardrail metrics for long-term user experience.

### Interview Self-Check

- What problem does ESMM solve?
- How do you handle conflicting objectives?
- Why can CTR improvement be bad for user experience?

---

## 5. Retrieval Models

### Concept

Retrieval models select a manageable candidate set from a huge item corpus before ranking.

### Mechanism

Two-tower model:

```text
user features -> user tower -> user embedding
item features -> item tower -> item embedding
score = dot(user_embedding, item_embedding)
```

The item embeddings can be indexed with approximate nearest neighbor search.

### Trade-off

Two-tower retrieval is fast and scalable, but limited by the expressiveness of separate user/item representations. Negative sampling strongly affects training quality.

### Production Practice

- Use hard negatives and in-batch negatives carefully.
- Evaluate recall@k and downstream ranking impact.
- Rebuild or incrementally update ANN indexes.
- Monitor candidate diversity and coverage.

### Interview Self-Check

- Why are two-tower models suitable for retrieval?
- How does negative sampling affect retrieval training?
- How do you evaluate retrieval versus ranking?

---

## 6. Parameter Servers and Distributed Training

### Concept

Recommendation models often have huge sparse embeddings, making parameter servers useful for distributed storage and updates.

### Mechanism

```text
workers
  -> pull sparse embedding rows
  -> compute gradients
  -> push updated rows
parameter servers
  -> store sharded embeddings
```

### Trade-off

Parameter servers handle sparse huge embeddings well, but may introduce consistency issues and network bottlenecks. AllReduce is often better for dense model parameters.

### Production Practice

- Shard embeddings by ID hash or feature group.
- Cache hot embeddings.
- Use asynchronous or bounded-staleness updates carefully.
- Monitor hot keys and update skew.

### Interview Self-Check

- Why do recommender systems often use parameter servers?
- Why is AllReduce not always ideal for sparse embeddings?
- How do you handle hot embeddings?

---

## 7. Online Learning and Model Updates

### Concept

Online learning updates models or embeddings with recent user behavior to reduce freshness gaps.

### Mechanism

Patterns:

- full retraining,
- incremental training,
- streaming feature updates,
- real-time embedding updates,
- nearline ranking model refresh.

### Trade-off

Freshness improves responsiveness but can amplify noise, feedback loops, and delayed feedback bias.

### Production Practice

- Separate stable features from real-time features.
- Use delayed feedback handling for CVR.
- Monitor training-serving skew.
- Keep rollback for model and feature versions.
- Use exploration traffic to reduce bias.

### Interview Self-Check

- What is training-serving skew?
- Why is delayed feedback difficult for CVR?
- How do you prevent feedback loops?

---

## 8. LLMs and Recommendation Systems

### Concept

LLMs can enhance recommendation through content understanding, query rewriting, explanation, conversational recommendation, and synthetic training data.

### Mechanism

Use cases:

- extract item attributes from text/images,
- rewrite user intent into structured features,
- generate explanations,
- build conversational recommenders,
- enrich cold-start item representations,
- generate labels or training examples with human validation.

### Trade-off

LLMs add semantic understanding but are slower, more expensive, and harder to evaluate than classic ranking models. They may hallucinate item attributes or explanations.

### Production Practice

- Use LLMs offline or nearline when latency is strict.
- Use classic retrieval/ranking for high-throughput serving.
- Validate generated attributes.
- Separate explanation generation from ranking decisions.
- Evaluate user trust and factual consistency.

### Interview Self-Check

- Where can LLMs help recommender systems most?
- Why is an LLM not usually the entire ranking system?
- How do you validate LLM-generated item features?

---

## 9. Senior Interview Q&A

### Q1: Online CTR drops after a recommendation model release. How do you separate model, data, and traffic causes?

**Answer**: Start with segmentation and replay. Compare fixed sentinel traffic, feature distributions, retrieval coverage, ranking scores, latency, fallback rate, and experiment allocation. If replayed old traffic also scores differently, suspect model or feature changes. If only live traffic changed, inspect channel mix, user mix, campaigns, content pool changes, and logging delays.

### Q2: Why can offline AUC improve while online business metrics decline?

**Answer**: Offline AUC may optimize the wrong label, ignore position bias, miss delayed feedback, overfit frequent users/items, or fail to represent the current candidate distribution. Online metrics include exploration, latency, diversity, fatigue, retention, and long-term value. A senior answer should connect offline metrics to online decision quality rather than treating AUC as the final goal.

### Q3: How do you detect and fix training-serving skew in recommender systems?

**Answer**: Sample online requests, recompute offline features, and compare values, missing rates, default-value usage, normalization, embedding versions, and cross features. Fix skew by unifying feature definitions, versioning schemas, sharing transformation code, adding consistency checks to release gates, and monitoring input distributions continuously.

### Q4: How should exploration data enter training without polluting the main distribution?

**Answer**: Mark exploration samples explicitly, cap exploration traffic, use high-information exploration rather than pure random exposure, and evaluate long-term contribution separately from immediate CTR. Training should account for exposure bias and propensity where needed. Guardrail metrics such as negative feedback, complaints, and retention protect user experience.

---

## 10. Interview Self-Check

- Explain embeddings and why they dominate recommender model design.
- Compare FM, Wide & Deep, DeepFM, and DCN.
- Explain DIN and why target-aware attention helps.
- Design a two-tower retrieval model and its evaluation plan.
- Explain parameter servers for sparse embeddings.
- Diagnose online metric fluctuation after a model release.
- Explain training-serving skew and how to detect it.
- Design a rollback strategy for a recommendation model.
