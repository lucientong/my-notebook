# MCP and Tool Integration

Language: English | [中文](../AI知识库/02-MCP与工具集成.md)

> This document focuses on how AI applications safely and consistently connect to external tools, resources, prompts, and context through MCP-style integration. For general agent concepts, read [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md). For production governance, read [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md).

---

## 0. What This Document Solves

After reading this document, you should be able to explain:

- when to use a tool and when to use a resource,
- how an MCP client and server communicate,
- how to design tool schemas, resource URIs, and prompt templates,
- how to control permissions, performance, debugging, and compatibility.

---

## 1. MCP Protocol Fundamentals

### Concept

MCP, or Model Context Protocol, is a standardized protocol for connecting AI applications to external tools, data, prompts, and contextual resources.

It provides a common integration layer between an AI client and one or more MCP servers:

```text
AI application
  -> MCP client
  -> MCP server
       -> tools
       -> resources
       -> prompts
```

### Mechanism

MCP commonly uses JSON-RPC style request and response messages. A client discovers server capabilities, lists tools or resources, and invokes operations with structured parameters.

Core objects:

| Object | Purpose | Example |
|---|---|---|
| Tool | Execute an action or computation | search, query database, create ticket |
| Resource | Provide readable context | document, schema, file, config |
| Prompt | Reusable instruction template | code review prompt, triage prompt |

### Trade-off

Standardization improves portability and discoverability, but you still need application-level security, validation, observability, and compatibility planning. A protocol cannot make unsafe tools safe by itself.

### Production Practice

- Separate tool definitions from business authorization.
- Version tool schemas and resource contracts.
- Use structured errors instead of free-form failure text.
- Log discovery, calls, latency, and failures.
- Treat MCP servers as production services with SLOs.

### Interview Self-Check

- What problem does MCP solve compared with ad hoc tool integration?
- Why is capability discovery useful?
- What risks remain even if the protocol is standardized?

---

## 2. Client and Server Architecture

### Concept

An MCP server exposes capabilities; an MCP client consumes them on behalf of an AI application or agent.

### Mechanism

Typical lifecycle:

```text
Initialize connection
  -> exchange capabilities
  -> list tools / resources / prompts
  -> select capability
  -> call tool or read resource
  -> return structured result
```

The server owns tool implementation and resource access. The client owns orchestration, user context, and model interaction.

### Trade-off

Putting capabilities behind servers makes integration modular, but introduces network failures, version mismatch, authentication complexity, and latency.

### Production Practice

- Keep server capabilities cohesive and domain-specific.
- Make operations idempotent when possible.
- Add timeouts and cancellation support.
- Return machine-readable error codes.
- Test client-server compatibility before rollout.

### Interview Self-Check

- What belongs in the MCP server versus the AI application?
- How would you handle server version upgrades?
- Why should tool execution not be hidden inside prompts?

---

## 3. Tool Design

### Concept

A tool is a structured operation that the model may ask the application to execute. Good tool design is the difference between reliable automation and fragile text parsing.

### Mechanism

A tool definition should include:

- name,
- clear description,
- input schema,
- required fields,
- examples and constraints,
- success result shape,
- error result shape,
- permission scope.

### Trade-off

Fine-grained tools are safer and easier to reason about, but too many small tools can confuse the model. Coarse-grained tools are convenient but risk over-permissioning and ambiguous behavior.

### Production Practice

- Prefer explicit verbs: `search_documents`, `get_order_status`, `create_refund_request`.
- Use enums for bounded choices.
- Use nullable fields instead of asking the model to invent missing values.
- Validate all inputs server-side.
- Require human approval for irreversible side effects.

### Interview Self-Check

- How would you design a safe refund tool?
- Why does tool description quality affect model behavior?
- When should you split one tool into multiple tools?

---

## 4. Resource Management

### Concept

A resource is readable context exposed by a server, such as a document, file, database schema, configuration, or knowledge artifact.

### Mechanism

Resources are usually identified by URIs and metadata:

```text
resource URI -> metadata -> content -> model context
```

Unlike tools, resources are not primarily actions. They provide context the model can use.

### Trade-off

Resources reduce unnecessary tool calls and improve context availability, but stale or overbroad resources can leak data or pollute the prompt.

### Production Practice

- Use stable URI naming.
- Include version and provenance metadata.
- Apply permission filtering before content reaches the model.
- Keep resources concise and task-relevant.
- Cache resources carefully with invalidation rules.

### Interview Self-Check

- When should a database schema be a resource rather than a tool?
- How do you prevent resource-based data leakage?
- Why does provenance matter for resources?

---

## 5. Prompt Templates

### Concept

Prompt templates are reusable prompts exposed by an MCP server or application to standardize repeated workflows.

### Mechanism

A template defines arguments and produces a complete prompt:

```text
template name + arguments -> rendered prompt -> model request
```

### Trade-off

Templates improve consistency and reuse, but excessive templating can hide business logic and make prompts hard to audit.

### Production Practice

- Version templates.
- Keep template arguments explicit.
- Add examples only where ambiguity exists.
- Test rendered prompts, not only template source.
- Avoid mixing user content into system-level policy text.

### Interview Self-Check

- What should be parameterized in a prompt template?
- How would you test a prompt template?
- Why can prompt templates become a maintenance risk?

---

## 6. Transport Layer

### Concept

Transport defines how MCP messages move between client and server. Common styles include local stdio, HTTP streaming, WebSocket-like channels, or service-to-service RPC.

