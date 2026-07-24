# AI Safety and Alignment

Language: English | [中文](../AI知识库/08-AI安全与对齐.md)

> This document covers AI alignment, RLHF, Constitutional AI, red teaming, guardrails, hallucination mitigation, privacy, model security, governance, and enterprise safety practice.

---

## 0. What This Document Solves

AI safety is not only a research topic. In production systems it becomes concrete engineering: prevent harmful outputs, unauthorized actions, privacy leakage, prompt injection, model misuse, and untraceable failures.

---

## 1. AI Alignment Overview

### Concept

AI alignment means making AI systems behave according to human intent, values, and constraints, not just literal instructions or proxy rewards.

### Mechanism

Alignment has several layers:

| Layer | Question |
|---|---|
| Capability alignment | Can the model understand and execute the instruction? |
| Intent alignment | Does it infer what the user actually wants? |
| Value alignment | Does behavior remain consistent with human and social values? |

Inner versus outer alignment:

- **Outer alignment**: the training objective matches human intent.
- **Inner alignment**: the learned model objective matches the training objective.

### Trade-off

Stronger safety controls can reduce harmful behavior but may add alignment tax: lower utility, higher latency, false refusals, and additional review cost.

### Production Practice

- Define acceptable and unacceptable behavior per product domain.
- Separate policy from user-controllable prompts.
- Evaluate safety with adversarial tests.
- Add escalation paths for uncertain or high-risk cases.

### Interview Self-Check

- What is the difference between outer and inner alignment?
- What is reward hacking?
- What is alignment tax?

---

## 2. RLHF and Preference Optimization

### Concept

RLHF uses human feedback to align model outputs with human preferences. DPO and related methods simplify preference optimization.

### Mechanism

RLHF pipeline:

```text
SFT model
  -> generate candidate responses
  -> humans rank responses
  -> train reward model
  -> optimize model with PPO and KL penalty
```

DPO directly optimizes the model to prefer chosen responses over rejected responses without training a separate reward model and running RL in the same way.

### Trade-off

RLHF is powerful but complex and can suffer from reward hacking or unstable optimization. DPO is simpler but depends heavily on preference data quality.

### Production Practice

- Keep preference data diverse and audited.
- Evaluate helpfulness, truthfulness, harmlessness, and refusal rate separately.
- Monitor over-refusal and sycophancy.
- Use human review for high-risk evaluation.

### Interview Self-Check

- What are the three stages of RLHF?
- Why is KL penalty used in PPO-style RLHF?
- How does DPO simplify preference learning?

---

## 3. Constitutional AI

### Concept

Constitutional AI uses a set of written principles to guide model critique, revision, and preference learning.

### Mechanism

Typical process:

```text
model generates answer
  -> model critiques answer against principles
  -> model revises answer
  -> preferences or training data are built from these revisions
```

### Trade-off

Constitutional AI scales oversight and makes principles explicit, but principles can be incomplete, ambiguous, or culturally dependent.

### Production Practice

- Write domain-specific principles.
- Test principle conflicts.
- Keep human review for policy-sensitive domains.
- Version and audit safety principles.

### Interview Self-Check

- How does Constitutional AI differ from RLHF?
- Why are explicit principles useful?
- What can go wrong with a constitution?

---

## 4. Red Teaming

### Concept

Red teaming intentionally attacks an AI system to discover safety failures before real users do.

### Mechanism

Attack categories:

- prompt injection,
- jailbreaks,
- data exfiltration,
- harmful content elicitation,
- tool abuse,
- privacy attacks,
- policy bypass,
- multi-turn manipulation.

### Trade-off

Manual red teaming finds nuanced failures but is slow. Automated red teaming scales coverage but may miss creative attacks or generate noisy findings.

### Production Practice

- Maintain adversarial test suites.
- Test both model-only and full system behavior.
- Include RAG and tool-call attack scenarios.
- Track attack success rate by release.
- Feed failures back into guardrails and evaluation.

### Interview Self-Check

- What is prompt injection?
- How do jailbreaks differ from prompt injection?
- How would you red-team a RAG assistant with tools?

---

## 5. Guardrails

### Concept

Guardrails are controls around model input, tool use, retrieval, and output to keep the system within policy and safety boundaries.

### Mechanism

Guardrail layers:

```text
input filter
  -> policy classifier
  -> retrieval permission filter
  -> tool authorization
  -> output validator
  -> human escalation
```

### Trade-off

Guardrails reduce risk but can add false positives, latency, cost, and user friction. High-risk domains need stricter controls.

### Production Practice

- Combine rule-based, model-based, and human controls.
- Keep authorization outside the prompt.
- Apply permission filtering before RAG context injection.
- Validate structured outputs.
- Log safety decisions for audit.

### Interview Self-Check

- What belongs in input guardrails versus output guardrails?
- Why are prompts not enough for safety?
- How do you tune false positives and false negatives?

---

## 6. Hallucination

### Concept

Hallucination is fluent but unsupported or incorrect output.

### Mechanism

Causes:

- missing knowledge,
- weak retrieval,
- misleading context,
- model uncertainty,
- decoding randomness,
- pressure to answer,
- lack of verification.

### Trade-off

Strict grounding reduces hallucination but can make the system less helpful for open-ended reasoning. Verification adds cost and latency.

### Production Practice

