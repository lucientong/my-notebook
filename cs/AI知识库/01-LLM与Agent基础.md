# LLM 与 Agent 基础

---

## 📑 目录

1. [LLM核心原理](#1-llm核心原理)
2. [Prompt Engineering](#2-prompt-engineering)
3. [Agent架构设计](#3-agent架构设计)
4. [Tool Calling工具调用](#4-tool-calling工具调用)
5. [RAG检索增强生成](#5-rag检索增强生成)
6. [Memory记忆机制](#6-memory记忆机制)
7. [幻觉问题与解决](#7-幻觉问题与解决)
8. [常见面试题](#8-常见面试题)
9. [实战案例](#9-实战案例)

---

## 1. LLM核心原理

### 1.1 Transformer架构

**核心创新**：Self-Attention机制

**组件**：
```
Input Embedding → Positional Encoding
    ↓
Multi-Head Attention
    ↓
Add & Norm
    ↓
Feed Forward Network
    ↓
Add & Norm
    ↓
Output
```

**Attention公式**：
```
Attention(Q, K, V) = softmax(QK^T / √d_k) * V

其中：
- Q (Query)：查询向量
- K (Key)：键向量
- V (Value)：值向量
- d_k：键向量维度
```

**为什么除以√d_k？**
- 防止点积过大导致softmax梯度消失
- 示例：d_k=64，√64=8，缩放8倍

**Multi-Head Attention**：
```python
class MultiHeadAttention:
    def __init__(self, d_model=512, num_heads=8):
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = Linear(d_model, d_model)
        self.W_k = Linear(d_model, d_model)
        self.W_v = Linear(d_model, d_model)
        self.W_o = Linear(d_model, d_model)
    
    def forward(self, x):
        batch_size = x.size(0)
        
        # 线性变换
        Q = self.W_q(x)  # (batch, seq_len, d_model)
        K = self.W_k(x)
        V = self.W_v(x)
        
        # 分割成多头
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # 计算attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        attn = F.softmax(scores, dim=-1)
        output = torch.matmul(attn, V)
        
        # 拼接多头
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.num_heads * self.d_k)
        
        # 输出线性变换
        return self.W_o(output)
```

### 1.2 Tokenization分词

**常见算法**：
- **BPE (Byte-Pair Encoding)**：GPT系列
- **WordPiece**：BERT
- **SentencePiece**：T5、LLaMA

**BPE示例**：
```python
# 原始文本
text = "lower lowest"

# 初始词表
vocab = ['l', 'o', 'w', 'e', 'r', 's', 't']

# 迭代合并高频对
# 第1轮：'lo' 出现2次 → vocab.append('lo')
# 第2轮：'low' 出现2次 → vocab.append('low')
# 第3轮：'lowe' 出现2次 → vocab.append('lowe')
# ...

# 最终词表
vocab = ['l', 'o', 'w', 'e', 'r', 's', 't', 'lo', 'low', 'lower', 'lowest', ...]

# 编码
encode("lower") → ['lower']
encode("lowest") → ['lowest']
encode("slow") → ['s', 'low']
```

**Python实现**：
```python
import tiktoken

# GPT-4使用cl100k_base
encoder = tiktoken.get_encoding("cl100k_base")

text = "Hello, world!"
tokens = encoder.encode(text)
print(tokens)  # [9906, 11, 1917, 0]

decoded = encoder.decode(tokens)
print(decoded)  # "Hello, world!"

# Token数量
print(f"Token count: {len(tokens)}")  # 4
```

**Token限制**：
| 模型 | 上下文窗口 | 输入 | 输出 |
|------|------------|------|------|
| GPT-4 | 8K/32K/128K | 支持 | 4K |
| GPT-4 Turbo | 128K | 支持 | 4K |
| Claude 3 | 200K | 支持 | 4K |
| Gemini 1.5 | 1M | 支持 | 8K |

### 1.3 LLM训练流程

**三阶段训练**：

**1. 预训练（Pre-training）**
```
目标：学习语言基础能力
数据：海量无标注文本（TB级）
任务：Next Token Prediction
损失函数：Cross-Entropy Loss

L = -∑ log P(token_i | token_1, ..., token_{i-1})
```

**2. 有监督微调（SFT, Supervised Fine-Tuning）**
```
目标：学习遵循指令
数据：高质量标注数据（10K-100K条）
格式：
  User: 请帮我写一首诗
  Assistant: 好的，这是一首关于...的诗：...
```

**3. 人类反馈强化学习（RLHF）**
```
目标：对齐人类偏好
步骤：
  1. 收集人类偏好数据（A vs B，哪个更好？）
  2. 训练奖励模型（Reward Model）
  3. 使用PPO算法优化策略模型
```

**示例代码（简化版）**：
```python
# 预训练
for batch in dataset:
    input_ids, labels = batch
    logits = model(input_ids)
    loss = cross_entropy(logits, labels)
    loss.backward()
    optimizer.step()

# SFT
for batch in instruction_dataset:
    prompt, response = batch
    input_ids = tokenizer.encode(f"{prompt}\n{response}")
    logits = model(input_ids)
    loss = cross_entropy(logits, labels)
    loss.backward()
    optimizer.step()

# RLHF
for batch in dataset:
    prompt = batch['prompt']
    response = policy_model.generate(prompt)
    reward = reward_model(prompt, response)
    ppo_loss = compute_ppo_loss(response, reward)
    ppo_loss.backward()
    optimizer.step()
```

---

## 2. Prompt Engineering

### 2.1 基础技巧

**1. Zero-Shot（零样本）**
```
Prompt:
"""
你是一个专业的客服，请回答用户的问题。

用户：我的订单什么时候到？
"""

回答：请提供您的订单号，我帮您查询物流信息。
```

**2. Few-Shot（少样本）**
```
Prompt:
"""
将以下句子分类为正面或负面：

示例1：这个产品很棒！ → 正面
示例2：质量太差了。 → 负面
示例3：还不错，值得购买。 → 正面

请分类：这个手机电池续航不行。
"""

回答：负面
```

**3. Chain-of-Thought（思维链，CoT）**
```
Prompt:
"""
请一步步思考：

问题：小明有5个苹果，吃了2个，又买了3个，现在有几个？

思考过程：
1. 初始有5个苹果
2. 吃了2个，剩余：5 - 2 = 3个
3. 又买了3个，总共：3 + 3 = 6个

答案：6个苹果
"""
```

**4. ReAct (Reasoning + Acting)**
```
Prompt:
"""
任务：查询北京今天的天气并告诉我是否适合出门跑步

思考：我需要先查询北京的天气情况
行动：call_tool("get_weather", {"city": "北京", "date": "today"})
观察：今天北京晴天，温度15-25度，空气质量优

思考：天气很好，适合户外运动
回答：北京今天天气很好，晴天15-25度，空气质量优，非常适合出门跑步！
"""
```

### 2.2 高级技巧

**1. Self-Consistency（自一致性）**
```python
# 生成多个推理路径，选择最一致的答案
def self_consistency(prompt, n=5):
    answers = []
    for i in range(n):
        response = llm.generate(prompt, temperature=0.7)
        answer = extract_answer(response)
        answers.append(answer)
    
    # 投票选择最频繁的答案
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]

# 示例
prompt = "如果小明有5个苹果，吃了2个，又买了3个，现在有几个？请一步步思考。"
answer = self_consistency(prompt, n=5)
# 输出：6（多次推理都得到6，置信度高）
```

**2. Tree of Thoughts（思维树）**
```
问题：如何将24个数字排列成4×6的矩阵，使得每行每列的和相等？

思维树：
               [根节点：问题]
                     |
         ┌───────────┼───────────┐
         |           |           |
    [思路1]      [思路2]      [思路3]
    先确定和      数学推导      试错法
         |           |           |
    [细化1-1]   [细化2-1]   [细化3-1]
    每行和60     方程组      暴力枚举
         |           |           |
    [评估]      [评估]      [评估]
    可行✓       可行✓       不可行✗

最优路径：思路2 → 数学推导 → 方程组
```

**Python实现**：
```python
class ThoughtNode:
    def __init__(self, thought, parent=None):
        self.thought = thought
        self.parent = parent
        self.children = []
        self.value = 0  # 评估分数
    
    def expand(self, llm):
        """扩展思路"""
        prompt = f"基于思路：{self.thought}\n请生成3个可能的下一步思考："
        responses = llm.generate(prompt, n=3)
        for resp in responses:
            child = ThoughtNode(resp, parent=self)
            self.children.append(child)
    
    def evaluate(self, llm):
        """评估思路"""
        prompt = f"评估以下思路的可行性（0-10分）：{self.thought}"
        score = llm.generate(prompt)
        self.value = float(score)
    
    def best_path(self):
        """返回最佳路径"""
        if not self.children:
            return [self]
        
        best_child = max(self.children, key=lambda c: c.value)
        return [self] + best_child.best_path()

# 使用
root = ThoughtNode("如何解决问题X？")
root.expand(llm)
for child in root.children:
    child.evaluate(llm)
    child.expand(llm)

best_path = root.best_path()
print("最佳思考路径：", [node.thought for node in best_path])
```

**3. Program-Aided Language Models (PAL)**
```python
# LLM生成代码，由代码执行器执行
prompt = """
问题：小明有5个苹果，吃了2个，又买了3个，现在有几个？

请用Python代码计算：
"""

response = llm.generate(prompt)
# 输出：
# apples = 5
# apples = apples - 2
# apples = apples + 3
# print(apples)

# 执行代码
exec(response)  # 输出：6
```

### 2.3 Prompt模板设计

**最佳实践**：
```python
# 结构化Prompt模板
template = """
# 角色定义
你是一个{role}，擅长{skills}。

# 任务描述
{task_description}

# 输入数据
{input_data}

# 输出格式
请按照以下JSON格式输出：
{output_format}

# 约束条件
- {constraint_1}
- {constraint_2}

# 示例
输入：{example_input}
输出：{example_output}

现在请处理以下输入：
{user_input}
"""

# 填充模板
prompt = template.format(
    role="金融风控专家",
    skills="信用评估、欺诈检测",
    task_description="分析用户的信用风险",
    input_data="用户ID、交易历史、个人信息",
    output_format='{"risk_level": "低/中/高", "score": 0-100, "reason": "..."}',
    constraint_1="评分必须在0-100之间",
    constraint_2="必须提供具体的风险原因",
    example_input="...",
    example_output="...",
    user_input="用户ID: 12345, ..."
)
```

---

## 3. Agent架构设计

### 3.1 Agent核心循环

**经典架构**：
```
感知 (Perceive) → 规划 (Plan) → 执行 (Act) → 观察 (Observe) → 反思 (Reflect)
         ↑                                                               ↓
         └───────────────────────────────────────────────────────────────┘
```

**Python实现**：
```python
class Agent:
    def __init__(self, llm, tools, memory):
        self.llm = llm
        self.tools = {tool.name: tool for tool in tools}
        self.memory = memory
        self.max_iterations = 10
    
    def run(self, task):
        """运行Agent"""
        self.memory.add("task", task)
        
        for i in range(self.max_iterations):
            # 1. 感知：获取当前状态
            state = self.perceive()
            
            # 2. 规划：决定下一步
            plan = self.plan(state)
            
            # 3. 执行：调用工具
            result = self.act(plan)
            
            # 4. 观察：记录结果
            observation = self.observe(result)
            
            # 5. 反思：是否完成
            if self.is_done(observation):
                return self.get_final_answer()
        
        return "超过最大迭代次数"
    
    def perceive(self):
        """感知：构建当前状态"""
        return {
            "task": self.memory.get("task"),
            "history": self.memory.get_recent(5),
            "available_tools": list(self.tools.keys())
        }
    
    def plan(self, state):
        """规划：LLM决定下一步行动"""
        prompt = f"""
        任务：{state['task']}
        历史：{state['history']}
        可用工具：{state['available_tools']}
        
        请使用ReAct格式决定下一步行动：
        思考：...
        行动：tool_name(args)
        """
        
        response = self.llm.generate(prompt)
        return self.parse_action(response)
    
    def act(self, plan):
        """执行：调用工具"""
        tool_name = plan['tool']
        args = plan['args']
        
        if tool_name not in self.tools:
            return {"error": f"工具{tool_name}不存在"}
        
        tool = self.tools[tool_name]
        try:
            result = tool.run(**args)
            return {"success": True, "result": result}
        except Exception as e:
            return {"success": False, "error": str(e)}
    
    def observe(self, result):
        """观察：记录结果"""
        self.memory.add("observation", result)
        return result
    
    def is_done(self, observation):
        """判断是否完成"""
        prompt = f"""
        观察结果：{observation}
        
        任务是否已完成？回答：是/否
        """
        response = self.llm.generate(prompt)
        return "是" in response
    
    def get_final_answer(self):
        """生成最终答案"""
        prompt = f"""
        任务：{self.memory.get('task')}
        历史记录：{self.memory.get_all()}
        
        请总结最终答案：
        """
        return self.llm.generate(prompt)
```

### 3.2 Agent类型

**1. ReAct Agent**（推理+行动）
```python
from langchain.agents import create_react_agent, AgentExecutor
from langchain_openai import ChatOpenAI
from langchain.tools import Tool

# 定义工具
def search_tool(query):
    return f"搜索结果：{query}的相关信息..."

def calculator_tool(expression):
    return eval(expression)

tools = [
    Tool(name="Search", func=search_tool, description="搜索信息"),
    Tool(name="Calculator", func=calculator_tool, description="计算表达式")
]

# 创建Agent
llm = ChatOpenAI(model="gpt-4")
agent = create_react_agent(llm, tools)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 执行任务
result = agent_executor.invoke({
    "input": "比特币价格是多少？如果买1000美元能买多少个？"
})
```

**2. Plan-and-Execute Agent**（先规划，再执行）
```python
class PlanAndExecuteAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = tools
    
    def run(self, task):
        # 1. 制定计划
        plan = self.make_plan(task)
        print(f"计划：{plan}")
        
        # 2. 执行计划
        results = []
        for step in plan:
            result = self.execute_step(step)
            results.append(result)
        
        # 3. 总结答案
        return self.summarize(results)
    
    def make_plan(self, task):
        """制定详细计划"""
        prompt = f"""
        任务：{task}
        
        请制定详细的执行计划（JSON格式）：
        [
          {{"step": 1, "action": "search", "args": {{"query": "..."}}}},
          {{"step": 2, "action": "calculate", "args": {{"expression": "..."}}}},
          ...
        ]
        """
        response = self.llm.generate(prompt)
        return json.loads(response)
    
    def execute_step(self, step):
        """执行单个步骤"""
        tool = self.tools[step['action']]
        return tool.run(**step['args'])
```

### 3.3 Multi-Agent系统

**协作模式**：
```python
class MultiAgentSystem:
    def __init__(self, agents):
        self.agents = agents  # {"researcher": Agent1, "writer": Agent2, ...}
    
    def run(self, task):
        # 1. Researcher Agent收集信息
        research_result = self.agents['researcher'].run(
            f"研究以下主题：{task}"
        )
        
        # 2. Analyst Agent分析数据
        analysis_result = self.agents['analyst'].run(
            f"分析以下数据：{research_result}"
        )
        
        # 3. Writer Agent撰写报告
        report = self.agents['writer'].run(
            f"根据以下分析撰写报告：{analysis_result}"
        )
        
        return report

# 使用
system = MultiAgentSystem({
    "researcher": ResearchAgent(llm, search_tools),
    "analyst": AnalystAgent(llm, data_tools),
    "writer": WriterAgent(llm)
})

report = system.run("分析2024年AI行业发展趋势")
```

---

## 4. Tool Calling工具调用

### 4.1 Function Calling格式

**OpenAI格式**：
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，例如：北京、上海"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 调用LLM
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "北京今天天气怎么样？"}
    ],
    tools=tools,
    tool_choice="auto"  # 让LLM自动决定是否调用工具
)

# LLM返回工具调用
tool_call = response.choices[0].message.tool_calls[0]
# {
#   "id": "call_123",
#   "type": "function",
#   "function": {
#     "name": "get_weather",
#     "arguments": '{"city": "北京", "unit": "celsius"}'
#   }
# }

# 执行工具
args = json.loads(tool_call.function.arguments)
result = get_weather(**args)
# {"temperature": 20, "condition": "晴天"}

# 回传结果给LLM
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "北京今天天气怎么样？"},
        response.choices[0].message,
        {
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result)
        }
    ]
)

# LLM生成最终回答
print(response.choices[0].message.content)
# "北京今天天气晴天，温度20度。"
```

### 4.2 工具设计原则

**1. 单一职责**
```python
# ❌ 不好：一个工具做多件事
def do_everything(action, **kwargs):
    if action == "search":
        return search(**kwargs)
    elif action == "calculate":
        return calculate(**kwargs)
    ...

# ✅ 好：每个工具做一件事
def search(query: str) -> str:
    """搜索信息"""
    pass

def calculate(expression: str) -> float:
    """计算表达式"""
    pass
```

**2. 清晰描述**
```python
# ❌ 不好：描述不清楚
def tool1(x, y):
    """处理数据"""
    pass

# ✅ 好：清晰描述输入输出
def search_knowledge_base(query: str, top_k: int = 5) -> List[Dict]:
    """
    在知识库中搜索相关内容
    
    Args:
        query: 搜索关键词，必填
        top_k: 返回结果数量，默认5，范围1-20
    
    Returns:
        List[Dict]: 搜索结果列表，每个结果包含：
            - title: 标题
            - content: 内容摘要
            - score: 相关性分数
    
    Example:
        >>> search_knowledge_base("分布式事务", top_k=3)
        [{"title": "2PC协议", "content": "...", "score": 0.95}, ...]
    """
    pass
```

**3. 参数验证**
```python
def transfer_money(from_account: str, to_account: str, amount: float) -> Dict:
    """转账工具"""
    
    # 参数验证
    if not from_account or not to_account:
        return {"error": "账户不能为空"}
    
    if amount <= 0:
        return {"error": "金额必须大于0"}
    
    if amount > 10000:
        return {"error": "单笔转账不能超过10000元"}
    
    # 执行转账
    try:
        result = do_transfer(from_account, to_account, amount)
        return {"success": True, "transaction_id": result.id}
    except Exception as e:
        return {"error": str(e)}
```

### 4.3 错误处理

```python
class ToolExecutor:
    def __init__(self, tools, max_retries=3):
        self.tools = tools
        self.max_retries = max_retries
    
    def execute(self, tool_name, args):
        """执行工具（带重试）"""
        tool = self.tools.get(tool_name)
        if not tool:
            return {"error": f"工具{tool_name}不存在"}
        
        for i in range(self.max_retries):
            try:
                result = tool.run(**args)
                return {"success": True, "result": result}
            except TimeoutError:
                if i < self.max_retries - 1:
                    time.sleep(2 ** i)  # 指数退避
                    continue
                return {"error": "工具调用超时"}
            except ValueError as e:
                return {"error": f"参数错误：{str(e)}"}
            except Exception as e:
                return {"error": f"执行失败：{str(e)}"}
```

---

## 5. RAG检索增强生成

### 5.1 RAG架构

**流程**：
```
用户问题 → Embedding → 向量检索 → 召回文档 → 构造Prompt → LLM生成答案
```

**代码实现**：
```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chains import RetrievalQA
from langchain_openai import ChatOpenAI

# 1. 构建向量库
documents = load_documents()  # 加载文档
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(documents, embeddings)

# 2. 创建检索链
retriever = vectorstore.as_retriever(
    search_type="similarity",  # 相似度检索
    search_kwargs={"k": 3}     # 返回前3个结果
)

qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4"),
    retriever=retriever,
    return_source_documents=True  # 返回引用来源
)

# 3. 查询
result = qa_chain.invoke({"query": "什么是分布式事务？"})

print("回答：", result["result"])
print("引用来源：", result["source_documents"])
```

### 5.2 向量检索优化

**1. Hybrid Search（混合检索）**
```python
from langchain.retrievers import EnsembleRetriever
from langchain.retrievers import BM25Retriever

# 向量检索器
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# 关键词检索器（BM25）
bm25_retriever = BM25Retriever.from_documents(documents)
bm25_retriever.k = 10

# 混合检索器（权重：向量0.5，关键词0.5）
ensemble_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.5, 0.5]
)

