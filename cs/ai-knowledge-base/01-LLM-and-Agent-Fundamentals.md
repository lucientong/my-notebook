# LLM and Agent Fundamentals

Language: English | [中文](../AI知识库/01-LLM与Agent基础.md)

> This document explains the mechanism side of LLMs, prompts, RAG, tool calling, memory, agents, and hallucination. For production governance, SLOs, rollout, and rollback, read [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md).

---

## 0. What This Document Solves

A strong interview answer should connect four questions:

- What is an LLM, and why can it generate useful language?
- What problems do prompt engineering, RAG, tool calling, and agents solve?
- Why is an AI application a system, not just a model call?
- When quality is poor, should you suspect the model, knowledge, tool, or workflow?

Minimal mental model:

```text
Text input
  -> tokenizer
  -> LLM predicts next-token distribution
  -> RAG if knowledge is missing
  -> tool calling if real-world actions are needed
  -> agent if multi-step planning is needed
```

---

## 1. LLM Core Concepts

### Concept

An LLM is a large neural language model trained to predict tokens from context. After pretraining, instruction tuning, and alignment, it becomes useful as an assistant.

### Mechanism

Typical lifecycle:

```text
Raw text data
  -> cleaning / deduplication / filtering
  -> tokenizer training
  -> pretraining
  -> supervised fine-tuning
  -> preference alignment such as RLHF or DPO
  -> compression and inference optimization
  -> RAG / tools / agents in production
```

Key dimensions:

| Dimension | Main Effect | Does Not Guarantee |
|---|---|---|
| Parameters | Model capacity and pattern complexity | factual correctness |
| Training tokens | Exposure to language and knowledge distribution | high-quality alignment |
| Context window | How much input the model can see at once | long-term memory |

### Trade-off

Larger models often improve reasoning, robustness, and language quality, but increase cost, latency, memory usage, and operational complexity. Long context improves recall of supplied information but can worsen latency and "lost in the middle" behavior.

### Production Practice

- Use a model router to match model size to task complexity.
- Keep prompts concise and structured.
- Use RAG for private or fresh knowledge.
- Use tools for deterministic data access and actions.
- Evaluate regressions when changing model, prompt, retrieval, or decoding parameters.

### Interview Self-Check

- Why does a larger context window not equal permanent memory?
- Why can alignment improve helpfulness without guaranteeing truthfulness?
- How would you choose between a larger model and a better retrieval pipeline?

---

## 2. Transformer, Attention, and Tokenization

### Concept

The Transformer architecture uses self-attention to let each token condition on other tokens in the sequence. Tokenization converts text into discrete units the model can process.

### Mechanism

Self-attention computes query, key, and value vectors:

```text
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V
```

This lets the model assign different weights to different tokens in the context. Position encodings provide order information because attention itself is permutation-insensitive.

Tokenization matters because:

- cost is usually measured in tokens,
- context windows are token limits, not character limits,
- multilingual and code text may tokenize differently,
- chunking strategy in RAG depends on token boundaries.

### Trade-off

Self-attention is powerful but expensive because naive attention scales quadratically with sequence length. Optimizations such as KV cache, FlashAttention, paged attention, batching, and speculative decoding reduce inference cost but add implementation complexity.

### Production Practice

- Estimate token budgets before designing prompts and retrieval chunks.
- Track time to first token and total generation latency.
- Use KV cache for autoregressive decoding.
- Avoid sending irrelevant context just because the model supports long context.

### Interview Self-Check

- Why did Transformers largely replace RNNs for LLMs?
- What does KV cache cache, and why does it speed up decoding?
- Why can long context be expensive even if it is supported?

---

## 3. Prompt Engineering

### Concept

Prompt engineering is task specification, context control, and output constraint design. It is not magic wording.

### Mechanism

A good prompt often includes:

- role and objective,
- relevant context,
- constraints and forbidden behavior,
- output schema,
- examples for ambiguous tasks,
- evaluation criteria or self-check instructions.