- Use RAG with citations for factual tasks.
- Check answer support against sources.
- Allow abstention.
- Use deterministic validators for numbers, dates, IDs, and code.
- Evaluate hallucination by domain and risk level.

### Interview Self-Check

- Why does RAG not fully solve hallucination?
- How do you detect unsupported claims?
- When should the system refuse or abstain?

---

## 7. Privacy and Data Security

### Concept

AI systems can leak private data through training data memorization, prompt logs, retrieved context, tool outputs, or generated responses.

### Mechanism

Risks:

- training data extraction,
- membership inference,
- prompt logging leakage,
- cross-user memory contamination,
- overbroad retrieval permissions,
- sensitive tool result exposure.

### Trade-off

More data improves personalization and quality but increases privacy risk. Differential privacy and redaction reduce risk but may reduce utility.

### Production Practice

- Minimize sensitive data sent to models.
- Redact secrets and personal data in logs.
- Enforce per-user permission filters.
- Separate tenant data.
- Provide deletion and retention controls.
- Use differential privacy where appropriate.

### Interview Self-Check

- How can an AI application leak private data?
- Why is RAG permission filtering critical?
- What is differential privacy trying to guarantee?

---

## 8. Model Security

### Concept

Model security protects models from malicious manipulation, extraction, evasion, and misuse.

### Mechanism

Threats:

- backdoor attacks,
- adversarial examples,
- model extraction,
- data poisoning,
- watermark removal,
- unsafe fine-tuning,
- supply-chain compromise.

### Trade-off

Security checks add friction and may limit openness. Open models provide flexibility but require stronger deployment controls.

### Production Practice

- Verify model sources and hashes.
- Scan datasets for poisoning signals.
- Test adversarial robustness.
- Restrict fine-tuning data and adapters.
- Monitor abnormal query patterns.
- Keep model and dataset provenance.

### Interview Self-Check

- What is a model backdoor?
- How can data poisoning affect model behavior?
- How would you protect a public model endpoint from extraction?

---

## 9. AI Ethics and Governance

### Concept

AI governance defines accountability, risk classification, fairness, explainability, documentation, and compliance for AI systems.

### Mechanism

Governance areas:

- bias detection,
- fairness metrics,
- explainability,
- risk assessment,
- documentation,
- auditability,
- human oversight,
- regulatory compliance such as EU AI Act style risk categories.

### Trade-off

Governance can slow product development but reduces legal, reputational, and safety risk. The right depth depends on domain risk.

### Production Practice

- Classify use cases by risk.
- Document model, data, evaluation, and limitations.
- Monitor fairness metrics across groups.
- Keep decision logs for high-impact systems.
- Define ownership for incidents.

### Interview Self-Check

- What fairness metrics might conflict with each other?
- Why is explainability domain-dependent?
- How would you govern a high-risk AI feature?

---

## 10. Enterprise Safety Practice

### Concept

Enterprise AI safety combines architecture, policy, evaluation, monitoring, incident response, and human governance.

### Mechanism

Safety program:

```text
risk classification
  -> policy definition
  -> secure architecture
  -> red-team evaluation
  -> rollout gate
  -> monitoring
  -> incident response
  -> continuous improvement
```

### Trade-off

Centralized safety platforms create consistency, while product teams need flexibility. Strong governance should enable safe shipping, not block all iteration.

### Production Practice

- Build shared safety classifiers and evaluation datasets.
- Use release gates for high-risk features.
- Track safety incidents and near misses.
- Define rollback and kill switches.
- Run periodic red-team exercises.

### Interview Self-Check

- How would you design an enterprise AI safety review?
- What should trigger a kill switch?
- How do you balance innovation and safety?

---

## 11. Senior Interview Q&A

### Q1: How would you evaluate whether guardrails are actually working?

**Answer**: Measure true positive rate, false positive rate, attack success rate, policy coverage, latency overhead, user friction, and consistency across similar inputs. Use adversarial datasets, automated red teaming, human review, production sampling, and release-to-release trend tracking. Guardrails should be evaluated as a system, not only as a classifier.

### Q2: Prompt injection bypasses a RAG assistant and causes a risky tool call. What architectural fixes matter most?

**Answer**: Treat retrieved content as untrusted data, keep policy outside retrieved text, apply tool authorization before execution, restrict tools by user and task, require confirmation for side effects, and log the full decision path. Prompt wording helps, but permission boundaries and deterministic controls are the real safety layer.

### Q3: How do you handle a production incident involving unsafe model output?

**Answer**: Contain first: disable the risky feature, roll back prompt/model/index versions, or route to a safer fallback. Preserve traces, identify affected users, classify the failure, and communicate if required by policy or law. Then add the incident cases to safety regression tests, update guardrails, and define a measurable prevention target.

### Q4: What privacy risks are unique to AI memory and RAG systems?

**Answer**: AI memory can persist sensitive or stale user facts and inject them into unrelated contexts. RAG can leak documents through missing permission filters, shared caches, prompt logs, or cross-tenant index mistakes. Mitigation requires data minimization, per-user authorization, tenant isolation, redaction, retention limits, deletion workflows, and audit logs.

---

## 12. Interview Self-Check

- Explain AI alignment to a backend engineer.
- Compare RLHF, DPO, and Constitutional AI.
- Design guardrails for an AI customer-support agent.
- Design a red-team plan for a RAG system.
- Explain how prompt injection can lead to tool abuse.
- Explain privacy risks in AI memory and RAG.
- Design an incident response process for unsafe model output.