docs = ensemble_retriever.get_relevant_documents("分布式事务")
```

**2. Reranking（重排序）**
```python
from sentence_transformers import CrossEncoder

# 1. 初步检索（召回100个候选）
candidates = vectorstore.similarity_search(query, k=100)

# 2. 重排序（选出前10个）
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
scores = reranker.predict([(query, doc.page_content) for doc in candidates])

# 3. 按分数排序
ranked_docs = [candidates[i] for i in scores.argsort()[::-1][:10]]
```

**3. Query Expansion（查询扩展）**
```python
def expand_query(query, llm):
    """扩展查询"""
    prompt = f"""
    原始查询：{query}
    
    请生成3个语义相似的查询，用于更全面地检索：
    1. ...
    2. ...
    3. ...
    """
    expanded = llm.generate(prompt)
    return [query] + parse_queries(expanded)

# 使用
query = "分布式事务"
queries = expand_query(query, llm)
# ["分布式事务", "2PC协议", "TCC事务", "SAGA模式"]

# 对每个查询检索，合并结果
all_docs = []
for q in queries:
    docs = vectorstore.similarity_search(q, k=5)
    all_docs.extend(docs)

# 去重
unique_docs = list({doc.page_content: doc for doc in all_docs}.values())
```

### 5.3 Chunking策略

**1. 固定大小分块**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # 每块500字符
    chunk_overlap=50,     # 重叠50字符
    separators=["\n\n", "\n", "。", " "]  # 分隔符优先级
)

chunks = splitter.split_text(long_text)
```