Common patterns:

- zero-shot for simple tasks,
- few-shot for format and style consistency,
- chain-of-thought style decomposition when reasoning is needed,
- prompt chaining when a task is safer as multiple validated steps.

### Trade-off

Prompting is fast to iterate and does not require training, but it is sensitive to context, model version, and prompt injection. It cannot reliably teach new private knowledge by itself.

### Production Practice

- Version prompts like code.
- Keep prompt templates separate from runtime data.
- Validate structured outputs with schema.
- Use tests for representative prompts and edge cases.
- Monitor model drift after model upgrades.

### Interview Self-Check

- What is the difference between prompt constraints and business validation?
- When would you use few-shot examples?
- How do you prevent prompt templates from becoming unmaintainable?

---

## 4. RAG: Retrieval-Augmented Generation

### Concept

RAG retrieves external knowledge and injects it into the model context before generation. It is used when the model lacks private, fresh, or traceable knowledge.

### Mechanism

```text
Documents
  -> chunking
  -> embedding
  -> vector / hybrid index
  -> retrieval
  -> reranking
  -> context construction
  -> generation
  -> citation / answer validation
```

RAG failures usually come from:

- poor chunking,
- weak embedding model,
- missing metadata filters,
- low-quality retrieval query,
- no reranking,
- context too long or noisy,
- generation ignoring retrieved evidence.

### Trade-off

RAG avoids retraining and improves freshness, but adds retrieval latency and can fail silently. It improves access to knowledge, not reasoning correctness by itself.

### Production Practice

- Use hybrid retrieval when exact terms matter.
- Add metadata filtering for permissions and versioning.
- Rerank top candidates before context injection.
- Require citations for factual answers.
- Evaluate retrieval recall separately from answer quality.

### Interview Self-Check

- Why can RAG retrieve the right document but still produce the wrong answer?
- How would you evaluate retrieval quality?
- When is fine-tuning better than RAG?

---

## 5. Tool Calling

### Concept

Tool calling lets the model invoke external functions with structured arguments. It connects language reasoning with real-world capabilities.

### Mechanism

The model receives tool definitions, selects a tool, generates arguments, and the application executes the tool. The tool result is then returned to the model for the next step or final answer.

Key design points:

- tool name and description,
- JSON schema,
- required vs optional fields,
- permission scope,
- timeout and retry policy,
- structured success and error responses.

### Trade-off

Tool calling improves factuality and actionability, but creates security and reliability risks. A bad tool schema can lead to wrong calls; overly broad permissions can cause real damage.

### Production Practice

- Use least privilege for every tool.
- Validate arguments before execution.
- Separate read-only tools from write tools.
- Require confirmation for irreversible actions.
- Log tool call inputs, outputs, latency, and error type.

### Interview Self-Check

- Why is "ask the model to output JSON" weaker than real tool calling?
- How do you design a safe write-action tool?
- How should a tool return errors to an agent?

---

## 6. Memory

### Concept

Memory gives an AI system access to information beyond the current prompt. It can be session memory, user profile memory, semantic memory, or task state.

### Mechanism

Common memory types:

- short-term conversation state,
- summarized session memory,
- long-term user preferences,
- vector memory for semantic recall,
- workflow state for agents.

### Trade-off

Memory improves personalization and continuity, but risks privacy leakage, stale assumptions, and context pollution. Long-term memory must be explicit, permissioned, and editable.

### Production Practice

- Separate conversational state from long-term user memory.
- Store provenance and timestamps.
- Let users inspect or delete persistent memory.
- Avoid injecting all memory blindly into every prompt.
- Add retrieval and ranking over memory items.

### Interview Self-Check

- Why is memory not the same as a long context window?
- What memory should not be persisted?
- How do you prevent stale memory from harming answers?

---

## 7. Agent Architecture

### Concept

