# Claude Architect Certification Knowledge System

Language: English | [中文](../AI知识库/06-Claude架构师认证知识体系.md)

> This document summarizes Claude-oriented agent architecture concepts: Messages API, tool use, agentic loops, hooks, sessions, MCP integration, Claude Code workflows, prompt engineering, context management, and reliability. Most design patterns transfer to other LLM platforms.

---

## 0. Scope and Interview Goal

This document is useful for:

- Claude and Anthropic platform interviews,
- agent platform architecture,
- developer productivity tools,
- AI coding assistant workflows,
- LLM application architecture discussions.

The goal is to explain how a Claude-style system makes tool calls reliable, auditable, and controllable.

---

## 1. Claude API Basics

### Concept

Claude's Messages API is stateless. The client sends the current conversation state and configuration on each request.

### Mechanism

Important request fields:

- model,
- max tokens,
- system prompt,
- messages,
- tools,
- tool choice.

Important response control:

- `stop_reason = end_turn`: the model is done,
- `stop_reason = tool_use`: execute a tool and continue,
- `stop_reason = max_tokens`: output was truncated,
- `stop_reason = stop_sequence`: a custom stop condition fired.

### Trade-off

Stateless APIs are scalable and explicit, but the client must manage conversation history, context size, session persistence, and tool loop control.

### Production Practice

- Use `stop_reason` rather than parsing text to detect completion.
- Store messages and tool results needed for replay.
- Trim or summarize context deliberately.
- Set output token limits based on task type.

### Interview Self-Check

- Why is `stop_reason` important in agent loops?
- What does API statelessness imply for session management?
- How do you handle `max_tokens` safely?

---

## 2. Tool Use and JSON Schema

### Concept

Tool use provides structured, schema-constrained interaction between the model and external capabilities.

### Mechanism

Tool definitions specify:

- name,
- description,
- input schema,
- required fields,
- enum and nullable choices,
- tool choice behavior.

Tool use is stronger than asking the model to "return JSON" because schema and tool invocation are part of the protocol.

### Trade-off

Schema constraints improve syntax reliability but do not guarantee semantic correctness. Business validation remains necessary.

### Production Practice

- Use enums for controlled categories.
- Use nullable fields for unknown values.
- Validate semantic constraints after tool output.
- Retry with clear validation feedback only when retry can fix the issue.

### Interview Self-Check

- Why use tool use for structured extraction?
- What is the difference between syntax validation and semantic validation?
- When should `tool_choice` force a specific tool?

---

## 3. Agentic Loop and Multi-Agent Topology

### Concept

An agentic loop repeatedly calls the model, executes tools, appends results, and continues until the model ends the turn or the system escalates.

### Mechanism

```text
send request
  -> receive model response
  -> if tool_use: execute tool
  -> append tool result
  -> continue
  -> if end_turn: return final answer
```

Multi-agent systems often use a hub-spoke topology:

- coordinator decomposes the task,
- specialized agents solve subtasks,
- coordinator aggregates and validates results.

### Trade-off

Multi-agent designs improve specialization and parallelism, but create context fragmentation, error propagation, duplicated work, and higher cost.

### Production Practice

- Pass explicit context to sub-agents.
- Define each agent's responsibility and tool access.
- Keep coordinator responsible for validation.
- Limit recursion, steps, and budget.
- Record traces for each delegated task.

### Interview Self-Check

- Why use hub-spoke instead of every agent talking to every other agent?
- How do you avoid context loss when delegating?
- How do you handle a failed sub-agent?

---

## 4. Lifecycle Hooks

### Concept

Hooks are deterministic lifecycle controls around model or tool actions. They enforce policies that should not rely only on prompt obedience.

### Mechanism

Examples:

- pre-tool-use hook: block unsafe or unauthorized calls,
- post-tool-use hook: redact or transform results,
- session hook: initialize environment or load context,
- validation hook: check outputs before returning.

### Trade-off

Hooks improve control and auditability, but excessive hook logic can make behavior hard to understand and debug.

### Production Practice

- Use hooks for hard policies.
- Use prompts for soft behavior guidance.
- Keep hook decisions structured and logged.
- Test hooks independently from model prompts.

### Interview Self-Check

- When should you use a hook instead of a prompt?
- What policies must be deterministic?
- How do hooks help with security?

---

## 5. Session Management

### Concept

Session management controls continuity, branching, replay, and context persistence.

### Mechanism

Patterns:

- resume a named session to continue a task,
- fork a session to explore alternatives,
- start a new session to avoid context contamination,
- summarize or compact long sessions.

### Trade-off

Long sessions preserve context but can accumulate stale assumptions and irrelevant details. New sessions reduce contamination but may lose useful state.

### Production Practice

- Store session metadata and provenance.
- Fork when exploring independent alternatives.
- Start fresh when goals or constraints changed significantly.
- Keep critical state in structured files or durable records, not only chat history.

### Interview Self-Check

- When should you resume versus start a new session?
- Why can long context be harmful?
- How do you preserve provenance across sessions?

---

## 6. MCP and Tool Design