### Mechanism

Transport must preserve request IDs, responses, errors, cancellation, and streaming semantics when needed.

### Trade-off

Local transport is simple and low-latency for developer tools. Network transport is better for shared services but requires authentication, retries, rate limits, and observability.

### Production Practice

- Use local stdio for single-user local tools.
- Use authenticated network transport for shared enterprise services.
- Support timeouts and cancellation.
- Emit request IDs for tracing.
- Monitor transport-level error rates separately from tool-level failures.

### Interview Self-Check

- When would you choose local versus remote MCP servers?
- What makes streaming useful for AI tools?
- How do transport errors differ from business errors?

---

## 7. Sampling and Model-Mediated Operations

### Concept

Sampling allows a server-side workflow to request model assistance through the client, while keeping model access controlled by the application.

### Mechanism

Instead of the server directly calling a model, it asks the client to sample from a model under the client's policy and user context.

### Trade-off

This preserves model governance and user control, but introduces callback complexity and tighter coupling between client policy and server workflows.

### Production Practice

- Use sampling only when server-side logic genuinely needs model reasoning.
- Keep user approval and model policy on the client side.
- Log why sampling was requested.
- Bound token budget and number of sampling attempts.

### Interview Self-Check

- Why should an MCP server not freely call arbitrary models?
- What policy should govern sampling?
- How would you prevent recursive or runaway sampling?

---

## 8. Security and Permission Control

### Concept

Tool integration is a security boundary. The model should never directly inherit broad user or service privileges.

### Mechanism

Risks include:

- prompt injection,
- confused deputy attacks,
- overbroad tool permissions,
- data exfiltration through resources,
- unauthorized write actions,
- tool result spoofing.

### Trade-off

Strict controls improve safety but may reduce automation. The right design separates low-risk read actions, high-risk write actions, and human-approved operations.

### Production Practice

- Use least privilege per tool and per user.
- Apply authorization before execution, not only in prompts.
- Sanitize tool outputs before returning them to the model.
- Add confirmation for destructive or external actions.
- Keep audit logs for who requested what, under which context.

### Interview Self-Check

- How can prompt injection affect tool calling?
- What is the confused deputy problem in AI systems?
- How do you design a permission model for tools?

---

## 9. Performance Optimization

### Concept

MCP integration can add latency through capability discovery, network calls, retrieval, serialization, and tool execution.

### Mechanism

Latency sources:

- connection setup,
- listing tools or resources,
- remote service calls,
- slow downstream APIs,
- large resource payloads,
- repeated model-tool loops.

### Trade-off

Caching and batching reduce latency but can create stale results. Parallel tool calls improve throughput but can increase load and complicate error handling.

### Production Practice

- Cache static capability lists.
- Cache stable resources with versioning.
- Batch independent reads.
- Add deadlines and circuit breakers.
- Track p50, p95, p99 latency per tool.

### Interview Self-Check

- How would you reduce latency in an agent that calls many tools?
- What should be cached and what should not?
- How do you prevent a slow tool from blocking the whole agent?

---

## 10. Debugging and Troubleshooting

### Concept

MCP debugging requires tracing both model decisions and tool/server behavior.

### Mechanism

For a failed run, inspect:

1. user request,
2. tool definitions visible to the model,
3. selected tool and arguments,
4. authorization decision,
5. server execution logs,
6. returned result or error,
7. model's final response.

### Trade-off

Detailed logs improve debugging but may contain sensitive data. Observability must be paired with redaction and access control.

### Production Practice

- Use correlation IDs across model, client, server, and downstream systems.
- Redact secrets and personal data.
- Store structured traces for agent runs.
- Classify errors as model selection, schema, permission, transport, downstream, or business failure.

### Interview Self-Check

- How would you debug a model choosing the wrong tool?
- How would you debug a correct tool call that produced a bad answer?
- What logs are necessary for auditability?

---

## 11. Senior Interview Q&A

### Q1: How would you design an MCP server for 50 internal tools without making the model confused?

**Answer**: Group tools by domain, keep names action-oriented and mutually exclusive, and expose only the tools allowed for the current user and task. Use capability discovery, schema versioning, examples for ambiguous arguments, and regression tests with realistic user requests. At scale, tool governance matters as much as protocol implementation.

### Q2: What is the minimum trace needed to debug an unsafe or wrong tool call?

**Answer**: Capture the user request, visible tool list, rendered tool descriptions, selected tool, generated arguments, authorization decision, server execution result, downstream error if any, final model response, and correlation ID. Redact sensitive values but keep enough structure to reproduce the decision path.

### Q3: How do you handle tool timeouts and retries safely?

**Answer**: Retry only transient failures such as network errors, rate limits, or upstream 5xx responses. Do not retry validation errors, authorization failures, or irreversible side effects without idempotency keys. Use bounded retries with exponential backoff, jitter, deadlines, and clear error categories returned to the agent.

### Q4: When should a schema change be treated as a breaking change?

**Answer**: It is breaking when a required field is added, a field's meaning changes, enum values are removed, response structure changes, or error semantics change. Prefer additive changes, capability negotiation, contract tests, replay tests, and per-client rollout metrics.

---

## 12. Interview Self-Check

- Explain MCP in one minute.
- Compare tools, resources, and prompt templates.
- Design an MCP server for an internal knowledge base.
- Design a permission model for a tool that can read and update tickets.
- Explain how you would version tool schemas without breaking clients.
- Explain how to debug a failed agent run involving multiple tools.