**2. 语义分块**
```python
from langchain.text_splitter import SemanticChunker

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # 百分位阈值
    breakpoint_threshold_amount=95           # 95%分位点
)

chunks = splitter.split_text(long_text)
```

**3. 保留上下文**
```python
def chunk_with_context(text, chunk_size=500, context_size=100):
    """分块时保留上下文"""
    chunks = []
    for i in range(0, len(text), chunk_size):
        start = max(0, i - context_size)
        end = min(len(text), i + chunk_size + context_size)
        chunk = text[start:end]
        chunks.append({
            "content": chunk,
            "metadata": {
                "start": i,
                "end": i + chunk_size,
                "has_prefix": start < i,
                "has_suffix": end > i + chunk_size
            }
        })
    return chunks
```

---

## 6. Memory记忆机制

### 6.1 短期记忆

**对话历史（Context Window内）**：
```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()

# 添加对话
memory.save_context(
    {"input": "我叫小明"},
    {"output": "你好小明！"}
)

memory.save_context(
    {"input": "我叫什么名字？"},
    {"output": "你叫小明。"}
)

# 获取历史
print(memory.load_memory_variables({}))
# {
#   "history": "Human: 我叫小明\nAI: 你好小明！\nHuman: 我叫什么名字？\nAI: 你叫小明。"
# }
```

