# AI Overview and Learning Path

Language: English | [中文](../AI知识库/00-AI入门与全景认知.md)

> This document is the entry point of the AI knowledge base. It separates model concepts, application patterns, engineering systems, and training optimization so you do not jump into Transformer formulas, agent frameworks, or fine-tuning details too early.

---

## 1. Build the Big Picture First

Many beginners mix four different layers:

- **Model fundamentals**: why LLMs generate text, what training objective they use, and how tokens and context windows work.
- **Application orchestration**: how prompts, RAG, tool calling, memory, and agents are combined.
- **System engineering**: how to control cost, latency, evaluation, permissions, observability, and rollout risk.
- **Model training and optimization**: how to fine-tune, align, quantize, distill, and deploy models.

These layers form a chain:

```text
Model fundamentals
  -> Application orchestration
  -> Production engineering
  -> Training and optimization
```

If you study LoRA, DPO, or quantization before understanding the first three layers, your knowledge structure will be inverted.

---

## 2. What an LLM Is

### Concept

An LLM, or Large Language Model, is a probabilistic model that predicts the next token given a context.

It is not a database, a search engine, or a deterministic rule engine. It is good at:

- generating natural language,
- compressing and recombining language patterns,
- generalizing within and near its training distribution.

It is not naturally good at:

- precise, long-term, traceable factual memory,
- real-time access to external systems,
- deterministic execution in high-risk workflows.

This is why production AI systems add RAG, tool calling, agents, evaluation, and safety controls.

### Mechanism

```text
Input text -> tokenizer -> token sequence -> model probability distribution -> next token
```

The model repeatedly predicts the next token until it reaches a stop condition. Generation quality depends on the model, context, decoding parameters, and external controls.

### Trade-off

The same probabilistic generation that makes LLMs flexible also creates uncertainty. You get broad capability, but you must manage hallucination, instability, cost, and latency.

### Production Practice

Use the model as one component in a controlled system:

- retrieve private or fresh knowledge instead of relying only on model memory,
- use tools for real-world state and actions,
- validate structured outputs,
- log prompts, retrieved context, tool calls, model responses, and user feedback,
- evaluate both offline and online.

### Interview Self-Check

- Why is an LLM not a database?
- Why does next-token prediction still produce useful reasoning-like behavior?
- Why do RAG and tool calling exist if the model is already powerful?

---

## 3. Why an AI Application Is Not Just an API Call

### Concept

A minimal AI application may only call a model, but a production AI system must manage context, tools, safety, reliability, cost, and feedback.

### Mechanism

Simple flow:

```text
User query
  -> prompt construction
  -> model call
  -> response parsing
  -> answer
```

Production flow:

```text
User query
  -> intent detection / routing
  -> context construction
  -> RAG / memory / user profile
  -> optional tool call
  -> model generation
  -> structured parsing / validation / safety filtering
  -> response + logs + metrics
```

### Trade-off

A simple API call is fast to build but hard to control. A full system is more reliable but introduces more moving parts: retrieval errors, tool failures, schema drift, latency, and permission risks.

### Production Practice

Before shipping, ask:

- What can go wrong if the model hallucinates?
- Which tools can the model access, and under what permissions?
- How do we detect regressions after changing prompts or models?
- How do we control token cost and tail latency?
- How do we trace a bad answer back to its source?

### Interview Self-Check

- How would you debug an AI answer that is fluent but factually wrong?
- When should you add RAG, and when is prompt-only enough?
- Why does tool calling require schema and permission design?

---

## 4. Prompt, RAG, Tool Calling, and Agent

### Concept

These four terms solve different problems:

| Pattern | Solves | Typical Use |
|---|---|---|
| Prompt | Task definition and output constraint | Simple generation, extraction, classification |
| RAG | Missing or private knowledge | Internal knowledge base, documentation Q&A |
| Tool Calling | Real-world capability | Database query, API call, code execution |
| Agent | Multi-step decision and execution | Planning, tool coordination, long-running tasks |

### Mechanism

- Prompt controls the instruction and context passed to the model.
- RAG retrieves external information and injects it into the context.
- Tool calling lets the model select structured operations and arguments.
- An agent wraps prompts, tools, memory, planning, and state into a workflow.

### Trade-off

