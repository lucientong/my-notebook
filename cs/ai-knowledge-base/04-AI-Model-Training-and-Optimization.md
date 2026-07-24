# AI Model Training and Optimization

Language: English | [中文](../AI知识库/04-AI模型训练与优化.md)

> This document explains the model lifecycle: pretraining, supervised fine-tuning, alignment, parameter-efficient tuning, quantization, inference optimization, and distributed training.

---

## 0. What This Document Solves

Training topics are often memorized as isolated terms: LoRA, QLoRA, RLHF, DPO, GPTQ, AWQ, ZeRO. The right interview structure is to place each term in the model lifecycle.

```text
Raw data
  -> pretraining
  -> base model
  -> SFT / fine-tuning
  -> instruction model
  -> RLHF / DPO
  -> aligned model
  -> quantization / distillation / inference optimization
  -> production model
```

---

## 1. Training Lifecycle

### Concept

Training and optimization are not one step. They are a sequence of stages that solve different problems.

### Mechanism

| Stage | What It Does | Main Problem Solved |
|---|---|---|
| Pretraining | next-token prediction on large corpora | language patterns and broad knowledge |
| SFT | supervised examples of tasks and instructions | assistant-style behavior |
| Alignment | human or preference-based feedback | helpfulness, harmlessness, preference matching |
| Compression | quantization, pruning, distillation | lower cost and faster inference |
| Deployment optimization | batching, KV cache, serving engine | production latency and throughput |

### Trade-off

Better model behavior can come from training, retrieval, prompting, tools, or product constraints. Training is powerful but expensive and slower to iterate than system-level controls.

### Production Practice

Before training, ask:

- Is the issue missing knowledge, wrong style, weak reasoning, poor format, or unsafe behavior?
- Can prompt/RAG/tooling solve it first?
- Do we have high-quality labeled or preference data?
- How will we evaluate before and after training?

### Interview Self-Check

- Why does pretraining not make a model a helpful assistant by itself?
- When is SFT enough, and when do you need alignment?
- When should you not fine-tune?

---

## 2. Training Basics

### Concept

Language model training optimizes model parameters so predicted token distributions match training data.

### Mechanism

Core pieces:

- dataset and tokenizer,
- model architecture,
- loss function such as cross-entropy,
- optimizer such as AdamW,
- learning rate schedule,
- gradient backpropagation,
- validation metrics such as loss and perplexity.

For next-token prediction:

```text
input:  "The cat sits on the"
target: "cat sits on the mat"
```

The model learns to predict the next token at each position.

### Trade-off

Lower loss does not always mean better user-facing quality. Data quality, task distribution, alignment, and evaluation design matter as much as raw training loss.

### Production Practice

- Deduplicate and filter training data.
- Separate training, validation, and test sets.
- Track data lineage.
- Monitor overfitting and distribution mismatch.
- Keep evaluation sets that reflect real product tasks.

### Interview Self-Check

- What does cross-entropy loss optimize in language modeling?
- Why is perplexity not enough for evaluating assistants?
- Why is data quality often more important than more data?

---

## 3. Fine-Tuning

### Concept

Fine-tuning adapts a pretrained model to a task, domain, style, or instruction format.

### Mechanism

Common methods:

| Method | Mechanism | Use Case |
|---|---|---|
| Full fine-tuning | update all parameters | high-quality adaptation with enough data and compute |
| LoRA | train low-rank adapter matrices | efficient adaptation |
| QLoRA | LoRA with quantized base model | low-memory fine-tuning |
| Prefix / Prompt tuning | train continuous prompt vectors | lightweight task adaptation |

### Trade-off

Full fine-tuning is powerful but expensive and can cause catastrophic forgetting. LoRA is cheaper and modular but may have limited capacity for deep behavior changes.

### Production Practice

- Use RAG for private knowledge before fine-tuning.
- Use fine-tuning for repeated format, style, or task behavior.
- Keep adapters versioned and evaluated.
- Test for regressions outside the fine-tuned domain.
- Prefer small, high-quality datasets over large noisy ones.

### Interview Self-Check

- What problem does LoRA solve?
- Why does QLoRA reduce memory usage?
- How would you decide between RAG and fine-tuning?

---

## 4. RLHF, DPO, and Preference Alignment

### Concept

Alignment trains a model to produce responses that better match human preferences, safety constraints, and helpful behavior.

### Mechanism

RLHF has three common stages:

```text
SFT model
  -> collect preference rankings
  -> train reward model
  -> optimize policy with PPO under KL constraint
```

DPO, or Direct Preference Optimization, skips explicit reward-model reinforcement learning and directly optimizes chosen responses over rejected responses.

### Trade-off

RLHF is flexible but complex and unstable. DPO is simpler and often easier to train, but depends heavily on preference data quality. Alignment can improve helpfulness and safety while sometimes reducing creativity or introducing refusal overreach.

### Production Practice

- Collect diverse preference data.
- Track helpfulness, harmlessness, and truthfulness separately.
- Evaluate refusal behavior and over-refusal.
- Watch for reward hacking.
- Keep human review for high-risk categories.