**窗口记忆（限制长度）**：
```python
from langchain.memory import ConversationBufferWindowMemory

# 只保留最近3轮对话
memory = ConversationBufferWindowMemory(k=3)
```

**摘要记忆（压缩历史）**：
```python
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(llm=ChatOpenAI())

# 长对话会被LLM自动总结
memory.save_context(
    {"input": "我昨天去了北京...（很长的对话）"},
    {"output": "听起来你的北京之旅很愉快！..."}
)

# 获取摘要
print(memory.load_memory_variables({}))
# {"history": "用户昨天去了北京旅游，参观了故宫和长城..."}
```

### 6.2 长期记忆

**向量库存储**：
```python
class LongTermMemory:
    def __init__(self, vectorstore):
        self.vectorstore = vectorstore
    
    def save(self, conversation):
        """保存对话到长期记忆"""
        # 提取关键信息
        summary = extract_key_info(conversation)
        
        # 存入向量库
        self.vectorstore.add_texts([summary])
    
    def recall(self, query, top_k=3):
        """回忆相关记忆"""
        docs = self.vectorstore.similarity_search(query, k=top_k)
        return [doc.page_content for doc in docs]

# 使用
memory = LongTermMemory(vectorstore)

# 保存
memory.save({
    "user": "我喜欢Python",
    "assistant": "很好！Python是一门优秀的语言。"
})

# 回忆
related_memories = memory.recall("我的编程偏好")
# ["用户喜欢Python", ...]
```