### Concept

Claude-style agent systems often use MCP to connect tools and resources in a standardized way.

### Mechanism

Key design choices:

- tool versus resource,
- clear tool descriptions,
- structured errors,
- `isError` semantics,
- permission and authorization boundary,
- resource injection versus exploratory tool calls.

### Trade-off

Resources reduce tool-call overhead but can expose too much context. Tools enable controlled actions but require schemas and error handling.

### Production Practice

- Make tool descriptions task-specific and unambiguous.
- Return structured errors with recoverability hints.
- Use resources for stable context such as schemas or documentation.
- Use tools for dynamic queries and actions.
- Apply least privilege.

### Interview Self-Check

- How do you decide between tool and resource?
- Why is tool description engineering important?
- How should a tool communicate a recoverable error?

---

## 7. Claude Code Workflows

### Concept

Claude Code-style workflows combine project instructions, slash commands, skills, planning, execution, and CI/CD automation.

### Mechanism

Important concepts:

- project instructions such as `CLAUDE.md`,
- reusable slash commands,
- agent skills,
- planning mode for complex work,
- non-interactive CLI mode for automation,
- structured output for CI/CD integration.

### Trade-off

Automation improves productivity but can create unsafe changes if permissions, review, and test gates are weak.

### Production Practice

- Keep project rules concise and scoped.
- Use planning for large or ambiguous changes.
- Use non-interactive commands for CI tasks.
- Require tests and review for generated changes.
- Keep secrets out of prompts and logs.

### Interview Self-Check

- Why are layered project instructions useful?
- When should an agent plan before editing?
- How would you run an AI code review in CI?

---

## 8. Prompt Engineering and Structured Output

### Concept

Claude-style prompt engineering emphasizes explicit criteria, examples, decomposition, and validation loops.

### Mechanism

Patterns:

- few-shot prompting,
- explicit rubrics,
- prompt chaining,
- dynamic task decomposition,
- validation-retry loops,
- batch processing for asynchronous high-volume work.

### Trade-off

Prompt chaining improves control but increases latency and cost. Batch APIs reduce cost for large offline jobs but are not appropriate for interactive workflows.

### Production Practice

- Use explicit standards instead of vague instructions.
- Use tools for structured output when correctness matters.
- Retry only when feedback is actionable.
- Use batch processing for non-urgent large workloads.

### Interview Self-Check

- Why do explicit criteria outperform vague prompts?
- When is prompt chaining useful?
- Why is synchronous API still needed if batch is cheaper?

---

## 9. Context Management and Reliability

### Concept

Context management is about placing the right information where the model can use it, without overwhelming or misleading it.

### Mechanism

Challenges:

- long context cost,
- lost-in-the-middle behavior,
- stale context,
- conflicting evidence,
- missing provenance,
- context pollution from unrelated tasks.

### Trade-off

More context can improve recall but reduce focus. Summaries reduce tokens but may lose details. Delegation protects context but adds coordination overhead.

### Production Practice

- Put critical instructions and data near attention-friendly positions.
- Preserve source attribution.
- Resolve conflicting evidence explicitly.
- Use scratchpad files or structured state for long investigations.
- Escalate when evidence is insufficient or risk is high.

### Interview Self-Check

- What is lost in the middle?
- How do you preserve source attribution?
- When should an agent escalate to a human?

---

## 10. Senior Interview Q&A

### Q1: How do you design an auditable Claude-style agent workflow?

**Answer**: Record every model turn, visible tools, selected tool, arguments, tool result, stop reason, hook decision, validation result, cost, latency, and final output under a shared trace ID. Keep redaction and retention policies explicit. For sensitive operations, add approval records and replayable state so incidents can be reconstructed.

### Q2: When should a policy be implemented as a hook instead of a prompt instruction?

**Answer**: Use hooks for deterministic, enforceable policies: blocking destructive tools, redacting secrets, validating schema, checking permissions, or requiring approval. Use prompts for soft preferences such as tone, prioritization, and explanation style. Anything that must hold under prompt injection should not rely only on model obedience.

### Q3: A sub-agent returns partial or conflicting results. What should the coordinator do?

**Answer**: The coordinator should require structured status, evidence, confidence, and error categories from sub-agents. It can retry with additional context, ask a different specialist, merge non-conflicting parts, or escalate to a human. It should not silently ignore errors or treat all sub-agent output as equally reliable.

### Q4: How do you decide that an agent solution should not be built yet?

**Answer**: Do not build it if the success metric is unclear, tools are unreliable, side effects cannot be audited, permissions are broad, rollback is missing, or the team cannot operate incidents. Agent autonomy should come after stable tools, evaluation, tracing, and governance.

---

## 11. Interview Self-Check

- Design a Claude-style customer support agent with safe refund tools.
- Explain how `stop_reason` drives an agent loop.
- Compare hooks and prompt instructions.
- Explain hub-spoke multi-agent topology.
- Explain when to use Tool Use for structured extraction.
- Design a context strategy for a long codebase investigation.
- Explain how to build an auditable agent workflow.
