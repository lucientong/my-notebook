# AI Engineering Practices

Language: English | [中文](../AI知识库/03-AI工程化实践.md)

> This document focuses on turning LLM demos into reliable production systems: architecture, prompt management, evaluation, cost, latency, observability, safety, CI/CD, and framework selection.

---

## 0. What This Document Solves

An AI demo can work once. A production AI system must work repeatedly, safely, observably, and economically.

You should be able to answer:

- What must be designed beyond the model itself?
- Why do AI systems need routing, caching, rate limiting, fallback, evaluation, and tracing?
- How do you debug whether a bad answer came from prompt, retrieval, tool, model, or workflow?

---

## 1. System View

### Concept

AI engineering is about controlling an uncertain model inside a reliable software system.

### Mechanism

Typical architecture:

```text
User request
  -> API / business entry
  -> model router
  -> context builder
       -> prompt
       -> RAG
       -> user context
       -> memory
  -> model call
  -> tool call / data query
  -> output parsing and safety filtering
  -> logs, metrics, evaluation, feedback
```

### Trade-off

More layers improve control and debuggability, but they also add latency, cost, and operational complexity.

### Production Practice

For each request, capture:

- model and prompt version,
- retrieved documents and scores,
- tool calls and results,
- token usage and latency,
- safety decisions,
- final answer and user feedback.

### Interview Self-Check

- How would you debug an AI assistant that suddenly got worse after a prompt change?
- Why is observability harder for AI systems than deterministic services?
- What would you log for a RAG answer?

---

## 2. LLM Application Architecture

### Concept

A production LLM application is usually layered: gateway, routing, policy, context, model service, tools/resources, and observability.

### Mechanism

Key components:

| Layer | Responsibility |
|---|---|
| API gateway | authentication, quotas, request shaping |
| Model router | choose model by task, cost, latency, risk |
| Policy layer | rate limit, cache, fallback, safety rules |
| Context layer | prompt, RAG, memory, user profile |
| LLM service | model calls, streaming, parsing |
| Tool/resource layer | external data and actions |
| Observability | logs, metrics, traces, evaluation |

### Trade-off

Routing can reduce cost and latency, but routing mistakes can hurt quality. Caching can save money but risks stale or user-specific leakage. Fallback improves availability but may reduce answer quality.

### Production Practice

- Use smaller models for simple deterministic tasks.
- Route high-risk or complex tasks to stronger models.
- Cache only safe and stable outputs.
- Separate user-specific cache from global cache.
- Define fallback responses per scenario.

### Interview Self-Check

- How would you design a model router?
- What outputs are unsafe to cache globally?
- How do you handle provider outages?

---

## 3. Prompt Management

### Concept

Prompts are production artifacts. They need versioning, testing, review, rollout, and rollback.

### Mechanism

Prompt management includes:

- templates,
- variables,
- system instructions,
- few-shot examples,
- output schema,
- version metadata,
- test cases,
- release history.

### Trade-off

Hard-coded prompts are easy initially but hard to audit. Dynamic prompt systems are flexible but can become opaque and difficult to reproduce.

### Production Practice

- Store prompt versions.
- Record the exact rendered prompt per request when privacy allows.
- Maintain golden test sets.
- Run offline evaluation before rollout.
- Use staged rollout and rollback.

### Interview Self-Check

- Why should prompts be versioned?
- How do you evaluate a prompt change?
- How can prompt injection bypass naive prompt instructions?

---

## 4. Evaluation System

### Concept

AI evaluation measures whether a system is improving. It must evaluate quality, retrieval, safety, latency, cost, and user impact.

### Mechanism

Evaluation layers:

| Layer | Example Metrics |
|---|---|
| Retrieval | recall@k, MRR, nDCG, citation coverage |
| Generation | correctness, helpfulness, faithfulness, format validity |
| Safety | policy violation rate, injection success rate |
| System | latency, cost, error rate, fallback rate |
| Business | task success, conversion, retention, CSAT |

### Trade-off

LLM-as-judge scales evaluation but may be biased or inconsistent. Human review is higher quality but slower and more expensive. Offline metrics are controlled but may not predict user behavior.

### Production Practice

- Keep separate test sets for retrieval and answer quality.
- Use human-labeled samples for calibration.
- Combine rule-based checks, model-based judging, and human review.
- Track regression by prompt version, model version, and knowledge base version.
- Use online A/B testing for product impact.

### Interview Self-Check

- How do you evaluate a RAG system?
- When is LLM-as-judge acceptable?
- Why can offline accuracy improve while online satisfaction drops?

---

## 5. Cost Optimization

### Concept

LLM cost is driven by input tokens, output tokens, model choice, tool calls, retrieval, retries, and traffic pattern.

### Mechanism

Cost levers:

- prompt compression,
- context pruning,
- model routing,
- caching,
- batching,
- response length limits,
- retrieval top-k control,
- fine-tuned smaller models for repeated tasks.

### Trade-off

Aggressive cost reduction can harm quality. Smaller models, shorter context, lower top-k, and caching must be validated against quality and safety.

### Production Practice

- Track cost per request, user, feature, and business outcome.
- Add token budgets per feature.
- Route simple tasks to cheaper models.
- Summarize long histories rather than replaying all turns.
- Use alerts for cost spikes.

### Interview Self-Check

- How would you reduce LLM cost by 50% without hurting quality too much?
- What is the risk of caching model outputs?
- How do you measure cost-effectiveness instead of cost alone?

---

## 6. Latency Optimization

### Concept

Latency comes from routing, retrieval, model inference, tool calls, output parsing, safety checks, and network overhead.

### Mechanism

Important metrics:

- time to first token,
- total response time,
- retrieval latency,
- tool latency,
- p95 and p99 latency,
- timeout and retry rates.

