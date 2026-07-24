# AI Knowledge Base

Language: English | [中文](../AI知识库/README.md)

> This directory is the English companion to `cs/AI知识库`. It is designed for global-company technical interviews, where you need to explain AI systems clearly in English: concepts, mechanisms, trade-offs, production practice, and interview-style self-checks.

---

## Purpose

This knowledge base helps you move from "I can call an LLM API" to "I can design, operate, and defend an AI system in production." It keeps the same numbering as the Chinese version so you can cross-read both languages.

The main interview goal is not memorizing terms. The goal is to answer questions with a clear structure:

1. Define the concept precisely.
2. Explain the mechanism.
3. Discuss trade-offs and failure modes.
4. Describe production controls.
5. Close with measurable evaluation or operational feedback.

---

## Recommended Reading Paths

### Path A: Build the AI System Foundation

1. [00-AI-Overview-and-Learning-Path.md](./00-AI-Overview-and-Learning-Path.md)
2. [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md)
3. [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md)
4. [02-MCP-and-Tool-Integration.md](./02-MCP-and-Tool-Integration.md)
5. [08-AI-Safety-and-Alignment.md](./08-AI-Safety-and-Alignment.md)

Use this path if you are preparing for backend, full-stack, AI application, or platform engineering interviews.

### Path B: Focus on Model Training and ML Systems

1. [00-AI-Overview-and-Learning-Path.md](./00-AI-Overview-and-Learning-Path.md)
2. [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md)
3. [04-AI-Model-Training-and-Optimization.md](./04-AI-Model-Training-and-Optimization.md)
4. [05-Recommendation-Models-and-Training.md](./05-Recommendation-Models-and-Training.md)
5. [07-Multimodal-and-Vision-AI.md](./07-Multimodal-and-Vision-AI.md)

Use this path if you are targeting ML engineer, recommendation engineer, applied scientist, or model infrastructure roles.

### Path C: Focus on Agentic AI and Claude Architecture

1. [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md)
2. [02-MCP-and-Tool-Integration.md](./02-MCP-and-Tool-Integration.md)
3. [06-Claude-Architect-Certification-Knowledge-System.md](./06-Claude-Architect-Certification-Knowledge-System.md)
4. [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md)
5. [08-AI-Safety-and-Alignment.md](./08-AI-Safety-and-Alignment.md)

Use this path if you are preparing for agent platform, AI tooling, developer productivity, or LLM architecture interviews.

---

## Difficulty Levels

### L1: Foundation

You should be able to explain:

- What an LLM is and why next-token prediction can produce useful language behavior.
- Why tokenization, context windows, temperature, top-p, and KV cache matter.
- The difference between prompt engineering, RAG, tool calling, and agents.
- Why an AI application is more than a single API call.

Recommended documents:

- [00-AI-Overview-and-Learning-Path.md](./00-AI-Overview-and-Learning-Path.md)
- [01-LLM-and-Agent-Fundamentals.md](./01-LLM-and-Agent-Fundamentals.md)

### L2: Production Engineering

You should be able to design:

- Model routing, prompt management, retrieval pipelines, tool execution, and output validation.
- Offline evaluation, online monitoring, tracing, human review, and rollback.
- Permission boundaries for tools and resources.
- Cost, latency, reliability, and safety controls.

Recommended documents:

- [03-AI-Engineering-Practices.md](./03-AI-Engineering-Practices.md)
- [02-MCP-and-Tool-Integration.md](./02-MCP-and-Tool-Integration.md)
- [08-AI-Safety-and-Alignment.md](./08-AI-Safety-and-Alignment.md)

### L3: Advanced Model and Architecture Topics

You should be able to reason about:

- Pretraining, SFT, RLHF, DPO, LoRA, QLoRA, quantization, distillation, and distributed training.
- Recommendation models, embedding systems, sequence models, multi-task learning, and training-serving skew.
- Vision transformers, CLIP, diffusion models, multimodal RAG, document understanding, and video understanding.
- Claude-style agent loops, tool schemas, hooks, session management, prompt chaining, and context reliability.

Recommended documents:

- [04-AI-Model-Training-and-Optimization.md](./04-AI-Model-Training-and-Optimization.md)
- [05-Recommendation-Models-and-Training.md](./05-Recommendation-Models-and-Training.md)
- [06-Claude-Architect-Certification-Knowledge-System.md](./06-Claude-Architect-Certification-Knowledge-System.md)
- [07-Multimodal-and-Vision-AI.md](./07-Multimodal-and-Vision-AI.md)

---

## Document Map

| No. | English Document | Chinese Document | Main Interview Focus |
|---|---|---|---|
| 00 | [AI Overview and Learning Path](./00-AI-Overview-and-Learning-Path.md) | [AI入门与全景认知](../AI知识库/00-AI入门与全景认知.md) | Mental model, learning order, concept boundaries |
| 01 | [LLM and Agent Fundamentals](./01-LLM-and-Agent-Fundamentals.md) | [LLM与Agent基础](../AI知识库/01-LLM与Agent基础.md) | Transformer, tokenization, RAG, memory, agents |
| 02 | [MCP and Tool Integration](./02-MCP-and-Tool-Integration.md) | [MCP与工具集成](../AI知识库/02-MCP与工具集成.md) | MCP protocol, tools, resources, transport, security |
| 03 | [AI Engineering Practices](./03-AI-Engineering-Practices.md) | [AI工程化实践](../AI知识库/03-AI工程化实践.md) | Architecture, evaluation, cost, latency, observability |
| 04 | [AI Model Training and Optimization](./04-AI-Model-Training-and-Optimization.md) | [AI模型训练与优化](../AI知识库/04-AI模型训练与优化.md) | Training lifecycle, fine-tuning, alignment, quantization |
| 05 | [Recommendation Models and Training](./05-Recommendation-Models-and-Training.md) | [推荐模型与训练](../AI知识库/05-推荐模型与训练.md) | Embeddings, ranking models, retrieval, online learning |
| 06 | [Claude Architect Certification Knowledge System](./06-Claude-Architect-Certification-Knowledge-System.md) | [Claude架构师认证知识体系](../AI知识库/06-Claude架构师认证知识体系.md) | Claude API, agentic loops, hooks, context reliability |
| 07 | [Multimodal and Vision AI](./07-Multimodal-and-Vision-AI.md) | [多模态与视觉AI](../AI知识库/07-多模态与视觉AI.md) | Vision, multimodal models, image generation, OCR, video |
| 08 | [AI Safety and Alignment](./08-AI-Safety-and-Alignment.md) | [AI安全与对齐](../AI知识库/08-AI安全与对齐.md) | Alignment, RLHF, guardrails, red teaming, governance |

---

## How to Answer in English Interviews

Use a five-part answer pattern:

1. **Concept**: "At a high level, X is..."
2. **Mechanism**: "Mechanically, it works by..."
3. **Trade-off**: "The benefit is..., but the cost/risk is..."
4. **Production practice**: "In production I would add..."
5. **Validation**: "I would evaluate it with..."

Example:

> "RAG is a pattern where we retrieve relevant external knowledge and place it into the model context before generation. Mechanically, it has indexing, retrieval, reranking, context construction, generation, and citation validation. The main trade-off is that it improves freshness and traceability without model training, but introduces retrieval failures, stale chunks, and latency. In production I would add chunk versioning, hybrid retrieval, reranking, permission filtering, citation checks, and offline/online evals."

---

## Interview Q&A Style

- Use **Interview Self-Check** for quick recall prompts.
- Use **Senior Interview Q&A** for answer-ready scenarios involving design trade-offs, evaluation, failure modes, cost/latency, security, compliance, and online troubleshooting.
- Keep English answers interview-ready rather than literal translations of the Chinese source.

---

## Maintenance Notes

- Keep file numbers aligned with the Chinese directory.
- Add the language link at the top of every English document, pointing to the matching Chinese file in `../AI知识库/`.
- Prefer interview-ready English over literal translation.
- Keep senior Q&A concise but answer-ready: every question should include a clear decision path, risks, and measurable validation where applicable.
- When the Chinese source changes, update the matching English document in the same numbered position.
- Global indexes such as the root `README.md`, `cs/README.md`, `KNOWLEDGE_BASE_MAP.md`, `DIRECTORY_TREE.txt`, and `QUICK_REFERENCE.md` should be updated by the parent task to avoid conflicts.