**结构化存储**：
```python
class StructuredMemory:
    def __init__(self):
        self.user_profile = {}
        self.conversation_history = []
    
    def update_profile(self, key, value):
        """更新用户画像"""
        self.user_profile[key] = value
    
    def get_profile(self):
        """获取用户画像"""
        return self.user_profile

# 使用
memory = StructuredMemory()

# 从对话中提取信息
memory.update_profile("name", "小明")
memory.update_profile("preferences", ["Python", "AI"])
memory.update_profile("location", "北京")

# 下次对话时注入
profile = memory.get_profile()
prompt = f"用户资料：{profile}\n\n用户问题：{query}"
```

---

## 7. 幻觉问题与解决

### 7.1 什么是幻觉？

**幻觉（Hallucination）**：LLM生成看似合理但实际错误的内容

**示例**：
```
用户：2024年诺贝尔物理学奖得主是谁？

LLM（幻觉）：2024年诺贝尔物理学奖得主是张三，因其在量子计算领域的突破性贡献。

实际：2024年奖项尚未公布，LLM编造了虚假信息。
```

### 7.2 解决方案

**1. RAG注入真实数据**
```python
# 从知识库检索真实数据
docs = vectorstore.similarity_search("2024年诺贝尔物理学奖")

# 构造Prompt
prompt = f"""
参考资料：
{docs}

问题：2024年诺贝尔物理学奖得主是谁？

请基于参考资料回答，如果资料中没有提到，请说"资料中未提及"。
"""

response = llm.generate(prompt)
# "资料中未提及2024年诺贝尔物理学奖得主信息。"
```

