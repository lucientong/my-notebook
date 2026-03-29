# Claude 架构师认证知识体系

> **定位**：基于 Anthropic [Claude Certified Architect — Foundations](https://github.com/paullarionov/claude-certified-architect) 认证体系（84 页官方学习指南），系统性提炼 AI Agent 架构设计的工程最佳实践。内容不限于 Claude 本身，**核心设计模式可迁移到任何 LLM 平台**。
>
> **前置建议**：`01-LLM与Agent基础`（Agent/Prompt 基础）、`02-MCP与工具集成`（MCP 协议基础）、`03-AI工程化实践`（LLM 应用架构）

---

## 📑 目录

### 第一部分：Agent 架构与编排（权重 27%）
1. [Claude API 基础](#1-claude-api-基础)
2. [Tool Use 与 JSON Schema](#2-tool-use-与-json-schema)
3. [Agent SDK 与 Agentic Loop](#3-agent-sdk-与-agentic-loop)
4. [生命周期 Hooks](#4-生命周期-hooks)
5. [Session 管理](#5-session-管理)

### 第二部分：Tool 设计与 MCP 集成（权重 18%）
6. [MCP 协议深入](#6-mcp-协议深入)
7. [Tool 描述工程](#7-tool-描述工程)

### 第三部分：Claude Code 配置与工作流（权重 20%）
8. [CLAUDE.md 配置体系](#8-claudemd-配置体系)
9. [Slash Commands 与 Skills](#9-slash-commands-与-skills)
10. [Planning Mode 与迭代开发](#10-planning-mode-与迭代开发)
11. [Claude Code CLI 与 CI/CD](#11-claude-code-cli-与-cicd)

### 第四部分：Prompt Engineering 与结构化输出（权重 20%）
12. [Few-shot Prompting](#12-few-shot-prompting)
13. [显式标准 vs 模糊指令](#13-显式标准-vs-模糊指令)
14. [Prompt Chaining 与动态分解](#14-prompt-chaining-与动态分解)
15. [校验-重试循环](#15-校验-重试循环)
16. [Message Batches API](#16-message-batches-api)

### 第五部分：上下文管理与可靠性（权重 15%）
17. [长上下文策略](#17-长上下文策略)
18. [溯源保留](#18-溯源保留)
19. [升级与人机协同](#19-升级与人机协同)

### 附录
20. [考试场景速查](#20-考试场景速查)
21. [Quick Reference](#21-quick-reference)
22. [面试题自查](#22-面试题自查)

---

# 第一部分：Agent 架构与编排（27%）

## 1. Claude API 基础

> **官方文档**：[Messages API](https://platform.claude.com/docs/en/api/messages) | [Prompt Engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)

### 1.1 API 请求结构

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a helpful assistant.",
  "messages": [
    {"role": "user", "content": "Hi!"},
    {"role": "assistant", "content": "Hello!"},
    {"role": "user", "content": "How are you?"}
  ],
  "tools": [...],
  "tool_choice": {"type": "auto"}
}
```

**关键参数**：

| 参数 | 说明 | 注意事项 |
|:---|:---|:---|
| `model` | 模型版本 | 不同模型有不同能力和成本 |
| `max_tokens` | 最大输出 token 数 | 设置过小可能截断输出 |
| `system` | 系统提示词 | 定义角色、约束、输出格式 |
| `messages` | 对话历史 | **必须包含完整历史**，API 无状态 |
| `tools` | 可用工具列表 | 需要 name、description、input_schema |
| `tool_choice` | 工具选择策略 | auto / any / 指定工具 |

### 1.2 stop_reason 详解

**stop_reason 是判断 Agent 循环是否结束的唯一可靠信号**：

| stop_reason | 含义 | 处理方式 |
|:---|:---|:---|
| `"end_turn"` | 模型自然结束 | 返回结果给用户 |
| `"tool_use"` | 需要调用工具 | 执行工具后继续对话 |
| `"max_tokens"` | 达到 token 限制 | 输出被截断，考虑增加限制 |
| `"stop_sequence"` | 遇到停止序列 | 根据业务逻辑处理 |

```python
# ✅ 正确：使用 stop_reason 判断
while True:
    response = call_claude_api(messages)
    
    if response.stop_reason == "end_turn":
        return response.content  # 任务完成
    
    if response.stop_reason == "tool_use":
        tool_result = execute_tool(response.tool_call)
        messages.append({"role": "tool", "content": tool_result})
        continue

# ❌ 反模式：
# - 解析文本检测 "Task completed"
# - 使用固定迭代次数 max_iterations=5
# - 检查是否产生了文本输出
```

### 1.3 tool_choice 参数

| 值 | 行为 | 使用场景 |
|:---|:---|:---|
| `{"type": "auto"}` | 模型自行决定是否调用工具 | 默认场景 |
| `{"type": "any"}` | 必须调用某个工具 | 保证结构化输出 |
| `{"type": "tool", "name": "xxx"}` | 必须调用指定工具 | 强制执行特定操作 |

```python
# 场景 1：强制结构化输出
# → tool_choice: {"type": "any"} 确保返回结构化数据

# 场景 2：强制执行顺序
# → tool_choice: {"type": "tool", "name": "extract_metadata"}
```

---

## 2. Tool Use 与 JSON Schema

> **官方文档**：[Tool Use](https://platform.claude.com/docs/en/build-with-claude/tool-use)

### 2.1 Tool Use vs 纯文本 JSON

| 维度 | Tool Use | 纯文本 "请返回 JSON" |
|:---|:---|:---|
| **语法保证** | ✅ 自动保证有效 JSON | ❌ 可能括号缺失、逗号错误 |
| **结构保证** | ✅ required 字段必须存在 | ❌ 可能遗漏字段 |
| **类型保证** | ✅ number 必须是数字 | ❌ 数字可能返回为字符串 |
| **空值语义** | ✅ nullable 明确处理 | ❌ 可能省略或编造 |

### 2.2 JSON Schema 设计最佳实践

```json
{
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "enum": ["bug", "feature", "docs", "unclear", "other"]
    },
    "category_detail": {
      "type": ["string", "null"],
      "description": "当 category 为 'other' 或 'unclear' 时的说明"
    },
    "severity": {
      "type": "string",
      "enum": ["critical", "high", "medium", "low"]
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 1
    },
    "optional_field": {
      "type": ["string", "null"],
      "description": "找不到此信息时返回 null，不要猜测"
    }
  },
  "required": ["category", "severity"]
}
```

**Schema 设计原则**：

| 原则 | 说明 |
|:---|:---|
| **Required vs Optional** | 只将必定存在的字段标记为 required |
| **Nullable 字段** | `"type": ["string", "null"]` 处理可能缺失的信息 |
| **Enum + Other** | 预定义类别加 "other" 兜底 |
| **Enum + Unclear** | 提供 "unclear" 选项表示无法确定 |

### 2.3 语法错误 vs 语义错误

| 错误类型 | 示例 | 解决方案 |
|:---|:---|:---|
| 语法错误 | JSON 格式无效、字段类型错误 | Tool Use + JSON Schema（消除） |
| 语义错误 | 金额不对、日期格式错 | 校验检查 + 重试反馈 |

---

## 3. Agent SDK 与 Agentic Loop

> **官方文档**：[Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | [Subagents](https://platform.claude.com/docs/en/agent-sdk/subagents)

### 3.1 Agentic Loop 执行流程

```
    ┌──────────────────────────────────────────────────────────┐
    │                                                          │
    ↓                                                          │
┌────────┐     ┌─────────────┐     ┌─────────────────┐        │
│ 发送   │────→│ 收到响应    │────→│ 检查            │        │
│ 请求   │     │             │     │ stop_reason     │        │
└────────┘     └─────────────┘     └────────┬────────┘        │
                                            │                  │
                    ┌───────────────────────┼───────────────┐  │
                    ↓                       ↓               │  │
            ┌───────────────┐       ┌───────────────┐       │  │
            │ "tool_use"    │       │ "end_turn"    │       │  │
            │ 执行工具      │       │ 任务完成      │       │  │
            │ 追加到 msgs   │       │ 返回结果      │       │  │
            └───────┬───────┘       └───────────────┘       │  │
                    └───────────────────────────────────────┘  │
                                    │                          │
                                    └──────────────────────────┘
```

### 3.2 AgentDefinition 配置

```python
from claude_agent_sdk import AgentDefinition

agent = AgentDefinition(
    name="customer_support",
    description="处理客户退货和订单问题",
    system_prompt="""你是一个客服代理。
    
职责：
- 查询客户信息和订单状态
- 处理退货请求（金额 ≤ $100）
- 超过权限范围时升级到人工

约束：
- 不可修改账户信息
- 不可访问支付卡号
- 金额 > $100 必须升级""",
    allowed_tools=["get_customer", "lookup_order", "process_refund", "escalate_to_human"],
)

# Coordinator 需要包含 Task 工具用于委托子代理：
coordinator = AgentDefinition(
    allowed_tools=["Task", "get_customer", "lookup_order"],
)
```

### 3.3 Hub-Spoke 拓扑（Multi-Agent 核心架构）

```
                    ┌─────────────────────┐
                    │    Coordinator      │
                    │    （协调者）        │
                    │                     │
                    │ 职责：              │
                    │ - 任务分解          │
                    │ - 动态选择子代理    │
                    │ - 聚合验证结果      │
                    │ - 错误处理与重试    │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ↓                   ↓                   ↓
   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
   │ Search Agent  │   │ Analysis Agent│   │Synthesis Agent│
   │ （检索代理）   │   │ （分析代理）  │   │ （综合代理）  │
   └───────────────┘   └───────────────┘   └───────────────┘
```

**关键原则：子代理上下文隔离**

```python
# ❌ 错误：缺少上下文
Task: "分析这个文档"
# 子代理：什么文档？在哪里？要分析什么？

# ✅ 正确：显式传递完整上下文
Task: """分析以下文档。

文档内容：
[完整文档文本或关键摘要]

之前的搜索结果：
[相关背景信息]

输出格式要求：
{
  "key_findings": [...],
  "confidence": 0.0-1.0
}
"""
```

**为什么必须通过 Coordinator 中转？**

| 优势 | 说明 |
|:---|:---|
| **全局可观测性** | 所有交互经过 Coordinator，可完整记录日志 |
| **统一错误处理** | 子代理失败时，由 Coordinator 统一决策 |
| **信息过滤** | 控制每个子代理接收的上下文量 |
| **灵活编排** | 可动态调整执行顺序、并行/串行策略 |

### 3.4 Task 工具与并行执行

```python
# Coordinator 可以在一个响应中调用多个 Task，并行执行

# Coordinator 的一次响应：
Task 1: "搜索关于 AI 音乐创作的最新文章"
Task 2: "分析上传的行业报告文档"  
Task 3: "搜索关于 AI 音乐版权问题的资料"

# 三个 Task 并发执行，结果汇总后返回给 Coordinator
```

**上下文传递决策**：

| 传递内容 | 适用场景 | 风险 |
|:---|:---|:---|
| 原始数据 | 需要完整信息的任务 | 上下文溢出 |
| 摘要/提取物 | 下游只需关键发现 | 可能丢失细节 |
| 结构化结果 | 明确的数据管道 | 需预定义 Schema |

---

## 4. 生命周期 Hooks

> **官方文档**：[Agent SDK Hooks](https://platform.claude.com/docs/en/agent-sdk/hooks)

### 4.1 Hook 类型

```
Agent 执行生命周期：

  ┌──────┐   ┌──────────────┐   ┌───────────┐   ┌──────────────┐   ┌──────┐
  │Start │──→│ PreToolUse   │──→│ Tool Call │──→│ PostToolUse  │──→│ Next │
  └──────┘   │ Hook         │   │           │   │ Hook         │   │ Step │
             └──────────────┘   └───────────┘   └──────────────┘   └──────┘
```

### 4.2 PreToolUse Hook：调用前拦截

```python
@hook("PreToolUse")
def enforce_refund_limit(tool_call):
    """强制执行退款金额限制"""
    if tool_call.name == "process_refund":
        amount = tool_call.args.get("amount", 0)
        
        if amount > 500:
            return redirect_to_escalation(
                reason=f"退款金额 ${amount} 超过 $500 限制"
            )
    return None  # 允许继续执行
```

### 4.3 PostToolUse Hook：结果处理

```python
@hook("PostToolUse")
def normalize_dates(tool_result):
    """标准化日期格式"""
    if "date" in tool_result:
        if isinstance(tool_result["date"], int):
            tool_result["date"] = datetime.fromtimestamp(
                tool_result["date"]
            ).isoformat()
    return tool_result

@hook("PostToolUse", tool="lookup_order")
def trim_order_fields(result):
    """只保留必要字段，减少上下文占用"""
    return {
        "order_id": result["order_id"],
        "status": result["status"],
        "total": result["total"],
        "items": result["items"],
        "return_eligible": result["return_eligible"]
    }
```

### 4.4 Hooks vs Prompt 指令

| 属性 | Hooks | Prompt 指令 |
|:---|:---|:---|
| 执行保证 | 确定性（100%） | 概率性（>90%，非 100%） |
| 实现层面 | 代码级强制 | 模型"理解"后执行 |
| 适用场景 | 关键业务规则、合规、安全 | 偏好、建议、风格指导 |
| 可被绕过 | 不可能 | 模型可能忽略 |

**选择原则**：

```
✅ 用 Hook（硬约束）——违反会造成严重后果：
   - 金额超过阈值必须人工审批
   - 所有 Tool 调用必须记审计日志
   - 敏感字段必须脱敏
   - PII 数据必须加密

❌ 不用 Hook（软指导）——可被灵活解读：
   - "语气要友好"
   - "优先使用简短回复"
   - "先确认用户意图"

黄金法则：
失败会导致金融、法律、安全问题 → 必须用 Hook
只是用户体验或效率问题 → Prompt 指令即可
```

---

## 5. Session 管理

> **官方文档**：[Agent SDK Sessions](https://platform.claude.com/docs/en/agent-sdk/sessions)

### 5.1 --resume 恢复会话

```bash
claude --resume investigation-auth-bug
# 继续之前的命名会话，保留对话历史和上下文
```

**风险提示**：
- 文件在暂停期间被修改，工具结果可能过时
- 时间过长后上下文可能"降解"

### 5.2 fork_session 会话分支

```
    代码库调查
         │
    fork_session
    /           \
方案 A:          方案 B:
使用 Redux      使用 Context API
   │                │
 探索...          探索...
   ↓                ↓
发现问题          可行！
（丢弃）          （采用）

特性：
- 两个分支继承 fork 点之前的所有上下文
- fork 之后的操作互不影响
```

### 5.3 何时开始新会话 vs 恢复

```
文件是否在暂停期间被修改？
  ├── 是 → 开始新会话
  └── 否 → 距离上次多久？
              ├── 几小时内 → 恢复
              └── 超过一天 → 新会话 + 简短摘要
```

---

# 第二部分：Tool 设计与 MCP 集成（18%）

## 6. MCP 协议深入

> **官方文档**：[MCP](https://modelcontextprotocol.io/) | [Tools](https://modelcontextprotocol.io/docs/concepts/tools) | [Resources](https://modelcontextprotocol.io/docs/concepts/resources)

### 6.1 MCP 三要素

| 类型 | 功能 | 调用方 | 示例 |
|:---|:---|:---|:---|
| **Tools** | 执行操作、产生副作用 | LLM 主动调用 | process_refund, send_email |
| **Resources** | 提供只读数据/上下文 | 应用程序注入 | database_schema, policy_doc |
| **Prompts** | 预定义提示模板 | 按场景调用 | code_review_template |

### 6.2 Tool vs Resource 选型

| 特征 | 用 Tool | 用 Resource |
|:---|:---|:---|
| **是否有副作用？** | ✅ 修改数据 | ❌ 只读 |
| **谁决定使用？** | LLM 自主判断 | 应用程序控制 |
| **是否需要参数？** | 通常需要动态参数 | 通常静态 |
| **典型场景** | 查询订单、退款 | 公司政策、Schema |

**Resource 优势示例**：

```python
# 不用 Resource：需要多次工具调用探索
# Step 1: list_tables() → 获取表名
# Step 2: describe_table("orders") → 获取结构
# Step 3: 才能写查询

# 使用 Resource：Schema 直接注入到 Prompt
# Agent 可以直接写查询，无需探索
```

### 6.3 MCP Server 配置

**项目级（.mcp.json）——团队共享**：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**用户级（~/.claude.json）——个人实验**：
- 不纳入版本控制
- 适合个人测试

**配置选择**：
- 标准集成 → 优先用社区 MCP Server
- 团队共享 → 项目级 `.mcp.json`
- 个人实验 → 用户级 `~/.claude.json`

### 6.4 isError 标志与结构化错误

```json
// ❌ 反模式：无用的错误信息
{
  "isError": true,
  "content": "Operation failed"
}

// ✅ 最佳实践：结构化错误
{
  "isError": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "服务暂时不可用",
    "attempted_query": "order_id=12345",
    "suggestions": ["稍后重试", "使用 search_order_by_email"]
  }
}
```

**错误分类**：

| errorCategory | 含义 | isRetryable |
|:---|:---|:---|
| `transient` | 临时故障 | true |
| `not_found` | 资源不存在 | false |
| `validation` | 参数错误 | true（改参数后） |
| `permission` | 权限不足 | false |

---

## 7. Tool 描述工程

### 7.1 Tool 描述的重要性

**LLM 选择调用哪个 Tool，完全依赖 name 和 description**。

```
描述质量    │ 路由准确率
───────────┼──────────────
极差       │ < 50%
普通       │ 70-80%
良好       │ 90-95%
优秀       │ > 98%
```

### 7.2 消歧设计

```json
// ❌ 语义重叠：45% 概率路由错误
{
  "analyze_content": "分析内容并提取关键信息",
  "analyze_document": "分析文档并提取关键信息"
}

// ✅ 消歧后：明确边界
{
  "extract_web_results": "处理从网络检索到的信息。仅用于网络数据，不处理本地文件。",
  "analyze_uploaded_document": "分析用户上传的本地文件（PDF、Word）。不处理 URL。"
}
```

**消歧四原则**：

| 原则 | 说明 | 示例 |
|:---|:---|:---|
| **名称要具体** | 用 `动作_对象` 命名 | `search_order_by_id` ✅ |
| **描述要互斥** | 包含"不做什么" | "不处理 URL" |
| **边界要明确** | 列出输入类型 | "接受 PDF/DOCX" |
| **避免同义词** | 不同 Tool 用不同动词 | extract vs retrieve |

### 7.3 完整 Tool 描述模板

```json
{
  "name": "search_order_by_id",
  "description": "根据订单 ID 查询订单详情。返回状态、金额、商品列表。当用户提供订单号（格式：ORD-XXXXX）时使用。如果只有邮箱，请使用 search_order_by_contact。",
  "input_schema": {
    "type": "object",
    "required": ["order_id"],
    "properties": {
      "order_id": {
        "type": "string",
        "description": "订单 ID，格式为 ORD-12345 或纯数字"
      }
    }
  }
}
```

---

# 第三部分：Claude Code 配置与工作流（20%）

## 8. CLAUDE.md 配置体系

> **官方文档**：[Claude Code Memory / CLAUDE.md](https://code.claude.com/docs/en/memory)

### 8.1 三层配置继承

```
┌──────────────────────────────────────────────────────────────┐
│ 第一层：~/.claude/CLAUDE.md（用户级）                         │
│ 作用：所有项目通用的个人偏好                                  │
│ 不纳入版本控制                                                │
└──────────────────────────┬───────────────────────────────────┘
                           ↓ 继承
┌──────────────────────────────────────────────────────────────┐
│ 第二层：项目根目录/CLAUDE.md（项目级）                        │
│ 作用：团队共享的项目规范                                      │
│ 纳入版本控制                                                  │
└──────────────────────────┬───────────────────────────────────┘
                           ↓ 继承
┌──────────────────────────────────────────────────────────────┐
│ 第三层：.claude/rules/*.md（路径规则）                        │
│ 作用：特定目录/文件类型的规则                                 │
│ 使用 paths 字段指定匹配                                       │
└──────────────────────────────────────────────────────────────┘

优先级：路径规则 > 项目配置 > 用户配置
```

### 8.2 @path 语法引用外部文件

```markdown
# 项目 CLAUDE.md

## 代码规范
@./standards/go-style.md

## 测试规范  
@./standards/testing.md
```

### 8.3 路径规则（.claude/rules/）

```yaml
# .claude/rules/api-rules.md
---
paths:
  - "api/**/*"
  - "internal/handlers/**/*"
---

# API 代码规范

## 必须遵守
- 所有 API 必须有请求参数校验
- 所有错误返回标准格式 {code, message}

## 禁止事项
- 禁止在 handler 中直接操作数据库
```

### 8.4 CLAUDE.md 编写最佳实践

```markdown
# ✅ 好的示例

## 项目概述
Go 微服务项目，使用 gRPC + protobuf。

## 代码规范
- 所有错误必须处理，禁止 `_` 忽略 error
- 使用 `slog` 日志，不用 `fmt.Println`

## 禁止事项
- 不要修改 `internal/generated/` 下的文件
```

**常见错误**：

| 错误 | 问题 | 建议 |
|:---|:---|:---|
| 过于冗长 | 消耗上下文 | < 200 行 |
| 矛盾规则 | 无法遵守 | 仔细检查一致性 |
| 含糊指导 | "写好的代码" | 给出具体标准 |

---

## 9. Slash Commands 与 Skills

> **官方文档**：[Claude Code Skills](https://code.claude.com/docs/en/skills)

### 9.1 自定义 Slash Commands

```
项目命令（团队共享）：.claude/commands/
用户命令（个人使用）：~/.claude/commands/

使用：/review → 执行代码审查命令
```

### 9.2 Agent Skills

```yaml
# .claude/skills/security-review/SKILL.md
---
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "要审查的文件或目录路径"
---

# 安全审查技能

## 检查清单
1. SQL 注入
2. XSS
3. 硬编码密钥
```

**关键配置**：

| 配置项 | 说明 |
|:---|:---|
| `context: fork` | Fork 独立上下文，防止污染主会话 |
| `allowed-tools` | 限制可用工具（最小权限） |
| `argument-hint` | 参数提示 |

---

## 10. Planning Mode 与迭代开发

### 10.1 Planning Mode vs Direct Execution

| 使用 Planning Mode | 使用 Direct Execution |
|:---|:---|
| 大规模变更（几十个文件） | 单文件修复 |
| 多种可行方案 | 明确的实现方案 |
| 不熟悉的代码库 | 已知的简单修改 |

### 10.2 Explore 子代理

```
主 Agent："调查支付模块的依赖"
  ↓
委托给 Explore 子代理（独立上下文）
  ↓
读取 15 个文件，追踪 import
  ↓
返回摘要："Payments 依赖 AuthService、OrderModel"
  ↓
主上下文只增加一行摘要（而非 15 个文件）
```

### 10.3 /compact 命令

```
上下文被冗长输出填满时，使用 /compact 压缩：
- 摘要早期对话
- 释放上下文空间

风险：精确数值、日期可能在摘要中丢失
```

### 10.4 迭代优化模式

**"访谈"模式**：让 Claude 先提问，挖掘隐含需求

```
用户："为订单添加折扣功能"
Claude："在实现之前确认：
1. 折扣是按订单还是按商品？
2. 支持百分比和固定金额吗？
3. 折扣可以叠加吗？"
```

---

## 11. Claude Code CLI 与 CI/CD

> **官方文档**：[GitHub Actions](https://code.claude.com/docs/en/github-actions) | [Headless](https://code.claude.com/docs/en/headless)

### 11.1 --print 模式

```bash
# -p：非交互模式，CI/CD 唯一正确方式
claude -p "分析这个 PR 的安全问题"
```

### 11.2 结构化输出

```bash
claude -p "审查这个 PR" \
  --output-format json \
  --json-schema '{...}'

# 输出可被解析，用于自动发布 PR comments
```

### 11.3 独立实例审查

```
问题：同一会话生成的代码，自己审查效果差
原因：模型已"认定"决策是对的，不会质疑自己

解决：使用独立的 Claude 实例审查
- 新实例没有生成时的推理上下文
- 相当于"新鲜的眼睛"
```

---

# 第四部分：Prompt Engineering 与结构化输出（20%）

## 12. Few-shot Prompting

### 12.1 Few-shot 示例类型

**类型 1：处理模糊场景**

```
请求："我的订单坏了"
动作：调用 get_customer → lookup_order
原因："坏了"需要订单详情判断

请求："给我找个经理"
动作：立即 escalate_to_human
原因：客户明确要求人工
```

**类型 2：输出格式规范**

```json
{
  "location": "src/auth/login.ts:42",
  "issue": "SQL 注入风险",
  "severity": "critical",
  "suggested_fix": "使用参数化查询"
}
```

**类型 3：区分可接受 vs 问题代码**

```javascript
// 可接受：
const items = data.filter(x => x.active);

// 问题：
const items = data.filter(x => x.active == true);
// 应使用 ===
```

### 12.2 格式规范化规则

```
规范化要求：
- 日期：ISO 8601 (YYYY-MM-DD)；"昨天" → 计算绝对日期
- 货币：数值 + 货币代码；"五美元" → {"amount": 5, "currency": "USD"}
- 百分比：小数；"一半" → 0.5
```

---

## 13. 显式标准 vs 模糊指令

### 13.1 显式标准示例

```
❌ 模糊：
"检查代码注释的准确性。"
"保守一些——只报告高置信度的发现。"

✅ 显式：
仅在以下情况标记注释为问题：
1. 注释描述的行为与实际代码行为矛盾
2. 注释引用了不存在的函数
3. TODO 指向的 bug 已修复

不要标记：
- 风格过时的注释
- 措辞略有不准确的注释
```

### 13.2 严重等级定义

```
CRITICAL：用户运行时失败
  示例：支付时 NullPointerException

HIGH：安全漏洞
  示例：SQL 注入、XSS

MEDIUM：逻辑 bug，无即时影响
  示例：排序错误、off-by-one

LOW：代码质量问题
  示例：重复代码
```

---

## 14. Prompt Chaining 与动态分解

### 14.1 Prompt Chaining

```
步骤 1：分析 auth.ts → 局部问题列表
步骤 2：分析 database.ts → 局部问题列表
步骤 3：整合分析 → 跨文件依赖问题

优势：避免注意力稀释，每个文件分析质量一致
```

### 14.2 Prompt Chaining vs 动态分解

| 方式 | 适用场景 |
|:---|:---|
| **Prompt Chaining** | 可预测的重复任务 |
| **动态分解** | 开放式探索任务 |

---

## 15. 校验-重试循环

### 15.1 标准模式

```python
MAX_RETRIES = 3

for attempt in range(MAX_RETRIES):
    result = llm.extract(document, schema)
    errors = validate(result)
    
    if not errors:
        return result
    
    # 带反馈重试
    feedback = f"上一次提取有错误：{errors}\n请修正。"

# 所有重试失败 → 人工处理
flag_for_human_review(document)
```

### 15.2 重试有效 vs 无效

| 重试有效 | 重试无效 |
|:---|:---|
| 格式错误 | 信息不存在 |
| 结构错误 | 需要外部文档 |
| 算术不一致 | 需要外部知识 |

### 15.3 自我校正模式

```json
{
  "stated_total": "$150.00",
  "calculated_total": "$145.00",
  "conflict_detected": true
}
// 声明值 ≠ 计算值 → 触发人工审核
```

---

## 16. Message Batches API

### 16.1 概述

| 属性 | 值 |
|:---|:---|
| 成本 | 节省 50% |
| 处理窗口 | 最长 24 小时 |
| 多轮 Tool Calling | ❌ 不支持 |

### 16.2 选择

| 任务 | API |
|:---|:---|
| PR 合并前检查 | 同步 |
| 过夜技术债务报告 | 批处理 |
| 万级文档提取 | 批处理 |
| 迭代式代码审查 | 同步 |

---

# 第五部分：上下文管理与可靠性（15%）

## 17. 长上下文策略

### 17.1 关键信息提取

```
=== 案例事实（请勿摘要）===
客户 ID: CUST-12345
订单 ID: ORD-67890
订单金额: $89.99
问题: 物品损坏
===

无论历史如何摘要，这个块始终完整保留。
```

### 17.2 Lost in the Middle 问题

```
召回率
  ↑
  │ ████                    ████
  │ ████████          ████████
  │ ████████████████████████
  └──────────────────────────→
    开头      中间      结尾

关键：开头和结尾记得好，中间容易"遗忘"
```

**应对策略**：

| 策略 | 做法 |
|:---|:---|
| 关键信息前置 | 最重要的指令放开头 |
| 关键信息重复 | 末尾重复关键约束 |
| 结构化标记 | 用 XML 标签标记重要段落 |
| 分块处理 | 超长文档拆成多次调用 |

### 17.3 Scratchpad Files 模式

```markdown
# investigation-scratchpad.md

## 关键发现
- PaymentProcessor 继承自 BaseProcessor
- refund() 被 3 处调用
- 外部 API 限制 100 请求/分钟

## 待调查
- [ ] 限流策略？
```

### 17.4 委托给子代理保护上下文

```
主 Agent："调查支付模块依赖"
  → 子代理：读取 15 个文件
  → 返回：一行摘要

主 Agent 上下文只增加摘要，而非 15 个文件
```

---

## 18. 溯源保留

### 18.1 归属保留

```json
// ✅ 保留归属
{
  "claim": "AI 音乐市场达 32 亿美元",
  "source_name": "全球 AI 音乐报告 2024",
  "publication_date": "2024-06-15",
  "confidence": 0.9
}
```

### 18.2 处理冲突数据

```json
{
  "claim": "AI 音乐占比",
  "values": [
    {"value": "12%", "source": "Spotify 2024", "date": "2024-03"},
    {"value": "8%", "source": "行业协会", "date": "2024-07"}
  ],
  "conflict_detected": true,
  "explanation": "方法论和时间段差异"
}
// 不要随意选一个，保留两者让协调者决定
```

---

## 19. 升级与人机协同

### 19.1 可靠的升级触发

| 触发条件 | 处理 |
|:---|:---|
| 用户明确要求人工 | 立即升级 |
| 政策未覆盖的情况 | 升级 |
| 多次尝试后失败 | 升级 |
| 金额超过阈值 | 升级（用 Hook） |
| 多个匹配结果 | 请求更多信息 |

### 19.2 不可靠的升级方式

| 方式 | 问题 |
|:---|:---|
| 情绪分析 | 情绪 ≠ 复杂度 |
| LLM 自评置信度 | 可能"自信地犯错" |

### 19.3 错误传播策略

```json
{
  "status": "error",
  "error_type": "parse_failure",
  "message": "PDF 损坏",
  "suggestions": ["尝试 OCR", "跳过该文档"]
}
```

**反模式**：
- 静默跳过错误
- 自动重试后直接抛异常
- 中断整个工作流

---

# 附录

## 20. 考试场景速查

| 场景 | 核心技能 |
|:---|:---|
| 客户支持 Agent | Tool 设计、升级、错误处理 |
| Claude Code 代码生成 | CLAUDE.md、Planning Mode |
| Multi-Agent 研究系统 | Hub-Spoke、上下文传递 |
| 数据提取管线 | JSON Schema、校验-重试 |
| CI 代码审查 | --print、结构化输出 |
| 开发者工具 | Skills、Slash Commands |

---

## 21. Quick Reference

### Domain 1: Agent 架构（27%）

| 概念 | 要点 |
|:---|:---|
| stop_reason | "end_turn" 是唯一可靠的完成信号 |
| tool_choice | auto/any/指定工具 |
| Hub-Spoke | 所有通信通过 Coordinator |
| 显式上下文 | 子代理不继承父上下文 |
| Hooks | 硬约束用 Hook，软指导用 Prompt |

### Domain 2: Tool 设计（18%）

| 概念 | 要点 |
|:---|:---|
| Tool 消歧 | 名称具体、描述互斥、避免同义词 |
| 结构化错误 | category + retryable + suggestions |
| Tool vs Resource | 有副作用用 Tool，只读用 Resource |

### Domain 3: Claude Code（20%）

| 概念 | 要点 |
|:---|:---|
| CLAUDE.md | 三层继承，路径规则用 paths |
| Skills | context:fork 隔离上下文 |
| Planning Mode | 大变更用规划，小修复直接执行 |
| --print | CI/CD 唯一正确方式 |

### Domain 4: Prompt 工程（20%）

| 概念 | 要点 |
|:---|:---|
| Few-shot | 比规则描述更有效 |
| 显式标准 | 避免模糊指令 |
| 校验-重试 | 信息不存在时重试无效 |
| Batch API | 50% 省成本，但不支持多轮 Tool |

### Domain 5: 上下文管理（15%）

| 概念 | 要点 |
|:---|:---|
| Lost in the Middle | 关键信息放开头和结尾 |
| Scratchpad | 中间结果写文件 |
| 升级触发 | 规则触发 > 情绪/置信度 |
| 错误传播 | 返回结构化错误给协调者 |

---

## 22. 面试题自查

### Q1: 为什么 Multi-Agent 用 Hub-Spoke 而非直连？

**答案**：Hub-Spoke 让 Coordinator 作为唯一通信中心，带来：
1. **全局可观测性**——所有交互可追踪
2. **统一错误处理**——由 Coordinator 决策重试/跳过/升级
3. **信息过滤**——控制每个子代理的上下文量

---

### Q2: 什么时候用 Hook 而不是 Prompt？

**答案**：
- **Hook**：硬约束，违反会造成严重后果（金额限制、审计日志）
- **Prompt**：软指导，可被灵活解读（语气友好、回复简洁）

原则：**LLM 可能忽略 Prompt，但 Hook 是代码级强制执行**。

---

### Q3: Tool 描述消歧的核心原则？

**答案**：
1. **名称要具体**：`search_order_by_id` 而非 `search`
2. **描述要互斥**：每个 Tool 边界不重叠
3. **避免同义词**：不同 Tool 用不同动词
4. **测试验证**：用模糊输入测试路由准确率

---

### Q4: 结构化提取为什么用 Tool Use？

**答案**：
- **Schema 校验**：自动检查字段完整、类型正确
- **类型安全**：number 就是 number
- **空值语义**：nullable 字段明确标注

配合**校验-重试循环**：提取 → 校验 → 失败则反馈错误重新提取。

---

### Q5: "Lost in the Middle" 怎么应对？

**答案**：
1. **关键信息前置**：最重要的指令放开头
2. **关键信息重复**：末尾重复关键约束
3. **结构化标记**：XML 标签标记重要段落
4. **分块处理**：超长文档拆成多次调用

---

### Q6: 什么时候该升级到人工？

**答案**：

**可靠触发**（基于规则）：
- 用户明确要求人工
- 政策未覆盖的情况
- 多次尝试后失败
- 金额超过阈值

**不可靠**：
- ❌ 情绪分析
- ❌ LLM 自评置信度

---

### Q7: 同步 API vs Batch API？

**答案**：
| 选同步 | 选 Batch |
|:---|:---|
| 用户在等待 | 结果不需实时 |
| 需要多轮 Tool | 单轮请求即可 |
| 延迟敏感 | 成本敏感（省 50%） |

---

### Q8: 子代理失败怎么处理？

**答案**：返回结构化错误给 Coordinator：

```json
{
  "status": "error",
  "error_type": "parse_failure",
  "suggestions": ["尝试 OCR", "跳过文档"]
}
```

Coordinator 决定：重试/跳过/升级。

**反模式**：静默跳过、直接抛异常、中断整个工作流。

---

### 知识体系总结

```
Claude 架构师认证 — 5 大领域知识地图：

Domain 1: Agent 架构与编排（27%）
  ├── Hub-Spoke 拓扑 vs 直连
  ├── Subagent 委托与显式上下文传递
  ├── 生命周期 Hooks（pre_tool / post_tool）
  └── 架构选型（单 Agent / Coordinator / Pipeline）

Domain 2: Tool 设计与 MCP（18%）
  ├── Tool 描述消歧（名称具体、描述互斥）
  ├── 结构化错误返回（category + retryable）
  └── Tool vs Resource 选型

Domain 3: Claude Code 配置（20%）
  ├── CLAUDE.md 三层配置体系
  ├── Agent Skills（context: fork, allowed-tools）
  └── Planning Mode（先计划后执行）

Domain 4: Prompt 与结构化输出（20%）
  ├── Tool Use 提取模式（JSON Schema 驱动）
  ├── 校验-重试循环
  ├── Few-shot 消歧
  └── Message Batches API

Domain 5: 上下文管理与可靠性（15%）
  ├── "Lost in the Middle"（关键信息前置+重复）
  ├── Scratchpad Files
  └── 升级策略（规则触发 > 情绪/自评）
```
