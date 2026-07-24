# Specialized Knowledge Base

Language: English | [中文](../专项知识库/)

This directory is the English mirror for selected documents from `cs/专项知识库`. It intentionally covers only documents 03-06, focusing on operations, SRE, testing, and IaaS infrastructure.

---

## Purpose

Use this collection as a senior-engineer learning and interview preparation track:

- Build fundamentals first, then connect them to production troubleshooting.
- Learn the operating procedures, trade-offs, and failure modes behind common platform systems.
- Practice answers from a senior interviewer perspective: not just "what command to run", but what signal proves or disproves a hypothesis.

---

## Document Map

| No. | English Document | Chinese Source | Focus |
|-----|------------------|----------------|-------|
| 03 | [Linux Operations and Troubleshooting](./03-Linux-Operations-and-Troubleshooting.md) | [Linux 运维与排障](../专项知识库/03-Linux运维与排障.md) | Linux commands, CPU, memory, network, disk I/O, crash dump, kernel memory, senior Q&A |
| 04 | [Advanced SRE Practices](./04-Advanced-SRE-Practices.md) | [SRE实践全景](../专项知识库/04-SRE高级实践.md) | Observability, incident response, SLO, error budget, change governance, capacity, FinOps, chaos |
| 05 | [Comprehensive Software Testing](./05-Comprehensive-Software-Testing.md) | [软件测试综合](../专项知识库/05-软件测试综合.md) | Test pyramid, mocks, contract testing, mutation testing, test data, performance testing, CI quality gates |
| 06 | [IaaS Infrastructure Technologies](./06-IaaS-Infrastructure-Technologies.md) | [IaaS基础设施技术](../专项知识库/06-IaaS基础设施技术.md) | KVM/QEMU, VM networking, VXLAN/VPC, server hardware, DPDK/SPDK/RDMA, Linux tuning |

---

## Reading Path

1. Start with [03 Linux Operations and Troubleshooting](./03-Linux-Operations-and-Troubleshooting.md) to build the node-level troubleshooting baseline.
2. Continue with [04 Advanced SRE Practices](./04-Advanced-SRE-Practices.md) to connect node signals to user-facing reliability, SLOs, incidents, and governance.
3. Read [05 Comprehensive Software Testing](./05-Comprehensive-Software-Testing.md) to understand how engineering teams prevent regressions before incidents happen.
4. Finish with [06 IaaS Infrastructure Technologies](./06-IaaS-Infrastructure-Technologies.md) to go deeper into cloud infrastructure, virtualization, networking, and hardware-backed performance.

Recommended review rhythm:

- First pass: skim headings and diagrams to build a map.
- Second pass: rehearse SOPs and explain each decision point out loud.
- Third pass: answer the interview questions without looking, then compare with the reference answers.

---

## Interview Answering Guide

For senior interviews, avoid command lists without reasoning. A strong answer usually has this shape:

1. Define the symptom and blast radius.
2. Split the problem by layer: user path, application, runtime, OS, network, storage, infrastructure.
3. Pick one or two decisive signals per layer.
4. Explain the stop-the-bleeding action before deep root cause analysis.
5. Close the loop with validation, rollback, and follow-up prevention.

Good answers also include trade-offs:

- Fast mitigation versus durable fix.
- Accuracy versus alert noise.
- Consistency versus availability.
- Performance versus operability.
- Cost optimization versus reliability margin.

---

## Maintenance Notes

- Keep numbering aligned with the Chinese source documents.
- Each English document should start with `Language: English | [中文](...)`.
- If Chinese content receives high-value additions, sync the same concept into the corresponding English document.
- Do not add low-signal filler. These notes are meant to stay useful for real production review and senior-level interviews.
- Keep this English directory scoped to README plus documents 03-06 unless the source scope changes.

---

## Chinese Links

- [专项知识库目录](../专项知识库/)
- [03 Linux 运维与排障](../专项知识库/03-Linux运维与排障.md)
- [04 SRE 高级实践](../专项知识库/04-SRE高级实践.md)
- [05 软件测试综合](../专项知识库/05-软件测试综合.md)
- [06 IaaS 基础设施技术](../专项知识库/06-IaaS基础设施技术.md)