**2. 工具调用**
```python
# 让LLM调用搜索工具，而非编造
tools = [
    Tool(name="Search", func=search_web, description="搜索最新信息")
]

agent = create_react_agent(llm, tools)
result = agent.run("2024年诺贝尔物理学奖得主是谁？")

# Agent会调用Search工具获取真实信息
```

**3. Prompt约束**
```python
prompt = """
请回答以下问题，注意：
1. 只回答你确定的信息
2. 如果不确定，明确说"我不知道"
3. 不要编造任何信息
4. 如果需要最新信息，说明需要查询

问题：2024年诺贝尔物理学奖得主是谁？
"""

response = llm.generate(prompt)
# "我不知道2024年诺贝尔物理学奖得主，因为我的知识截止到2023年，需要查询最新信息。"
```

**4. 自我一致性检查**
```python
def check_consistency(answer, llm):
    """检查答案一致性"""
    prompt = f"""
    答案：{answer}
    
    请判断以上答案是否包含：
    1. 事实性错误
    2. 逻辑矛盾
    3. 编造的信息
    
    如果有问题，回答"有问题"并说明原因；否则回答"无问题"。
    """
    result = llm.generate(prompt)
    return "无问题" in result
```

---

## 8. 常见面试题

### Q1: Transformer中的Self-Attention机制是什么？