Do not start with an agent by default. Agents increase autonomy but also increase cost, latency, nondeterminism, and safety risk. Many use cases only need prompt plus RAG plus one controlled tool.

### Production Practice

Choose the simplest pattern that solves the problem:

- fixed FAQ: prompt template,
- private knowledge: RAG,
- real data or actions: tool calling,
- multi-step workflow: agent,
- stable task style change: fine-tuning.

### Interview Self-Check

- What is the difference between RAG and fine-tuning?
- Why is an agent a system pattern rather than just a stronger model?
- How would you decide whether a task needs an agent?

---

## 5. Core Components of an AI System

```text
Frontend / API
  -> business layer
  -> model router
  -> context builder
       -> prompt
       -> RAG
       -> memory
       -> user context
  -> model call
  -> tools / data
  -> evaluation, logging, monitoring, safety
```

The architecture is designed around five engineering concerns:

1. **Quality**: Is the answer useful and stable?
2. **Cost**: Are token usage and model pricing controlled?
3. **Latency**: Is time-to-first-token and end-to-end latency acceptable?
4. **Safety**: Are prompt injection, privilege escalation, and data leakage controlled?
5. **Operability**: Can we evaluate, debug, roll back, and improve the system?

---

## 6. The First Concepts to Master

### Model View

- Transformer and attention
- Tokenization
- Context window
- Temperature and top-p
- KV cache

### Application View

- Prompt structure
- RAG chunking and retrieval
- Tool schema
- Agent state and memory

### Engineering View

- Caching, rate limiting, fallback
- Offline evaluation and online monitoring
- Tracing and feedback loops
- Permission boundaries for tools

---

## 7. Common Beginner Mistakes

- Treating an LLM as a database.
- Treating RAG as a universal solution.
- Treating prompt engineering as magic wording instead of task definition and context control.
- Treating agents as the default architecture.
- Studying fine-tuning and quantization before understanding the production inference chain.

---

## 8. Recommended Order

Start with:

1. [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md)
2. [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md)
3. [02-MCP-and-Tool-Integration.md](./02-MCP-and-Tool-Integration.md)

Then choose a direction:

- Training: [04-AI-Model-Training-and-Optimization.md](./04-AI-Model-Training-and-Optimization.md)
- Recommendation models: [05-Recommendation-Models-and-Training.md](./05-Recommendation-Models-and-Training.md)
- Claude and agent architecture: [06-Claude-Architect-Certification-Knowledge-System.md](./06-Claude-Architect-Certification-Knowledge-System.md)
- Multimodal systems: [07-Multimodal-and-Vision-AI.md](./07-Multimodal-and-Vision-AI.md)
- Safety and alignment: [08-AI-Safety-and-Alignment.md](./08-AI-Safety-and-Alignment.md)

---

## 9. Senior Interview Q&A

### Q1: A team wants to build an internal knowledge assistant. What should you design first?

**Answer**: Start with the product boundary and evaluation plan, not the model choice. Define supported question types, data permissions, source freshness, citation requirements, failure behavior, and human escalation. Then build a simple RAG pipeline with logging, retrieval evaluation, answer evaluation, and rollback. A senior answer should make clear that quality comes from the full system: documents, retrieval, prompts, model behavior, safety controls, and feedback loops.

### Q2: When is a single model API call acceptable, and when is it not?

**Answer**: A single API call is acceptable for low-risk generation, summarization, rewriting, or classification where stale knowledge and occasional variation are tolerable. It is not acceptable when the answer must use private data, trigger external actions, satisfy compliance requirements, or support operational debugging. In those cases, add retrieval, tool permission checks, structured validation, tracing, and release gates.

### Q3: How do you explain the AI learning path to someone who jumps directly into fine-tuning?

**Answer**: Fine-tuning changes model behavior, but many production issues are caused by missing context, weak retrieval, poor prompts, unsafe tools, or absent evaluation. Learn the inference chain first: prompt, context, model call, tool/RAG, validation, monitoring, and feedback. Then study training topics as one tool in a broader decision tree.

---

## 10. Interview Self-Check

You should be able to answer:

- What is the boundary between LLM, RAG, tool calling, and agent?
- Why is a production AI application more than a single model API call?
- How do you decide between prompt, RAG, tool, agent, and fine-tuning?
- Why are evaluation and observability first-class concerns in AI systems?
- What would you build first for a new internal knowledge assistant?