An agent is a system that uses an LLM to plan, call tools, observe results, update state, and continue until a task is complete or escalated.

### Mechanism

```text
User goal
  -> plan
  -> select tool
  -> execute tool
  -> observe result
  -> update state
  -> continue or finish
```

Common patterns:

- ReAct: reasoning plus acting in a loop,
- planner-executor: split planning from execution,
- reflection: critique and revise intermediate results,
- multi-agent: coordinator delegates to specialized agents.

### Trade-off

Agents can solve open-ended workflows, but increase nondeterminism, cost, latency, debugging difficulty, and safety exposure. They should be used when task complexity justifies the control loop.

### Production Practice

- Define maximum steps, budgets, and stop conditions.
- Give tools narrow permissions.
- Use structured state rather than relying only on conversation text.
- Add human approval for high-risk operations.
- Record agent traces for debugging and audit.

### Interview Self-Check

- Why is an agent more like a workflow controller than a model?
- When would you avoid using an agent?
- How do you debug a failed agent run?

---

## 8. Hallucination

### Concept

Hallucination means the model produces content that is fluent but unsupported, false, or inconsistent with available evidence.

### Mechanism

Causes include:

- next-token prediction optimized for plausible continuation,
- missing or noisy context,
- weak retrieval grounding,
- ambiguous user intent,
- decoding randomness,
- pressure to answer when evidence is insufficient.

### Trade-off

Lower temperature and stronger grounding reduce hallucination but may reduce creativity or recall. RAG improves evidence access but does not guarantee faithful generation.

### Production Practice

- Add "answer only from provided evidence" instructions for factual tasks.
- Use citations and source span checks.
- Separate retrieval evaluation from answer evaluation.
- Use verification models or deterministic validators for high-risk outputs.
- Allow "I do not know" as a valid response.

### Interview Self-Check

- Why does hallucination happen even in strong models?
- What is the difference between retrieval failure and generation failure?
- How would you design a hallucination mitigation pipeline?

---

## 9. Senior Interview Q&A

### Q1: A RAG system retrieves the correct document but still answers incorrectly. How would you diagnose it?

**Answer**: Separate retrieval, context construction, and generation. First verify whether the relevant span is present in the injected context, not only in the top document. Then check chunk size, reranking, metadata filters, prompt instructions, citation requirements, and whether the model is using unsupported prior knowledge. The fix may be better chunking, stronger reranking, answer-grounding checks, or an explicit abstention rule.

### Q2: How do you decide between RAG, fine-tuning, and tool calling?

**Answer**: Use RAG when the issue is missing, private, or fast-changing knowledge. Use fine-tuning when the issue is repeated behavior, format, style, or domain task adaptation that examples can teach. Use tool calling when the system needs real-time state, deterministic computation, or side effects. In production these are often combined, but each should solve a distinct failure mode.

### Q3: What makes an agent fail online even if each tool works in isolation?

**Answer**: Agent failures often come from orchestration: ambiguous goals, poor stop conditions, hidden state, tool selection mistakes, context pollution, retries without new information, and missing budget limits. Debug with a trace that records plan, selected tool, arguments, observations, state updates, token/cost budget, and final decision.

### Q4: How would you design memory without creating privacy or quality problems?

**Answer**: Separate short-term session state from persistent user memory. Persist only explicit, useful, permissioned facts with provenance and timestamps. Rank memory before injection, let users inspect or delete it, and avoid copying sensitive transient data into long-term memory. Evaluate memory by task success and by negative cases such as stale or cross-user leakage.

---

## 10. Interview Self-Check

Use these questions for final review:

- Explain the full LLM lifecycle from pretraining to production use.
- Compare RAG, fine-tuning, and tool calling.
- Design a customer-support agent with safe tool permissions.
- Explain how KV cache improves inference.
- Explain why agents need stop conditions, budgets, and traceability.
- Diagnose an AI assistant that is accurate offline but fails online.