**答案**：
Self-Attention计算序列中每个token与其他token的相关性。

**公式**：`Attention(Q, K, V) = softmax(QK^T / √d_k) * V`

**过程**：
1. 将输入转换为Q、K、V向量
2. 计算Q和K的点积，得到attention分数
3. 除以√d_k进行缩放
4. 经过softmax归一化
5. 与V相乘得到输出

**为什么除以√d_k？**
- 防止点积过大导致梯度消失

### Q2: LLM幻觉问题如何解决？

**答案**：
1. **RAG**：注入真实数据
2. **工具调用**：让LLM查询而非编造
3. **Prompt约束**："如果不知道，说不知道"
4. **后处理验证**：检查事实性
5. **自我一致性**：多次生成，选择一致的答案

### Q3: RAG的核心流程是什么？

**答案**：
1. **文档处理**：分块、生成Embedding
2. **向量存储**：存入向量数据库
3. **检索**：用户问题Embedding，相似度检索
4. **构造Prompt**：将检索到的文档作为上下文
5. **生成答案**：LLM基于上下文生成

**优化技巧**：
- Hybrid Search（向量+关键词）
- Reranking（重排序）
- Query Expansion（查询扩展）

### Q4: Agent的核心循环是什么？

**答案**：
```
感知 → 规划 → 执行 → 观察 → 反思
  ↑                           ↓
  └───────────────────────────┘
```

