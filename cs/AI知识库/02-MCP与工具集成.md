# MCP 与工具集成

---

## 📑 目录

### MCP协议基础
1. [MCP协议原理](#1-mcp协议原理)
2. [Server与Client架构](#2-server与client架构)
3. [Tool工具设计](#3-tool工具设计)
4. [Resource资源管理](#4-resource资源管理)
5. [Prompt Templates](#5-prompt-templates)

### 进阶特性
6. [Transport传输层](#6-transport传输层)
7. [Sampling采样机制](#7-sampling采样机制)
8. [安全与权限控制](#8-安全与权限控制)
9. [性能优化](#9-性能优化)

### 运维与实战
10. [调试与排障](#10-调试与排障)
11. [实战案例](#11-实战案例)
12. [面试题自查](#12-面试题自查)

---

## 1. MCP协议原理

### 1.1 什么是MCP？

**MCP (Model Context Protocol)**：标准化的AI与外部工具/数据的集成协议

**核心目标**：
- ✅ 统一AI应用与工具的交互方式
- ✅ 标准化Tool、Resource、Prompt的定义
- ✅ 简化AI能力扩展

**架构**：
```
[AI Application / Client]
         ↓
    [MCP Client]
         ↓
    ┌────┴────┐
    ↓         ↓
[MCP Server 1] [MCP Server 2] [MCP Server 3]
知识库服务      文件系统服务    数据库服务
```

### 1.2 核心概念

**1. Tools（工具）**
```json
{
  "name": "search_knowledge",
  "description": "在知识库中搜索内容",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词"
      },
      "top_k": {
        "type": "integer",
        "default": 5,
        "minimum": 1,
        "maximum": 20
      }
    },
    "required": ["query"]
  }
}
```

**2. Resources（资源）**
```json
{
  "uri": "kb://docs/api-guide",
  "name": "API使用指南",
  "description": "完整的API文档",
  "mimeType": "text/markdown"
}
```

**3. Prompts（提示模板）**
```json
{
  "name": "code_review",
  "description": "代码审查模板",
  "arguments": [
    {
      "name": "language",
      "description": "编程语言",
      "required": true
    },
    {
      "name": "code",
      "description": "待审查的代码",
      "required": true
    }
  ]
}
```

### 1.3 通信协议

**JSON-RPC 2.0格式**：

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_knowledge",
    "arguments": {
      "query": "分布式事务",
      "top_k": 3
    }
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "[{\"title\": \"2PC协议\", \"content\": \"...\", \"score\": 0.95}]"
      }
    ]
  }
}
```

**错误响应**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid params",
    "data": {
      "field": "top_k",
      "reason": "必须在1-20之间"
    }
  }
}
```

---

## 2. Server与Client架构

### 2.1 MCP Server实现

**Python SDK示例**：
```python
from mcp.server import Server, Tool
from mcp.server.models import TextContent
import asyncio

# 创建Server
server = Server("knowledge-base-server")

# 定义工具
@server.tool()
async def search_knowledge(query: str, top_k: int = 5) -> str:
    """
    在知识库中搜索内容
    
    Args:
        query: 搜索关键词
        top_k: 返回结果数量，默认5
    
    Returns:
        JSON格式的搜索结果
    """
    # 参数验证
    if not query:
        return json.dumps({"error": "query不能为空"})
    
    if not 1 <= top_k <= 20:
        return json.dumps({"error": "top_k必须在1-20之间"})
    
    # 执行搜索
    results = await vector_db.search(query, top_k)
    
    return json.dumps({
        "results": results,
        "count": len(results)
    }, ensure_ascii=False)

@server.tool()
async def add_document(title: str, content: str, metadata: dict = None) -> str:
    """添加文档到知识库"""
    doc_id = await vector_db.add_document(
        title=title,
        content=content,
        metadata=metadata or {}
    )
    
    return json.dumps({
        "success": True,
        "doc_id": doc_id
    })

# 定义资源
@server.resource("kb://config")
async def get_config():
    """获取知识库配置"""
    return {
        "uri": "kb://config",
        "name": "知识库配置",
        "mimeType": "application/json",
        "text": json.dumps({
            "version": "1.0",
            "max_results": 20,
            "embedding_model": "text-embedding-ada-002"
        })
    }

@server.resource("kb://stats")
async def get_stats():
    """获取统计信息"""
    stats = await vector_db.get_stats()
    return {
        "uri": "kb://stats",
        "name": "统计信息",
        "mimeType": "application/json",
        "text": json.dumps(stats)
    }

# 启动Server
if __name__ == "__main__":
    import mcp.server.stdio
    mcp.server.stdio.run(server)
```

**TypeScript SDK示例**：
```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

// 创建Server
const server = new Server(
  {
    name: "knowledge-base-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  }
);

// 定义工具
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "search_knowledge",
        description: "在知识库中搜索内容",
        inputSchema: {
          type: "object",
          properties: {
            query: { type: "string", description: "搜索关键词" },
            top_k: { type: "number", default: 5 }
          },
          required: ["query"]
        }
      }
    ]
  };
});

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "search_knowledge") {
    const results = await searchKnowledgeBase(args.query, args.top_k);
    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(results)
        }
      ]
    };
  }
  
  throw new Error(`Unknown tool: ${name}`);
});

// 启动Server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 2.2 MCP Client调用

**Python Client**：
```python
from mcp.client import MCPClient
import asyncio

async def main():
    # 创建Client
    client = MCPClient()
    
    # 连接到Server
    await client.connect(
        "knowledge-base",
        command="python",
        args=["-m", "mcp_server"]
    )
    
    # 列出可用工具
    tools = await client.list_tools()
    print("可用工具：", [t.name for t in tools])
    
    # 调用工具
    result = await client.call_tool(
        "search_knowledge",
        {"query": "分布式事务", "top_k": 3}
    )
    print("搜索结果：", result.content[0].text)
    
    # 读取资源
    config = await client.read_resource("kb://config")
    print("配置：", config.text)
    
    # 关闭连接
    await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

**TypeScript Client**：
```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  // 创建Client
  const transport = new StdioClientTransport({
    command: "python",
    args: ["-m", "mcp_server"]
  });
  
  const client = new Client({
    name: "my-client",
    version: "1.0.0"
  }, {
    capabilities: {}
  });
  
  await client.connect(transport);
  
  // 列出工具
  const { tools } = await client.listTools();
  console.log("可用工具：", tools.map(t => t.name));
  
  // 调用工具
  const result = await client.callTool({
    name: "search_knowledge",
    arguments: {
      query: "分布式事务",
      top_k: 3
    }
  });
  console.log("结果：", result.content[0].text);
}

main();
```

---

## 3. Tool工具设计

### 3.1 JSON Schema规范

**完整示例**：
```json
{
  "name": "create_order",
  "description": "创建订单",
  "inputSchema": {
    "type": "object",
    "properties": {
      "user_id": {
        "type": "integer",
        "description": "用户ID",
        "minimum": 1
      },
      "items": {
        "type": "array",
        "description": "商品列表",
        "items": {
          "type": "object",
          "properties": {
            "item_id": { "type": "integer" },
            "quantity": { "type": "integer", "minimum": 1 },
            "price": { "type": "number", "minimum": 0 }
          },
          "required": ["item_id", "quantity", "price"]
        },
        "minItems": 1,
        "maxItems": 100
      },
      "address": {
        "type": "object",
        "properties": {
          "province": { "type": "string" },
          "city": { "type": "string" },
          "detail": { "type": "string", "minLength": 5, "maxLength": 200 }
        },
        "required": ["province", "city", "detail"]
      },
      "payment_method": {
        "type": "string",
        "enum": ["alipay", "wechat", "bank_card"],
        "description": "支付方式"
      },
      "coupon_code": {
        "type": "string",
        "pattern": "^[A-Z0-9]{6,10}$",
        "description": "优惠券代码（可选）"
      }
    },
    "required": ["user_id", "items", "address", "payment_method"]
  }
}
```

**常用类型**：

| JSON类型 | 验证属性 | 示例 |
|----------|----------|------|
| `string` | minLength, maxLength, pattern, format | `{"type": "string", "format": "email"}` |
| `number` | minimum, maximum, exclusiveMinimum | `{"type": "number", "minimum": 0}` |
| `integer` | minimum, maximum, multipleOf | `{"type": "integer", "multipleOf": 10}` |
| `boolean` | - | `{"type": "boolean"}` |
| `array` | items, minItems, maxItems, uniqueItems | `{"type": "array", "items": {"type": "string"}}` |
| `object` | properties, required, additionalProperties | `{"type": "object", "properties": {...}}` |
| `enum` | enum | `{"type": "string", "enum": ["A", "B"]}` |

**format支持**：
```json
{
  "date": {"type": "string", "format": "date"},           // "2024-01-01"
  "datetime": {"type": "string", "format": "date-time"},  // "2024-01-01T12:00:00Z"
  "email": {"type": "string", "format": "email"},         // "user@example.com"
  "uri": {"type": "string", "format": "uri"},             // "https://example.com"
  "ipv4": {"type": "string", "format": "ipv4"},           // "192.168.1.1"
  "uuid": {"type": "string", "format": "uuid"}            // "123e4567-e89b-12d3-..."
}
```

### 3.2 Tool设计最佳实践

**1. 单一职责**
```python
# ❌ 不好：一个工具做多件事
@server.tool()
async def database_operation(operation: str, table: str, data: dict):
    if operation == "insert":
        return insert(table, data)
    elif operation == "update":
        return update(table, data)
    elif operation == "delete":
        return delete(table, data)

# ✅ 好：每个工具专注一件事
@server.tool()
async def insert_record(table: str, data: dict):
    """插入记录"""
    return await db.insert(table, data)

@server.tool()
async def update_record(table: str, id: int, data: dict):
    """更新记录"""
    return await db.update(table, id, data)

@server.tool()
async def delete_record(table: str, id: int):
    """删除记录"""
    return await db.delete(table, id)
```

**2. 清晰的描述**
```python
# ❌ 不好：描述不清楚
@server.tool()
async def tool1(x, y):
    """处理数据"""
    pass

# ✅ 好：详细描述输入输出和使用场景
@server.tool()
async def search_knowledge_base(
    query: str,
    top_k: int = 5,
    filter_type: str = None
) -> str:
    """
    在知识库中搜索相关内容
    
    适用场景：
    - 查找技术文档
    - 查询历史记录
    - 检索代码示例
    
    Args:
        query: 搜索关键词，支持中英文，建议10-50字
        top_k: 返回结果数量，默认5，范围1-20
        filter_type: 过滤类型，可选值：document/code/wiki
    
    Returns:
        JSON格式的搜索结果列表，每个结果包含：
        - title: 标题
        - content: 内容摘要（最多200字）
        - score: 相关性分数（0-1）
        - url: 原文链接
    
    Example:
        >>> search_knowledge_base("分布式事务", top_k=3)
        [
          {"title": "2PC协议", "content": "...", "score": 0.95, "url": "..."},
          {"title": "TCC模式", "content": "...", "score": 0.89, "url": "..."}
        ]
    """
    pass
```

**3. 参数验证**
```python
@server.tool()
async def transfer_money(
    from_account: str,
    to_account: str,
    amount: float,
    currency: str = "CNY"
) -> str:
    """转账工具"""
    
    # 1. 参数验证
    errors = []
    
    if not from_account or not to_account:
        errors.append("账户不能为空")
    
    if from_account == to_account:
        errors.append("不能转账到相同账户")
    
    if amount <= 0:
        errors.append("金额必须大于0")
    
    if amount > 50000:
        errors.append("单笔转账不能超过50000元")
    
    if currency not in ["CNY", "USD", "EUR"]:
        errors.append(f"不支持的货币类型：{currency}")
    
    if errors:
        return json.dumps({"error": ", ".join(errors)})
    
    # 2. 业务逻辑
    try:
        result = await do_transfer(from_account, to_account, amount, currency)
        return json.dumps({
            "success": True,
            "transaction_id": result.id,
            "timestamp": result.timestamp
        })
    except InsufficientBalanceError:
        return json.dumps({"error": "余额不足"})
    except AccountNotFoundError as e:
        return json.dumps({"error": f"账户不存在：{e.account}"})
    except Exception as e:
        logger.error(f"转账失败：{e}")
        return json.dumps({"error": "系统错误，请稍后重试"})
```

**4. 错误处理**
```python
from enum import Enum

class ToolError(Enum):
    INVALID_PARAMS = "参数错误"
    RESOURCE_NOT_FOUND = "资源不存在"
    PERMISSION_DENIED = "权限不足"
    RATE_LIMIT_EXCEEDED = "请求过于频繁"
    INTERNAL_ERROR = "系统错误"

@server.tool()
async def safe_tool(param: str) -> str:
    """带完善错误处理的工具"""
    try:
        # 1. 参数验证
        if not param:
            return error_response(ToolError.INVALID_PARAMS, "param不能为空")
        
        # 2. 权限检查
        if not has_permission():
            return error_response(ToolError.PERMISSION_DENIED)
        
        # 3. 限流检查
        if not rate_limiter.allow():
            return error_response(ToolError.RATE_LIMIT_EXCEEDED)
        
        # 4. 执行业务逻辑
        result = await execute_logic(param)
        return success_response(result)
        
    except ResourceNotFoundError:
        return error_response(ToolError.RESOURCE_NOT_FOUND)
    except Exception as e:
        logger.exception(f"工具执行失败：{e}")
        return error_response(ToolError.INTERNAL_ERROR)

def error_response(error_type: ToolError, detail: str = None):
    """统一错误响应格式"""
    return json.dumps({
        "success": False,
        "error": {
            "type": error_type.name,
            "message": error_type.value,
            "detail": detail
        }
    })

def success_response(data):
    """统一成功响应格式"""
    return json.dumps({
        "success": True,
        "data": data
    })
```

---

## 4. Resource资源管理

### 4.1 Resource定义

**示例**：
```python
@server.resource("file://{path}")
async def read_file(path: str):
    """读取文件内容"""
    if not os.path.exists(path):
        raise ResourceNotFoundError(f"文件不存在：{path}")
    
    with open(path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    return {
        "uri": f"file://{path}",
        "name": os.path.basename(path),
        "description": f"文件：{path}",
        "mimeType": guess_mime_type(path),
        "text": content
    }

@server.resource("db://{table}/{id}")
async def get_record(table: str, id: int):
    """获取数据库记录"""
    record = await db.query(f"SELECT * FROM {table} WHERE id = ?", (id,))
    
    if not record:
        raise ResourceNotFoundError(f"记录不存在：{table}/{id}")
    
    return {
        "uri": f"db://{table}/{id}",
        "name": f"{table}记录#{id}",
        "mimeType": "application/json",
        "text": json.dumps(record)
    }
```

### 4.2 动态Resource列表

```python
@server.list_resources()
async def list_all_resources():
    """列出所有可用资源"""
    resources = []
    
    # 1. 列出文件资源
    for file in os.listdir("/data/docs"):
        resources.append({
            "uri": f"file:///data/docs/{file}",
            "name": file,
            "mimeType": guess_mime_type(file)
        })
    
    # 2. 列出数据库资源
    tables = await db.list_tables()
    for table in tables:
        count = await db.count(table)
        resources.append({
            "uri": f"db://{table}",
            "name": f"{table}表",
            "description": f"共{count}条记录",
            "mimeType": "application/json"
        })
    
    return resources
```

---

## 5. Prompt Templates

### 5.1 定义Prompt模板

```python
@server.prompt("code_review")
async def code_review_prompt(language: str, code: str):
    """代码审查提示模板"""
    return {
        "description": "代码审查模板",
        "messages": [
            {
                "role": "system",
                "content": {
                    "type": "text",
                    "text": f"你是一个专业的{language}代码审查专家。"
                }
            },
            {
                "role": "user",
                "content": {
                    "type": "text",
                    "text": f"""
请审查以下代码，关注：
1. 潜在的bug
2. 性能问题
3. 代码规范
4. 安全漏洞

代码：
```{language}
{code}
```

请提供详细的审查意见。
"""
                }
            }
        ]
    }

@server.prompt("sql_query_builder")
async def sql_query_builder_prompt(table: str, requirements: str):
    """SQL查询构建器模板"""
    # 获取表结构
    schema = await db.get_table_schema(table)
    
    return {
        "description": "SQL查询构建器",
        "messages": [
            {
                "role": "system",
                "content": {
                    "type": "text",
                    "text": "你是SQL专家，根据需求生成SQL查询。"
                }
            },
            {
                "role": "user",
                "content": {
                    "type": "text",
                    "text": f"""
表名：{table}
表结构：
{schema}

需求：{requirements}

请生成SQL查询语句。
"""
                }
            }
        ]
    }
```

### 1.4 MCP vs Function Calling

| 特性 | OpenAI Function Calling | MCP |
|------|------------------------|-----|
| **标准化** | OpenAI专有 | 开放协议，跨平台 |
| **工具定义** | JSON Schema | JSON Schema + 运行时发现 |
| **资源支持** | 无 | Resource + URI模式 |
| **双向通信** | 单向（LLM→工具） | 双向（支持Sampling） |
| **生态** | OpenAI生态 | 多厂商支持 |
| **传输** | HTTP API | stdio/SSE/WebSocket |

**何时用MCP？**
- 需要跨多个AI平台
- 需要资源访问和动态发现
- 需要复杂的双向交互
- 需要本地化部署

**何时用Function Calling？**
- 只使用OpenAI API
- 简单工具调用场景
- 快速原型开发

---

## 6. Transport传输层

### 6.1 stdio传输

**最常用的传输方式**，Client通过子进程启动Server：

```python
# Server端（stdio模式）
from mcp.server import Server
import mcp.server.stdio

server = Server("my-server")

@server.tool()
async def my_tool(param: str) -> str:
    return f"处理结果：{param}"

if __name__ == "__main__":
    mcp.server.stdio.run(server)  # 监听stdin/stdout
```

```python
# Client端启动Server
from mcp.client import MCPClient
from mcp.client.stdio import StdioClientTransport

transport = StdioClientTransport(
    command="python",
    args=["-m", "my_mcp_server"],
    env={"API_KEY": "xxx"}  # 传递环境变量
)

client = MCPClient()
await client.connect(transport)
```

**优点**：简单、安全（进程隔离）、无需网络
**缺点**：只能单Client连接

### 6.2 SSE传输（Server-Sent Events）

**适用于HTTP环境**，支持远程Server：

```python
# Server端（SSE模式）
from mcp.server import Server
import mcp.server.sse

server = Server("my-server")

# 使用Starlette或FastAPI
from starlette.applications import Starlette
from starlette.routing import Route

app = Starlette(
    routes=[
        Route("/sse", mcp.server.sse.handle_sse(server)),
    ]
)

# 启动
uvicorn.run(app, host="0.0.0.0", port=8000)
```

```typescript
// Client端（TypeScript）
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";

const transport = new SSEClientTransport(
    new URL("http://localhost:8000/sse")
);

const client = new Client({name: "my-client", version: "1.0"});
await client.connect(transport);
```

**优点**：支持远程、可部署为Web服务
**缺点**：单向流（Server→Client），需要额外POST端点

### 6.3 WebSocket传输

**双向实时通信**：

```python
# Server端（WebSocket）
from fastapi import FastAPI, WebSocket
import mcp.server.websocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    await mcp.server.websocket.handle_websocket(server, websocket)
```

**传输方式对比**：
| 传输 | 场景 | 多Client | 远程 | 双向 |
|------|------|----------|------|------|
| stdio | 本地CLI | ❌ | ❌ | ✅ |
| SSE | Web服务 | ✅ | ✅ | ⚠️ |
| WebSocket | 实时应用 | ✅ | ✅ | ✅ |

---

## 7. Sampling采样机制

### 7.1 什么是Sampling？

**Sampling**允许MCP Server请求Client（通常是LLM）生成内容，实现双向交互。

**典型场景**：
- Server需要LLM帮助理解用户意图
- Server需要LLM生成中间结果
- 实现Agent-in-the-loop模式

### 7.2 Sampling实现

```python
# Server端请求Client进行采样
@server.tool()
async def smart_search(query: str) -> str:
    """智能搜索：先让LLM优化查询"""
    
    # 1. 请求Client（LLM）优化查询
    sampling_result = await server.request_sampling(
        messages=[
            {
                "role": "user",
                "content": {
                    "type": "text",
                    "text": f"请将以下搜索词优化为更精确的查询：{query}"
                }
            }
        ],
        max_tokens=100
    )
    
    optimized_query = sampling_result.content[0].text
    
    # 2. 使用优化后的查询搜索
    results = await vector_db.search(optimized_query)
    
    return json.dumps({
        "original_query": query,
        "optimized_query": optimized_query,
        "results": results
    })
```

```python
# Client端需要声明支持Sampling
client = MCPClient(
    capabilities={
        "sampling": {}  # 声明支持Sampling
    }
)

# Client需要处理Sampling请求
@client.on_sampling_request
async def handle_sampling(request):
    # 调用LLM生成
    response = await llm.generate(request.messages)
    return {
        "content": [{"type": "text", "text": response}]
    }
```

---

## 8. 安全与权限控制

### 6.1 Hooks机制

**Before Hook（调用前拦截）**：
```python
from mcp.server import HookResult

@server.before_tool_call
async def check_permission(tool_name: str, arguments: dict, context: dict):
    """工具调用前的权限检查"""
    user_id = context.get("user_id")
    user_role = context.get("user_role")
    
    # 1. 敏感工具需要管理员权限
    sensitive_tools = ["delete_user", "transfer_money", "execute_sql"]
    if tool_name in sensitive_tools and user_role != "admin":
        return HookResult(
            allowed=False,
            reason=f"工具{tool_name}需要管理员权限"
        )
    
    # 2. 金额限制
    if tool_name == "transfer_money":
        amount = arguments.get("amount", 0)
        
        # 普通用户单笔限额10000
        if user_role == "user" and amount > 10000:
            return HookResult(
                allowed=False,
                reason="普通用户单笔转账不能超过10000元"
            )
        
        # VIP用户单笔限额50000
        if user_role == "vip" and amount > 50000:
            return HookResult(
                allowed=False,
                reason="VIP用户单笔转账不能超过50000元"
            )
    
    # 3. 速率限制
    rate_key = f"rate_limit:{user_id}:{tool_name}"
    if not await rate_limiter.allow(rate_key, limit=10, window=60):
        return HookResult(
            allowed=False,
            reason="请求过于频繁，请稍后再试"
        )
    
    return HookResult(allowed=True)

@server.after_tool_call
async def log_tool_call(
    tool_name: str,
    arguments: dict,
    result: any,
    context: dict,
    error: Exception = None
):
    """工具调用后记录日志"""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "user_id": context.get("user_id"),
        "tool": tool_name,
        "arguments": arguments,
        "success": error is None,
        "error": str(error) if error else None,
        "result_size": len(str(result)) if result else 0
    }
    
    # 写入审计日志
    await audit_log.write(log_entry)
    
    # 敏感操作告警
    if tool_name in ["delete_user", "transfer_money"] and not error:
        await send_alert(f"敏感操作：{tool_name}", log_entry)
```

### 6.2 参数过滤与脱敏

```python
@server.before_tool_call
async def sanitize_params(tool_name: str, arguments: dict, context: dict):
    """参数过滤与验证"""
    
    # 1. SQL注入防护
    if tool_name == "execute_sql":
        sql = arguments.get("sql", "")
        
        # 禁止危险关键词
        dangerous_keywords = ["DROP", "TRUNCATE", "DELETE", "UPDATE"]
        if any(keyword in sql.upper() for keyword in dangerous_keywords):
            return HookResult(
                allowed=False,
                reason="SQL语句包含危险关键词"
            )
        
        # 只允许SELECT
        if not sql.strip().upper().startswith("SELECT"):
            return HookResult(
                allowed=False,
                reason="只允许执行SELECT查询"
            )
    
    # 2. 路径遍历防护
    if tool_name == "read_file":
        path = arguments.get("path", "")
        
        # 规范化路径
        normalized = os.path.normpath(path)
        
        # 禁止访问父目录
        if ".." in normalized:
            return HookResult(
                allowed=False,
                reason="禁止访问父目录"
            )
        
        # 只允许访问指定目录
        allowed_dirs = ["/data/docs", "/data/public"]
        if not any(normalized.startswith(d) for d in allowed_dirs):
            return HookResult(
                allowed=False,
                reason="只能访问允许的目录"
            )
    
    return HookResult(allowed=True)

@server.after_tool_call
async def sanitize_result(tool_name: str, result: any, context: dict):
    """结果脱敏"""
    if tool_name == "get_user_info":
        # 脱敏敏感字段
        data = json.loads(result)
        if "phone" in data:
            data["phone"] = mask_phone(data["phone"])  # 138****5678
        if "id_card" in data:
            data["id_card"] = mask_id_card(data["id_card"])  # 110***********1234
        return json.dumps(data)
    
    return result

def mask_phone(phone: str) -> str:
    """手机号脱敏"""
    if len(phone) == 11:
        return f"{phone[:3]}****{phone[7:]}"
    return phone

def mask_id_card(id_card: str) -> str:
    """身份证脱敏"""
    if len(id_card) == 18:
        return f"{id_card[:3]}***********{id_card[-4:]}"
    return id_card
```

---

## 9. 性能优化

### 7.1 缓存

```python
from functools import lru_cache
import hashlib

class MCPServer:
    def __init__(self):
        self.cache = {}
        self.cache_ttl = 300  # 5分钟
    
    @server.tool()
    async def cached_search(self, query: str, top_k: int = 5) -> str:
        """带缓存的搜索"""
        # 生成缓存键
        cache_key = hashlib.md5(
            f"{query}:{top_k}".encode()
        ).hexdigest()
        
        # 检查缓存
        if cache_key in self.cache:
            entry = self.cache[cache_key]
            if time.time() - entry['timestamp'] < self.cache_ttl:
                return entry['result']
        
        # 执行搜索
        result = await vector_db.search(query, top_k)
        
        # 写入缓存
        self.cache[cache_key] = {
            'result': result,
            'timestamp': time.time()
        }
        
        return result
```

### 7.2 批处理

```python
@server.tool()
async def batch_search(queries: list[str], top_k: int = 5) -> str:
    """批量搜索（一次调用多个查询）"""
    # 并发执行
    tasks = [vector_db.search(q, top_k) for q in queries]
    results = await asyncio.gather(*tasks)
    
    return json.dumps({
        "results": [
            {"query": q, "results": r}
            for q, r in zip(queries, results)
        ]
    })
```

### 7.3 连接池

```python
from asyncio import Semaphore

class MCPServer:
    def __init__(self, max_concurrent=10):
        self.semaphore = Semaphore(max_concurrent)
    
    @server.tool()
    async def rate_limited_tool(self, param: str) -> str:
        """限制并发数的工具"""
        async with self.semaphore:
            # 最多同时执行10个请求
            result = await expensive_operation(param)
            return result
```

---

## 10. 调试与排障

### 8.1 日志

```python
import logging

# 配置日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('mcp_server.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger('mcp_server')

@server.tool()
async def debug_tool(param: str) -> str:
    """带详细日志的工具"""
    logger.info(f"工具调用开始：param={param}")
    
    try:
        result = await process(param)
        logger.info(f"工具调用成功：result_size={len(result)}")
        return result
    except Exception as e:
        logger.error(f"工具调用失败：{e}", exc_info=True)
        raise
```

### 8.2 MCP Inspector

**使用Inspector调试**：
```bash
# 安装Inspector
npm install -g @modelcontextprotocol/inspector

# 启动Inspector
mcp-inspector python -m mcp_server

# 浏览器访问 http://localhost:5173
# 可以：
# - 查看所有Tools、Resources、Prompts
# - 测试工具调用
# - 查看请求/响应
# - 查看日志
```

### 8.3 常见问题排查

**问题1：连接超时**
```bash
# 检查Server是否启动
ps aux | grep mcp_server

# 检查端口
lsof -i :5000

# 查看日志
tail -f mcp_server.log
```

**问题2：工具调用失败**
```python
# 启用详细日志
export MCP_DEBUG=1
python -m mcp_server

# 输出会显示完整的JSON-RPC消息：
# → {"jsonrpc": "2.0", "method": "tools/call", ...}
# ← {"jsonrpc": "2.0", "result": {...}}
```

**问题3：参数验证错误**
```python
# 测试Schema验证
from jsonschema import validate, ValidationError

schema = {
    "type": "object",
    "properties": {
        "query": {"type": "string"},
        "top_k": {"type": "integer", "minimum": 1, "maximum": 20}
    },
    "required": ["query"]
}

# 测试有效参数
validate({"query": "test", "top_k": 5}, schema)  # OK

# 测试无效参数
try:
    validate({"query": "test", "top_k": 100}, schema)
except ValidationError as e:
    print(f"验证失败：{e.message}")
    # "100 is greater than the maximum of 20"
```

---

## 11. 实战案例

### 案例1：知识库MCP Server

```python
from mcp.server import Server
import faiss
import numpy as np

class KnowledgeBaseMCPServer:
    def __init__(self):
        self.server = Server("knowledge-base")
        self.index = faiss.IndexFlatL2(768)  # 向量维度768
        self.documents = []
        self.setup_tools()
    
    def setup_tools(self):
        @self.server.tool()
        async def search(query: str, top_k: int = 5) -> str:
            """搜索知识库"""
            # 1. 查询向量化
            query_vec = await self.embed(query)
            
            # 2. 向量检索
            distances, indices = self.index.search(
                np.array([query_vec]), top_k
            )
            
            # 3. 返回结果
            results = [
                {
                    "title": self.documents[i]["title"],
                    "content": self.documents[i]["content"][:200],
                    "score": 1 / (1 + distances[0][j])
                }
                for j, i in enumerate(indices[0])
            ]
            
            return json.dumps(results, ensure_ascii=False)
        
        @self.server.tool()
        async def add_document(title: str, content: str) -> str:
            """添加文档"""
            # 1. 文档向量化
            vec = await self.embed(content)
            
            # 2. 添加到索引
            self.index.add(np.array([vec]))
            
            # 3. 保存文档
            doc_id = len(self.documents)
            self.documents.append({
                "id": doc_id,
                "title": title,
                "content": content
            })
            
            return json.dumps({"success": True, "doc_id": doc_id})
        
        @self.server.resource("kb://stats")
        async def get_stats():
            """获取统计信息"""
            return {
                "uri": "kb://stats",
                "name": "知识库统计",
                "mimeType": "application/json",
                "text": json.dumps({
                    "total_documents": len(self.documents),
                    "index_size": self.index.ntotal
                })
            }
    
    async def embed(self, text: str) -> list[float]:
        """文本向量化"""
        # 调用Embedding API
        response = await openai.Embedding.create(
            model="text-embedding-ada-002",
            input=text
        )
        return response['data'][0]['embedding']
    
    def run(self):
        """启动Server"""
        import mcp.server.stdio
        mcp.server.stdio.run(self.server)

if __name__ == "__main__":
    server = KnowledgeBaseMCPServer()
    server.run()
```

### 案例2：数据库MCP Server

```python
class DatabaseMCPServer:
    def __init__(self, db_url: str):
        self.server = Server("database")
        self.db = create_engine(db_url)
        self.setup_tools()
    
    def setup_tools(self):
        @self.server.tool()
        async def query(sql: str) -> str:
            """执行SQL查询（只读）"""
            # 安全检查
            if not sql.strip().upper().startswith("SELECT"):
                return json.dumps({"error": "只允许SELECT查询"})
            
            # 执行查询
            with self.db.connect() as conn:
                result = conn.execute(text(sql))
                rows = [dict(row) for row in result]
            
            return json.dumps(rows)
        
        @self.server.tool()
        async def get_table_schema(table: str) -> str:
            """获取表结构"""
            sql = f"""
            SELECT column_name, data_type, is_nullable
            FROM information_schema.columns
            WHERE table_name = '{table}'
            """
            
            with self.db.connect() as conn:
                result = conn.execute(text(sql))
                schema = [dict(row) for row in result]
            
            return json.dumps(schema)
        
        @self.server.list_resources()
        async def list_tables():
            """列出所有表"""
            sql = "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'"
            
            with self.db.connect() as conn:
                result = conn.execute(text(sql))
                tables = [row[0] for row in result]
            
            return [
                {
                    "uri": f"db://{table}",
                    "name": table,
                    "description": f"表：{table}"
                }
                for table in tables
            ]
```

---

**总结**

MCP核心要点：
1. **协议标准**：JSON-RPC 2.0、Tool/Resource/Prompt三大概念
2. **Server开发**：Python/TypeScript SDK、工具定义、资源管理
3. **传输层**：stdio（本地）、SSE（Web）、WebSocket（实时）
4. **安全控制**：Hooks机制、权限检查、参数验证、结果脱敏
5. **性能优化**：缓存、批处理、连接池
6. **调试排障**：日志、Inspector、Schema验证


---

## 12. 面试题自查

### Q1: MCP是什么？解决了什么问题？

**答案**：
MCP（Model Context Protocol）是由Anthropic提出的开放协议，用于标准化AI应用与外部工具/数据源的集成。

**解决的问题**：
1. **碎片化**：每个AI应用都要单独对接各种工具，重复开发
2. **标准缺失**：Function Calling是厂商专有格式，不通用
3. **能力受限**：传统方式只能调用工具，无法动态发现资源

**MCP的价值**：
- 一次开发，多处使用（工具开发者）
- 统一接口，快速集成（AI应用开发者）
- 开放生态，避免锁定

---

### Q2: MCP的三大核心概念是什么？

**答案**：

| 概念 | 说明 | 示例 |
|------|------|------|
| **Tool** | 可执行的功能，LLM可以调用 | 搜索、计算、发送邮件 |
| **Resource** | 可读取的数据资源，URI寻址 | 文件、数据库记录、配置 |
| **Prompt** | 预定义的提示模板 | 代码审查模板、SQL生成模板 |

**区别**：
- Tool是"动词"——执行操作
- Resource是"名词"——获取数据
- Prompt是"模板"——标准化交互

---

### Q3: MCP与OpenAI Function Calling有什么区别？

**答案**：

| 维度 | Function Calling | MCP |
|------|------------------|-----|
| 标准化 | OpenAI专有 | 开放协议 |
| 工具发现 | 预定义 | 运行时动态发现 |
| 资源访问 | 不支持 | Resource + URI |
| 双向通信 | 单向 | 支持Sampling |
| 传输方式 | HTTP API | stdio/SSE/WebSocket |
| 多Server | 不支持 | 支持 |

**选择建议**：
- 简单场景、只用OpenAI → Function Calling
- 跨平台、复杂集成、本地部署 → MCP

---

### Q4: MCP支持哪些传输方式？各有什么特点？

**答案**：

| 传输 | 原理 | 适用场景 | 多Client | 远程 |
|------|------|----------|----------|------|
| **stdio** | 子进程stdin/stdout | 本地CLI工具、桌面应用 | ❌ | ❌ |
| **SSE** | HTTP Server-Sent Events | Web服务部署 | ✅ | ✅ |
| **WebSocket** | 全双工连接 | 实时应用、长连接 | ✅ | ✅ |

**选择原则**：
- 本地单用户 → stdio（最简单）
- 远程/多用户 → SSE或WebSocket
- 需要实时双向 → WebSocket

---

### Q5: 如何设计一个好的MCP Tool？

**答案**：

**1. 单一职责**
```python
# ❌ 不好
async def database_op(action, table, data): ...

# ✅ 好
async def insert_record(table, data): ...
async def update_record(table, id, data): ...
```

**2. 清晰描述**
- 说明功能、参数、返回值
- 提供使用示例
- 注明适用场景

**3. 严格验证**
- 必填参数检查
- 类型和范围验证
- 业务规则校验

**4. 优雅错误处理**
- 结构化错误响应
- 区分错误类型
- 提供修复建议

**5. 幂等性设计**
- 重复调用不产生副作用
- 或提供幂等键

---

### Q6: MCP的Sampling机制是什么？有什么用？

**答案**：
**Sampling**允许Server请求Client（LLM）生成内容，实现双向交互。

**普通调用**：Client → Server（调用工具）
**Sampling**：Server → Client（请求LLM生成）

**应用场景**：
1. **查询优化**：Server让LLM重写用户查询
2. **结果解释**：Server让LLM解释查询结果
3. **Agent-in-loop**：工具执行过程中需要LLM决策

**示例**：
```python
@server.tool()
async def smart_search(query: str):
    # 请求LLM优化查询
    result = await server.request_sampling(
        messages=[{"role": "user", "content": f"优化查询：{query}"}]
    )
    optimized = result.content[0].text
    return await search(optimized)
```

---

### Q7: 如何保证MCP工具调用的安全性？

**答案**：

**1. Hooks机制**
```python
@server.before_tool_call
async def check_permission(tool_name, args, context):
    # 权限检查
    if tool_name in sensitive_tools and not is_admin(context):
        return HookResult(allowed=False, reason="需要管理员权限")
```

**2. 参数验证**
- JSON Schema验证
- 业务规则校验
- 防注入（SQL、路径遍历）

**3. 速率限制**
```python
if not rate_limiter.allow(user_id, tool_name):
    return error("请求过于频繁")
```

**4. 结果脱敏**
- 敏感字段脱敏（手机号、身份证）
- 日志脱敏

**5. 审计日志**
- 记录所有调用
- 敏感操作告警

---

### Q8: MCP Server如何做性能优化？

**答案**：

**1. 缓存**
```python
@lru_cache(maxsize=100)
async def cached_search(query, top_k):
    return await vector_db.search(query, top_k)
```

**2. 批处理**
```python
# 一次调用处理多个查询
async def batch_search(queries: list[str]):
    tasks = [search(q) for q in queries]
    return await asyncio.gather(*tasks)
```

**3. 连接池**
- 数据库连接池
- HTTP连接池

**4. 并发控制**
```python
semaphore = Semaphore(10)  # 最多10并发
async with semaphore:
    await expensive_operation()
```

**5. 异步IO**
- 所有IO操作使用async/await
- 避免阻塞调用

---

### Q9: JSON Schema在MCP中如何使用？

**答案**：
MCP使用JSON Schema定义Tool的输入参数。

**常用类型**：
```json
{
  "string": {"type": "string", "minLength": 1, "maxLength": 100},
  "number": {"type": "number", "minimum": 0, "maximum": 1000},
  "integer": {"type": "integer", "multipleOf": 10},
  "boolean": {"type": "boolean"},
  "array": {"type": "array", "items": {...}, "minItems": 1},
  "object": {"type": "object", "properties": {...}},
  "enum": {"type": "string", "enum": ["A", "B", "C"]}
}
```

**format扩展**：
```json
{
  "email": {"type": "string", "format": "email"},
  "date": {"type": "string", "format": "date"},
  "uri": {"type": "string", "format": "uri"},
  "uuid": {"type": "string", "format": "uuid"}
}
```

**验证示例**：
```python
from jsonschema import validate
validate({"query": "test", "top_k": 5}, schema)
```

---

### Q10: 如何调试MCP Server？

**答案**：

**1. MCP Inspector**
```bash
npm install -g @modelcontextprotocol/inspector
mcp-inspector python -m my_server
# 浏览器访问 http://localhost:5173
```

**2. 详细日志**
```python
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('mcp')

@server.tool()
async def my_tool(param):
    logger.debug(f"调用参数：{param}")
    result = await process(param)
    logger.debug(f"返回结果：{result}")
    return result
```

**3. 环境变量**
```bash
export MCP_DEBUG=1  # 显示完整JSON-RPC消息
```

**4. Schema验证测试**
```python
from jsonschema import validate, ValidationError
try:
    validate(params, schema)
except ValidationError as e:
    print(f"参数错误：{e.message}")
```

---

### Q11: MCP的Resource和Tool有什么区别？何时用哪个？

**答案**：

| 维度 | Tool | Resource |
|------|------|----------|
| 语义 | 动作/操作 | 数据/内容 |
| 调用方式 | 传参数执行 | URI寻址读取 |
| 幂等性 | 不保证 | 天然幂等 |
| 缓存 | 通常不缓存 | 可缓存 |

**使用Tool的场景**：
- 需要执行操作（创建、修改、删除）
- 需要复杂参数
- 结果每次可能不同

**使用Resource的场景**：
- 读取静态/半静态数据
- 数据有明确的URI标识
- 需要支持缓存

**示例**：
```python
# Tool：执行搜索（需要参数，结果变化）
@server.tool()
async def search(query: str, top_k: int): ...

# Resource：读取配置（URI标识，相对稳定）
@server.resource("config://database")
async def get_db_config(): ...
```

---

### Q12: 如何实现多个MCP Server的协同？

**答案**：

**架构设计**：
```
[AI Application]
       ↓
  [MCP Client]
       ↓
 ┌─────┴─────┐
 ↓           ↓
[Server A]  [Server B]
知识库       数据库
```

**Client连接多Server**：
```python
client = MCPClient()

# 连接多个Server
await client.connect("knowledge", command="python", args=["-m", "kb_server"])
await client.connect("database", command="python", args=["-m", "db_server"])

# 调用不同Server的工具
kb_result = await client.call_tool("knowledge", "search", {"query": "xxx"})
db_result = await client.call_tool("database", "query", {"sql": "..."})
```

**工具路由**：
- 按前缀区分：`kb_search`, `db_query`
- 按Server名区分

**数据流转**：
- Server A的结果可以作为Server B的输入
- Client负责编排

---

### Q13: MCP的错误处理最佳实践？

**答案**：

**1. 错误分类**
```python
class ToolError(Enum):
    INVALID_PARAMS = "参数错误"
    NOT_FOUND = "资源不存在"
    PERMISSION_DENIED = "权限不足"
    RATE_LIMITED = "请求过频"
    INTERNAL_ERROR = "系统错误"
```

**2. 结构化响应**
```python
def error_response(error_type, detail=None):
    return json.dumps({
        "success": False,
        "error": {
            "type": error_type.name,
            "message": error_type.value,
            "detail": detail
        }
    })
```

**3. 分层处理**
```python
@server.tool()
async def my_tool(param):
    # 1. 参数验证
    if not param:
        return error_response(ToolError.INVALID_PARAMS, "param不能为空")
    
    # 2. 业务逻辑（try-catch）
    try:
        result = await process(param)
        return success_response(result)
    except NotFoundException:
        return error_response(ToolError.NOT_FOUND)
    except Exception as e:
        logger.exception(e)
        return error_response(ToolError.INTERNAL_ERROR)
```

**4. 提供修复建议**
```json
{
  "error": {
    "type": "INVALID_PARAMS",
    "message": "参数top_k超出范围",
    "suggestion": "top_k应在1-20之间"
  }
}
```

---

### Q14: MCP在生产环境部署需要注意什么？

**答案**：

**1. 可观测性**
- 接入监控（Prometheus/Grafana）
- 分布式追踪（Jaeger/Zipkin）
- 日志聚合（ELK/Loki）

**2. 高可用**
- 多实例部署
- 负载均衡（SSE/WebSocket模式）
- 健康检查

**3. 安全加固**
- TLS加密（远程传输）
- 认证鉴权
- 网络隔离

**4. 资源限制**
- 请求超时
- 内存限制
- 并发控制

**5. 版本管理**
- 工具版本化
- 向后兼容
- 灰度发布

---

### Q15: 如何测试MCP Server？

**答案**：

**1. 单元测试**
```python
import pytest

@pytest.mark.asyncio
async def test_search_tool():
    result = await search("分布式事务", top_k=3)
    data = json.loads(result)
    
    assert "results" in data
    assert len(data["results"]) <= 3
```

**2. 集成测试**
```python
@pytest.mark.asyncio
async def test_client_integration():
    client = MCPClient()
    await client.connect("test", command="python", args=["-m", "server"])
    
    # 测试工具列表
    tools = await client.list_tools()
    assert "search" in [t.name for t in tools]
    
    # 测试工具调用
    result = await client.call_tool("search", {"query": "test"})
    assert result.content[0].type == "text"
```

**3. Schema验证测试**
```python
def test_tool_schema():
    schema = get_tool_schema("search")
    
    # 有效参数
    validate({"query": "test"}, schema)  # 通过
    
    # 无效参数
    with pytest.raises(ValidationError):
        validate({"query": ""}, schema)  # minLength=1
```

**4. Mock测试**
```python
@pytest.fixture
def mock_db():
    with patch("server.vector_db") as mock:
        mock.search.return_value = [{"title": "test"}]
        yield mock

async def test_with_mock(mock_db):
    result = await search("query", 5)
    mock_db.search.assert_called_once_with("query", 5)
```