### Interview Self-Check

- What are the stages of RLHF?
- Why is PPO used with a KL penalty?
- How does DPO differ from RLHF?
- Why can alignment make a model sound better without being more factual?

---

## 5. Quantization and Compression

### Concept

Quantization reduces numeric precision of model weights or activations to lower memory and inference cost.

### Mechanism

Common methods:

| Method | Description |
|---|---|
| PTQ | post-training quantization without retraining |
| QAT | quantization-aware training |
| GPTQ | layer-wise post-training quantization for LLMs |
| AWQ | activation-aware weight quantization |
| Distillation | train a smaller student model from a larger teacher |

### Trade-off

Lower precision improves memory and speed but can reduce accuracy, especially for reasoning, rare tokens, and domain-specific tasks. Distillation can preserve behavior but requires high-quality teacher outputs.

### Production Practice

- Benchmark quantized models on real tasks, not only generic benchmarks.
- Measure quality, latency, throughput, memory, and cost.
- Use stronger models for high-risk requests if quantized models degrade.
- Keep rollback to the full-precision or previous quantized version.

### Interview Self-Check

- Why does quantization reduce deployment cost?
- What quality risks does quantization introduce?
- How is distillation different from quantization?

---

## 6. Inference and Deployment Optimization

### Concept

Inference optimization makes trained models serve requests faster and cheaper.

### Mechanism

Techniques:

- KV cache,
- continuous batching,
- paged attention,
- FlashAttention,
- speculative decoding,
- model parallel serving,
- response streaming,
- request routing.

### Trade-off

Serving optimizations improve throughput and latency but can complicate scheduler behavior, memory management, and debugging.

### Production Practice

- Track time to first token and tokens per second.
- Use batching for throughput-sensitive workloads.
- Use streaming for user-facing chat.
- Separate latency-critical and batch workloads.
- Monitor GPU memory fragmentation and queue time.

### Interview Self-Check

- Why does KV cache speed up autoregressive decoding?
- What is the difference between throughput and latency optimization?
- When is batching harmful?

---

## 7. Distributed Training

### Concept

Distributed training is required when model size, data volume, or batch size exceeds a single device.

### Mechanism

Parallelism types:

| Type | Splits | Typical Use |
|---|---|---|
| Data parallelism | data batches | scale training data throughput |
| Tensor parallelism | matrix operations | split large layers |
| Pipeline parallelism | model layers | split model stages |
| ZeRO | optimizer states, gradients, parameters | reduce memory duplication |

### Trade-off

Distributed training increases scale but adds communication overhead, failure complexity, load balancing issues, and debugging difficulty.

### Production Practice

- Profile communication versus computation.
- Use checkpointing and resume strategy.
- Track GPU utilization and stragglers.
- Validate numerical stability.
- Keep reproducibility metadata: code, data, seed, config, checkpoint.

### Interview Self-Check

- Why is AllReduce used in data parallel training?
- What problem does ZeRO solve?
- When would you combine data, tensor, and pipeline parallelism?

---

## 8. Senior Interview Q&A

### Q1: Training loss improves but online quality drops. What do you check first?

**Answer**: Check objective mismatch before increasing training. Verify whether the validation set reflects real traffic, whether preprocessing and tokenizer versions match serving, whether evaluation covers safety and refusal behavior, and whether the model overfits to formatting or annotator preference. Then inspect data drift, distribution shift, and deployment differences.

### Q2: How do you decide whether to use LoRA, QLoRA, or full fine-tuning?

**Answer**: Use LoRA when you need efficient task or style adaptation and can keep the base model stable. Use QLoRA when GPU memory is the main constraint and small quality trade-offs are acceptable. Use full fine-tuning only when you have enough data, compute, evaluation coverage, and a strong reason to update all parameters. For most product teams, adapter-based tuning plus RAG and evaluation is the safer first step.

### Q3: Quantization reduced latency but damaged quality. How would you recover?

**Answer**: Run task-level benchmarks to identify which capabilities degraded, then try better calibration data, mixed-precision fallback for sensitive layers, a less aggressive bit width, or route high-risk requests to a higher-quality model. Also measure throughput, p99 latency, memory, and cost together. Quantization is a deployment trade-off, not a universal win.

### Q4: Distributed training OOMs frequently. What options exist besides reducing batch size?

**Answer**: Use activation checkpointing, mixed precision, ZeRO/FSDP sharding, sequence-length bucketing, gradient accumulation, optimizer state partitioning, and careful checkpoint/resume strategy. Watch for hidden costs: lower memory can reduce throughput or increase communication overhead.

---

## 9. Interview Self-Check

- Explain the full model lifecycle from pretraining to deployment.
- Compare full fine-tuning, LoRA, and QLoRA.
- Compare RLHF and DPO.
- Explain why quantization can hurt quality.
- Design an evaluation plan for a fine-tuned model.
- Explain how you would train and deploy a smaller domain-specific assistant.