**ReAct模式**：
- **思考**：分析当前状态
- **行动**：调用工具
- **观察**：查看结果
- 循环直到完成

---

## 9. 实战案例

### 案例1：智能客服Agent

```python
class CustomerServiceAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = {
            "search_order": search_order_tool,
            "search_product": search_product_tool,
            "refund": refund_tool
        }
    
    def run(self, user_query):
        """处理客户咨询"""
        # 1. 意图识别
        intent = self.classify_intent(user_query)
        
        if intent == "查询订单":
            return self.handle_order_query(user_query)
        elif intent == "退款":
            return self.handle_refund(user_query)
        else:
            return self.handle_general_query(user_query)
    
    def classify_intent(self, query):
        """意图识别"""
        prompt = f"""
        用户咨询：{query}
        
        请判断用户意图（只返回以下之一）：
        - 查询订单
        - 查询物流
        - 退款
        - 咨询产品
        - 其他
        """
        return self.llm.generate(prompt).strip()
    
    def handle_order_query(self, query):
        """处理订单查询"""
        # 提取订单号
        order_id = extract_order_id(query)
        
        # 查询订单
        order = self.tools["search_order"](order_id)
        
        # 生成回复
        prompt = f"""
        用户查询：{query}
        订单信息：{order}
        
        请生成友好的客服回复：
        """
        return self.llm.generate(prompt)

# 使用
agent = CustomerServiceAgent(llm, tools)
response = agent.run("我的订单12345什么时候到？")
```

### 案例2：代码审查Agent

```python
class CodeReviewAgent:
    def __init__(self, llm):
        self.llm = llm
    
    def review(self, code, language="python"):
        """审查代码"""
        # 1. 静态分析
        issues = self.static_analysis(code, language)
        
        # 2. LLM审查
        llm_review = self.llm_review(code, language)
        
        # 3. 合并结果
        return self.merge_results(issues, llm_review)
    
    def llm_review(self, code, language):
        """LLM审查代码"""
        prompt = f"""
        请审查以下{language}代码，关注：
        1. 潜在的bug
        2. 性能问题
        3. 代码规范
        4. 安全漏洞
        
        代码：
        ```{language}
        {code}
        ```
        
        请以JSON格式返回：
        {{
          "bugs": [...],
          "performance": [...],
          "style": [...],
          "security": [...]
        }}
        """
        return json.loads(self.llm.generate(prompt))

# 使用
agent = CodeReviewAgent(llm)
result = agent.review("""
def transfer(from_account, to_account, amount):
    from_account.balance -= amount
    to_account.balance += amount
""")

print(result)
# {
#   "bugs": ["没有检查余额是否足够", "没有事务保护"],
#   "performance": [],
#   "style": ["缺少类型提示", "缺少文档字符串"],
#   "security": ["没有权限检查", "没有金额上限"]
# }
```

---

**总结**

LLM与Agent核心：
1. **LLM原理**：Transformer、Attention、Tokenization
2. **Prompt Engineering**：CoT、ReAct、Self-Consistency
3. **Agent架构**：感知→规划→执行→观察→反思
4. **Tool Calling**：Function Calling、参数验证、错误处理
5. **RAG**：向量检索、混合检索、重排序
6. **Memory**：短期（对话历史）、长期（向量库+结构化）
7. **幻觉解决**：RAG、工具调用、Prompt约束

面试时展示**对LLM核心原理的理解**和**Agent实战经验**！🤖

---

**文件状态**：✅ 已完成（约800行）