### Trade-off

Parallelism and streaming improve perceived latency but complicate control flow. Smaller models improve speed but may reduce quality. Caching improves speed but may produce stale results.

### Production Practice

- Stream tokens when useful.
- Run independent retrieval/tool calls in parallel.
- Set deadlines for each stage.
- Use fallback for slow dependencies.
- Track latency by model and tool.

### Interview Self-Check

- How do you optimize time to first token?
- How do you prevent one slow tool from blocking the whole response?
- Why is p99 latency important for AI systems?

---

## 7. Observability

### Concept

Observability lets you explain why the system produced an answer, how much it cost, and where it failed.

### Mechanism

Trace dimensions:

```text
request -> prompt -> retrieval -> tool calls -> model call -> validation -> response
```

### Trade-off

Deep traces are powerful but can contain sensitive data. You need redaction, access control, retention policy, and sampling.

### Production Practice

- Correlate logs across services.
- Store prompt, context, model, token count, and tool calls with redaction.
- Classify errors by source.
- Add dashboards for quality, cost, latency, and safety.
- Use user feedback to create regression tests.

### Interview Self-Check

- What traces would you need to debug hallucination?
- How do you log prompts without leaking sensitive data?
- What dashboard would you build for an AI platform?

---

## 8. Safety and Security

### Concept

AI safety in engineering includes prompt injection defense, data leakage prevention, tool permission control, output filtering, abuse monitoring, and human escalation.

### Mechanism

Common risks:

- user prompt overrides system policy,
- retrieved documents contain malicious instructions,
- model calls over-permissioned tools,
- output leaks secrets or personal data,
- generated content violates policy.

### Trade-off

More safety checks reduce risk but add latency, cost, and false positives. High-risk workflows need stronger controls than low-risk creative tasks.

### Production Practice

- Treat retrieved content as untrusted data.
- Keep policy outside user-controllable text.
- Apply authorization before tool execution.
- Use guardrails for input and output.
- Add human approval for irreversible actions.

### Interview Self-Check

- How would you defend against prompt injection in RAG?
- Why is tool permission more important than prompt wording?
- How do you design escalation for risky cases?

---

## 9. CI/CD and Automation

### Concept

AI systems need release pipelines for prompts, models, retrieval indexes, tools, and evaluation sets.

### Mechanism

Pipeline:

```text
change proposal
  -> unit checks
  -> offline evaluation
  -> safety tests
  -> shadow traffic
  -> canary rollout
  -> monitoring
  -> full rollout or rollback
```

### Trade-off

Strict gates slow iteration but reduce regressions. Fast prompt iteration is useful, but production prompts need the same discipline as code.

### Production Practice

- Use golden datasets for every major workflow.
- Run injection and safety regression tests.
- Canary by traffic segment.
- Keep rollback for prompt, model, and index versions.
- Review high-impact prompt or tool changes.

### Interview Self-Check

- How would you release a new prompt safely?
- What should be in an AI regression test suite?
- How do you roll back a retrieval index change?

---

## 10. Framework Selection

### Concept

LLM frameworks help with orchestration, memory, tool calling, tracing, and evaluation, but they should not hide core system behavior.

### Mechanism

Evaluate frameworks by:

- model/provider support,
- tool abstraction,
- tracing and observability,
- retrieval integration,
- streaming support,
- deployment maturity,
- extensibility,
- operational transparency.

### Trade-off

Frameworks accelerate prototypes but can create lock-in, opaque control flow, and difficult debugging. Low-level SDKs provide control but require more engineering work.

### Production Practice

- Prototype with frameworks if useful.
- Keep business logic outside framework-specific chains.
- Own your evaluation and tracing layer.
- Avoid irreversible coupling to one model provider.

### Interview Self-Check

- When would you use LangChain or a similar framework?
- What logic should not be hidden in a framework chain?
- How do you avoid provider lock-in?

---

## 11. Senior Interview Q&A

### Q1: Online hallucination incidents increased after a prompt release. What is your rollback and investigation plan?

**Answer**: Roll back the prompt or route traffic to the previous stable version first if user impact is high. Then compare traces by prompt version: retrieved context, citations, model output, refusal behavior, safety decisions, and user feedback. Convert representative bad cases into regression tests before trying a new fix. The point is to make hallucination governance part of release management, not an ad hoc prompt tweak.

### Q2: How would you define SLOs for an LLM application beyond HTTP success rate?

**Answer**: Use system SLOs, quality SLOs, and cost SLOs together. System SLOs include availability, p95/p99 latency, timeout rate, and provider error rate. Quality SLOs include task success, format validity, hallucination rate, citation coverage, refusal quality, and safety violation rate. Cost SLOs include token cost per request, budget burn rate, and cost per successful task.

### Q3: A model router reduced cost but user satisfaction dropped. What would you check?

**Answer**: Check routing thresholds, task complexity labels, fallback behavior, and whether weaker models are handling high-risk or high-context requests. Segment metrics by route, user type, language, prompt version, retrieval quality, and tool usage. Cost routing should be evaluated by cost per successful task, not only average token spend.

### Q4: How do you build an evaluation system that does not overfit to a golden set?

**Answer**: Keep separate sets for development, release gating, and hidden holdout evaluation. Add fresh production bad cases continuously, stratify by scenario and risk, calibrate LLM-as-judge against human labels, and monitor online metrics after release. A golden set should catch regressions, not become the only definition of quality.

---

## 12. Interview Self-Check

- Design a production RAG system for internal documentation.
- Explain how you would evaluate and roll out a prompt change.
- Diagnose high cost in an LLM application.
- Diagnose high latency in an agent workflow.
- Design observability for model, retrieval, and tool calls.
- Explain how you would handle hallucination, prompt injection, and unsafe tool calls.
