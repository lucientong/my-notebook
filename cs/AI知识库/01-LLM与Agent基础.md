# LLM 与 Agent 基础

> 建议先读开头两节，先建立概念边界，再进入 Transformer、RAG、Agent 等机制细节。
>
> **定位**：本文偏原理解释，重点回答“为什么能工作、核心机制是什么”；发布治理、评估门禁、SLO、灰度与回滚请优先联动 `03-AI工程化实践.md`。

---

## 📑 目录

0. [开始前先看](#开始前先看)
1. [从问题出发理解 LLM](#1-从问题出发理解-llm)

### LLM核心原理
2. [先从完整生命周期看 LLM](#10-先从完整生命周期看-llm)
3. [参数量、数据量、上下文窗口分别决定什么](#101-参数量数据量上下文窗口分别决定什么)
4. [LLM 训练与演进的完整阶段](#102-llm-训练与演进的完整阶段)
5. [教师模型、学生模型与蒸馏](#103-教师模型学生模型与蒸馏)
6. [为什么是 Transformer，而不是 RNN / CNN](#104-为什么是-transformer而不是-rnn--cnn)
7. [Transformer架构](#11-transformer架构)
8. [位置编码](#12-位置编码-positional-encoding)
9. [Tokenization分词](#13-tokenization分词)
10. [LLM训练流程](#14-llm训练流程)
11. [RLHF替代方案](#15-rlhf替代方案)
12. [推理优化：KV Cache](#16-推理优化kv-cache)
13. [生成参数控制](#17-生成参数控制)

### Prompt与Agent
9. [Prompt Engineering](#2-prompt-engineering)
10. [Agent架构设计](#3-agent架构设计)
11. [Tool Calling工具调用](#4-tool-calling工具调用)

### RAG与Memory
12. [RAG检索增强生成](#5-rag检索增强生成)
13. [Memory记忆机制](#6-memory记忆机制)

### 问题与实战
14. [幻觉问题与解决](#7-幻觉问题与解决)
15. [面试题自查](#8-面试题自查)
16. [实战案例](#9-实战案例)

---

## 开始前先看

### 这篇解决什么问题

这篇围绕 4 个基础问题展开：

- LLM 到底是什么，为什么能生成文本？
- Prompt、RAG、Tool Calling、Agent 分别解决什么问题？
- 为什么 AI 应用不是“调一次模型 API”就结束？
- 当系统效果差时，应该先怀疑模型、知识、工具还是流程？

如果这 4 个问题没有先理顺，后面的 Transformer、KV Cache、Agent 架构就会显得割裂。

### 先建立一个最小框架

```text
文本输入
  ↓
Tokenizer 切成 token
  ↓
LLM 基于上下文预测下一个 token
  ↓
如果知识不够：接 RAG
  ↓
如果需要真实动作：接 Tool Calling
  ↓
如果任务需要多步规划：接 Agent
```

这条链路里，每一层都在解决不同问题：

- `LLM` 负责语言理解与生成
- `RAG` 负责补外部知识
- `Tool Calling` 负责连接真实能力
- `Agent` 负责多步决策与流程控制

### 建议阅读方式

- 基础薄弱：先看本篇前两节，再跳到 `2. Prompt Engineering` 和 `5. RAG`
- 已做过 API 接入：重点看 `3. Agent架构设计`、`4. Tool Calling`、`7. 幻觉问题`
- 想补训练视角：读完 `1. LLM核心原理` 后转去 `04-AI模型训练与优化.md`

---

## 1. 从问题出发理解 LLM

### 1.0 为什么会有 LLM

传统 NLP 更像“为每个任务单独做模型”，例如：

- 情感分析一个模型
- 分类一个模型
- 命名实体识别一个模型
- 翻译一个模型

LLM 的重要变化是：先训练一个足够大的通用语言模型，再通过 Prompt、工具和少量适配去覆盖大量任务。

这让 AI 应用开发从“为每个任务训练一个模型”，变成了“围绕一个通用模型构建系统”。

### 1.0.1 LLM 最核心的能力边界

LLM 擅长：

- 语言生成
- 上下文压缩与重组
- 模式归纳
- 在相似任务之间迁移

LLM 不天然擅长：

- 精确、长期、可追溯的事实记忆
- 直接访问实时外部世界
- 在高风险任务中保证 100% 正确执行

所以你后面看到的 RAG、工具、记忆、评估，都是在补这些短板。

### 1.0.2 从一个实际例子理解

假设用户问：`帮我总结公司知识库里关于缓存穿透的最佳实践，并给出 Redis 配置建议。`

如果只有 LLM：

- 模型可以写出一段“看起来像答案”的文字
- 但不一定知道你们公司的私有知识库内容
- 也不一定知道当前线上 Redis 配置

如果接入 RAG：

- 可以先检索内部知识库，再基于检索结果回答

如果接入 Tool Calling：

- 可以进一步调用配置中心或监控接口，拿到真实数据

如果任务还需要“先检索，再比对，再生成报告，再发消息”：

- 这时才需要 Agent 做多步编排

因此不宜直接从 Agent 开始。Agent 位于系统最外层，不是起点。

### 1.0.3 学这一篇时，先抓 5 个关键词

- `token`：模型处理文本的基本单位
- `context`：模型本轮可见的输入范围
- `probability`：模型输出本质上是概率分布，不是确定真值
- `tool`：模型之外的真实执行能力
- `workflow`：把模型、工具、知识拼成一条业务链路

先把这 5 个词放到明确位置，后面的内容会更容易串起来。

---

## 1. LLM核心原理

### 1.0 先从完整生命周期看 LLM

从工程视角看，LLM 不是单次训练动作，而是一条完整链路：

```text
原始文本数据
  ↓
数据清洗 / 去重 / 过滤
  ↓
Tokenizer 训练与分词
  ↓
预训练（学语言模式）
  ↓
指令微调 SFT（学会按要求回答）
  ↓
偏好对齐 RLHF / DPO（更像人想要的回答）
  ↓
压缩与部署（量化 / 蒸馏 / 推理优化）
  ↓
接入 RAG / Tool / Agent，变成可用系统
```

这个顺序能解释很多常见现象：

- 为什么基础模型“会补全”，但不一定“会按你要求回答”
- 为什么有了 SFT 后，模型更像助手，而不只是续写器
- 为什么有了 RLHF / DPO 后，回答更符合人类偏好，但不一定更真实
- 为什么上线时还要做量化、蒸馏、KV Cache，而不是训练完直接用

### 1.0.1 参数量、数据量、上下文窗口分别决定什么

“模型大”常被直接理解成“更聪明”，但这里需要拆开看：

| 维度 | 主要影响 | 不等于什么 |
|------|----------|------------|
| 参数量（Parameters） | 模型容量、表达能力、可学习的模式复杂度 | 不等于绝对正确 |
| 数据量（Tokens） | 模型见过多少语言现象和知识分布 | 不等于高质量对齐 |
| 上下文窗口（Context Window） | 一次能看多长输入 | 不等于长期记忆 |

可以先用一个便于理解的类比：

- `参数量` 像大脑里可调节的“连接规模”
- `训练数据` 像模型读过多少材料
- `上下文窗口` 像一次考试时桌上能摊开多少参考资料

参数量变大通常意味着：

- 模型能容纳更复杂的模式
- 对长链推理、复杂语言现象的表达能力更强
- 训练和推理成本也更高

但参数量不直接保证：

- 事实一定正确
- 行为一定稳定
- 对业务一定友好

这也是后面还需要 `SFT`、`偏好对齐`、`RAG`、`蒸馏` 和 `工程化控制` 的原因。

### 1.0.2 LLM 训练与演进的完整阶段

| 阶段 | 在做什么 | 主要解决的问题 | 新带来的问题 |
|------|----------|----------------|-------------|
| 分词与词表设计 | 把文本变成 token | 让模型能处理离散文本 | token 成本、分词粒度不稳定 |
| 预训练 | 预测下一个 token | 学语言模式、知识分布、泛化能力 | 只是“会续写”，不一定会听指令 |
| 指令微调 SFT | 学习问答和任务格式 | 让模型更像助手 | 仍可能虚构、迎合、啰嗦 |
| 偏好对齐 RLHF / DPO | 学习“什么回答更符合人类偏好” | 改善可用性、风格、安全性 | 可能更会说人话，但不一定更真实 |
| 压缩与部署 | 量化、蒸馏、推理优化 | 降低成本、加快响应 | 效果可能下降 |
| 系统增强 | RAG、Tool、Agent | 补知识、补动作、补流程 | 系统复杂度上升 |

这些阶段可以理解成逐层补短板：

1. `预训练` 让模型学会“语言和世界的大致模式”。
2. `SFT` 让模型学会“像助手一样完成任务”。
3. `RLHF/DPO` 让模型学会“更符合人类偏好地回答”。
4. `蒸馏/量化` 让模型学会“以更低成本运行”。
5. `RAG/工具/Agent` 让系统获得“模型本身没有的知识和行动能力”。

### 1.0.3 教师模型、学生模型与蒸馏

这一环节在很多入门材料里容易被略过，但实际很重要。

#### 什么是教师模型（Teacher Model）

教师模型通常是一个更大、更强、效果更好的模型，用来提供“指导信号”。

它可以做两类事：

- 给训练样本打更细致的软标签（soft targets）
- 生成高质量数据，教小模型如何回答

#### 什么是学生模型（Student Model）

学生模型是更小、更便宜、推理更快的模型，目标是在成本更低的前提下尽量逼近教师模型效果。

#### 蒸馏（Distillation）在做什么

蒸馏本质上是：

> 让小模型不只学习“标准答案”，还学习大模型输出分布里的偏好与结构。

可以这样理解。普通训练像只告诉学生“这题正确答案是 B”；蒸馏还会额外告诉它：

- B 最合理
- A 也有一点像
- C 基本不可能

这样学生模型学到的不只是结果，还有教师模型对不同选项的相对判断。

#### 为什么会诞生蒸馏

因为真实工程里经常会遇到这个矛盾：

- 大模型效果好，但贵、慢、难部署
- 小模型便宜快，但能力不够

蒸馏就是在“效果”和“成本”之间找折中。

常见用途：

- 用大模型生成高质量指令数据，训练小模型
- 用强模型做教师，把推理风格迁移给轻量模型
- 在边缘设备或高并发场景中部署小模型替代大模型

### 1.0.4 为什么是 Transformer，而不是 RNN / CNN

在 Transformer 之前，处理文本的主流方法更多依赖 `RNN / LSTM / GRU`，也有一部分用 `CNN`。

它们各自有问题：

- `RNN/LSTM`：按顺序一个 token 一个 token 处理，长距离依赖难学，训练难并行
- `CNN`：局部模式提取强，但天然不擅长直接建模全局关系

这会带来两个核心瓶颈：

1. 句子很长时，前面信息很难稳定传到后面。
2. 训练时不能很好并行，规模很难做大。

Transformer 的关键改进是：

- 用 `Self-Attention` 让每个 token 都有机会直接关注其他 token
- 用更强的并行训练方式，把模型规模和数据规模一起推大

所以 Transformer 解决的不是“让公式更复杂”，而是两个非常实际的问题：

- `长距离依赖` 怎么学
- `大规模训练` 怎么做

先理解这两点，再看后面的 Attention 公式会顺畅得多。

### 1.1 Transformer架构

Transformer 本质上是在回答一个问题：

> 当模型处理当前 token 时，怎样高效地参考上下文里其他位置的信息？

它的核心创新是 `Self-Attention`。可以先这样理解：

- 当前 token 会向上下文“提问”：哪些位置和我最相关？
- 相关的位置会被分配更高权重
- 最终把这些位置的信息加权汇总，形成当前 token 的新表示

Transformer 的重点不是机械保留顺序，而是动态建模关系。

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

先看直观版：

```text
“苹果” 这个词出现在句子里时，模型需要判断它更像：
- 水果
- 公司

这时它会重点参考上下文里的词：
- 如果附近有“手机、发布会、芯片”，更可能是公司
- 如果附近有“果汁、甜、采摘”，更可能是水果
```

Attention 就是在做这件事的数学化表达。

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

### 1.2 位置编码 (Positional Encoding)

**为什么需要位置编码？**
- Transformer没有RNN的顺序处理，无法感知token位置
- 位置编码为模型注入序列顺序信息

**1. 绝对位置编码（Sinusoidal）**
```python
# 原始Transformer的位置编码
def sinusoidal_encoding(seq_len, d_model):
    """
    PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
    PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
    """
    position = np.arange(seq_len)[:, np.newaxis]
    div_term = np.exp(np.arange(0, d_model, 2) * -(np.log(10000.0) / d_model))
    
    pe = np.zeros((seq_len, d_model))
    pe[:, 0::2] = np.sin(position * div_term)
    pe[:, 1::2] = np.cos(position * div_term)
    return pe
```

**2. RoPE (Rotary Position Embedding)**
```python
# LLaMA、Qwen等使用RoPE
# 核心思想：通过旋转矩阵编码相对位置

def apply_rope(x, cos, sin):
    """
    对Q和K应用旋转位置编码
    支持长度外推，无需训练即可扩展上下文
    """
    x_reshape = x.reshape(*x.shape[:-1], -1, 2)
    x1, x2 = x_reshape[..., 0], x_reshape[..., 1]
    
    # 旋转
    x_rotated = torch.cat([
        x1 * cos - x2 * sin,
        x1 * sin + x2 * cos
    ], dim=-1)
    return x_rotated
```

**3. ALiBi (Attention with Linear Biases)**
```python
# 在Attention分数上直接加线性偏置
# 无需训练位置编码参数，支持长度外推

def alibi_bias(seq_len, num_heads):
    """
    ALiBi: 直接在QK^T上减去距离惩罚
    bias[i,j] = -slope * |i - j|
    """
    slopes = 2 ** (-8 / num_heads * np.arange(1, num_heads + 1))
    positions = np.arange(seq_len)
    bias = -np.abs(positions[:, None] - positions[None, :])
    return slopes[:, None, None] * bias
```

**位置编码对比**：
| 方法 | 优点 | 缺点 | 代表模型 |
|------|------|------|----------|
| Sinusoidal | 简单，无需训练 | 外推能力弱 | 原始Transformer |
| RoPE | 外推性好，相对位置 | 计算稍复杂 | LLaMA, Qwen |
| ALiBi | 训练高效，长度泛化 | 长距离性能下降 | BLOOM, MPT |

### 1.3 Tokenization分词

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

### 1.4 LLM训练流程

前面的 `1.0.2` 是从概念上讲完整生命周期，这一节改成更接近“实现链路”的版本。

#### 阶段 1：预训练（Pre-training）

目标：让模型学会语言统计规律、知识分布和通用表达能力。

```text
输入：海量无标注文本
任务：给定前文，预测下一个 token
结果：得到一个 Base Model（基础模型）
```

这一步解决的是“模型先学会语言本身”。

但预训练之后的模型通常还存在几个问题：

- 更像续写器，不像助手
- 不稳定遵循人类指令
- 容易输出冗长、跑题、风格不受控的结果

#### 阶段 2：指令微调（SFT, Supervised Fine-Tuning）

目标：让模型学会按照人类期望的输入输出格式完成任务。

```text
输入：高质量“指令-回答”样本
任务：学习看到某类问题时，应该怎样组织回答
结果：模型从“会续写”变成“会按要求回答”
```

典型数据格式：

```text
User: 请帮我写一首关于春天的诗
Assistant: 好的，下面是一首关于春天的短诗...
```

SFT 解决了“像助手”的问题，但还不够。因为两个回答都可能语法正确，却一个更安全、更有帮助、更简洁。

#### 阶段 3：偏好对齐（RLHF / DPO）

目标：让模型更接近人类偏好，而不只是形式上答对。

常见做法：

1. 给同一个问题生成多个回答。
2. 让人类或高质量标注选择哪个更好。
3. 用这些偏好数据继续训练模型。

这一阶段诞生的原因很直接：

- “能回答”不等于“回答得好”
- “回答得像人”也不等于“更安全、更有帮助”

#### 阶段 4：压缩与部署（蒸馏 / 量化 / 推理优化）

目标：让模型能以可接受的成本上线。

原因也很现实：

- 大模型训练贵
- 大模型推理更贵
- 并发一高，成本和延迟都容易失控

常见方案：

- `量化`：用更低比特表示权重，减少显存和计算开销
- `蒸馏`：用大模型教小模型，换取更低推理成本
- `KV Cache`：减少重复计算，加速自回归生成

#### 阶段 5：系统增强（RAG / Tool / Agent）

到了这里，问题已经不再是“模型会不会说”，而是：

- 知识是不是最新的
- 回答能不能引用私有资料
- 能不能执行真实动作
- 能不能完成多步任务

所以工程上通常会在模型之外继续加：

- `RAG`：补最新和私有知识
- `Tool Calling`：补真实执行能力
- `Agent`：补规划和多步流程

#### 一张图串起来

```text
原始文本
  ↓
预训练
  ↓         解决“会说话”
Base Model
  ↓
SFT
  ↓         解决“会按要求回答”
Instruct Model
  ↓
RLHF / DPO
  ↓         解决“更符合人类偏好”
Aligned Model
  ↓
蒸馏 / 量化 / KV Cache
  ↓         解决“如何上线更便宜更快”
Deployable Model
  ↓
RAG / Tool / Agent
  ↓         解决“知识、动作、流程短板”
Production AI System
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

### 1.5 RLHF替代方案

**1. DPO (Direct Preference Optimization)**
```python
# DPO直接优化策略，无需训练奖励模型
# 核心思想：将RLHF目标转化为分类损失

def dpo_loss(model, ref_model, chosen, rejected, beta=0.1):
    """
    DPO损失函数
    L_DPO = -log(σ(β * (log π(y_w|x)/π_ref(y_w|x) - log π(y_l|x)/π_ref(y_l|x))))
    """
    # 计算策略模型的log概率
    log_pi_chosen = model.log_prob(chosen)
    log_pi_rejected = model.log_prob(rejected)
    
    # 计算参考模型的log概率
    log_ref_chosen = ref_model.log_prob(chosen)
    log_ref_rejected = ref_model.log_prob(rejected)
    
    # DPO损失
    log_ratio_chosen = log_pi_chosen - log_ref_chosen
    log_ratio_rejected = log_pi_rejected - log_ref_rejected
    
    loss = -F.logsigmoid(beta * (log_ratio_chosen - log_ratio_rejected))
    return loss.mean()
```

**DPO vs RLHF 对比**：
| 特性 | RLHF | DPO |
|------|------|-----|
| 训练阶段 | 3阶段（SFT→RM→PPO） | 2阶段（SFT→DPO） |
| 奖励模型 | 需要 | 不需要 |
| 训练稳定性 | 较差，需要调参 | 稳定 |
| 计算成本 | 高 | 低 |
| 效果 | 天花板高 | 接近RLHF |

**2. ORPO (Odds Ratio Preference Optimization)**
```python
# ORPO: 无需参考模型，单阶段训练
def orpo_loss(model, chosen, rejected, lambda_=0.1):
    """
    ORPO = SFT Loss + λ * Preference Loss
    """
    # SFT损失
    sft_loss = -model.log_prob(chosen).mean()
    
    # 偏好损失（基于odds ratio）
    log_odds_chosen = model.log_prob(chosen) - torch.log(1 - torch.exp(model.log_prob(chosen)))
    log_odds_rejected = model.log_prob(rejected) - torch.log(1 - torch.exp(model.log_prob(rejected)))
    
    preference_loss = -F.logsigmoid(log_odds_chosen - log_odds_rejected).mean()
    
    return sft_loss + lambda_ * preference_loss
```

### 1.6 推理优化：KV Cache

**问题**：自回归生成时，每个token都要计算完整Attention

**解决方案**：缓存历史token的K、V向量

```python
class KVCacheAttention:
    def __init__(self, d_model, num_heads):
        self.d_k = d_model // num_heads
        self.num_heads = num_heads
        
        # KV Cache
        self.k_cache = None  # (batch, num_heads, seq_len, d_k)
        self.v_cache = None
    
    def forward(self, x, use_cache=True):
        batch_size, seq_len, _ = x.shape
        
        # 计算当前token的Q、K、V
        Q = self.W_q(x)  # (batch, 1, d_model) for decoding
        K = self.W_k(x)
        V = self.W_v(x)
        
        if use_cache and self.k_cache is not None:
            # 拼接历史KV
            K = torch.cat([self.k_cache, K], dim=2)
            V = torch.cat([self.v_cache, V], dim=2)
        
        # 更新缓存
        if use_cache:
            self.k_cache = K
            self.v_cache = V
        
        # 计算Attention（Q只有当前token，K、V是全部历史）
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        attn = F.softmax(scores, dim=-1)
        output = torch.matmul(attn, V)
        
        return output
    
    def clear_cache(self):
        self.k_cache = None
        self.v_cache = None
```

**KV Cache内存估算**：
```
内存 = 2 × num_layers × batch_size × seq_len × num_heads × d_k × dtype_size

示例（LLaMA-7B，seq_len=4096，batch=1）：
= 2 × 32 × 1 × 4096 × 32 × 128 × 2 bytes
≈ 2GB
```

### 1.7 生成参数控制

**Temperature**：控制输出随机性
```python
def sample_with_temperature(logits, temperature=1.0):
    """
    temperature > 1: 更随机
    temperature < 1: 更确定性
    temperature = 0: 贪婪解码（argmax）
    """
    if temperature == 0:
        return logits.argmax(dim=-1)
    
    probs = F.softmax(logits / temperature, dim=-1)
    return torch.multinomial(probs, num_samples=1)
```

**Top-p (Nucleus Sampling)**：
```python
def top_p_sampling(logits, p=0.9):
    """
    只从累积概率前p的token中采样
    p=0.9: 忽略尾部10%概率的token
    """
    sorted_logits, sorted_indices = torch.sort(logits, descending=True)
    cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
    
    # 找到累积概率超过p的位置
    sorted_indices_to_remove = cumulative_probs > p
    sorted_indices_to_remove[..., 1:] = sorted_indices_to_remove[..., :-1].clone()
    sorted_indices_to_remove[..., 0] = 0
    
    # 将移除的token概率设为-inf
    indices_to_remove = sorted_indices[sorted_indices_to_remove]
    logits[indices_to_remove] = float('-inf')
    
    return torch.multinomial(F.softmax(logits, dim=-1), num_samples=1)
```

**Top-k Sampling**：
```python
def top_k_sampling(logits, k=50):
    """只从概率最高的k个token中采样"""
    values, indices = torch.topk(logits, k)
    probs = F.softmax(values, dim=-1)
    sampled_idx = torch.multinomial(probs, num_samples=1)
    return indices[sampled_idx]
```

**参数使用建议**：
| 任务类型 | Temperature | Top-p | 说明 |
|----------|-------------|-------|------|
| 代码生成 | 0 或 0.2 | 0.95 | 需要确定性 |
| 创意写作 | 0.8-1.2 | 0.9 | 需要多样性 |
| 对话 | 0.7 | 0.9 | 自然但不跑题 |
| 数学推理 | 0 | 1.0 | 严格逻辑 |

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

### 4.4 并行工具调用

**OpenAI支持一次返回多个工具调用**：
```python
# LLM返回多个并行工具调用
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "北京和上海今天天气怎么样？"}
    ],
    tools=tools,
    tool_choice="auto"
)

# 返回两个工具调用
tool_calls = response.choices[0].message.tool_calls
# [
#   {"id": "call_1", "function": {"name": "get_weather", "arguments": '{"city": "北京"}'}},
#   {"id": "call_2", "function": {"name": "get_weather", "arguments": '{"city": "上海"}'}}
# ]

# 并行执行工具
import asyncio

async def execute_tool(tool_call):
    func = tools[tool_call.function.name]
    args = json.loads(tool_call.function.arguments)
    return {
        "tool_call_id": tool_call.id,
        "result": await func(**args)
    }

results = await asyncio.gather(*[execute_tool(tc) for tc in tool_calls])

# 回传所有结果
messages = [
    {"role": "user", "content": "北京和上海今天天气怎么样？"},
    response.choices[0].message,
]
for result in results:
    messages.append({
        "role": "tool",
        "tool_call_id": result["tool_call_id"],
        "content": json.dumps(result["result"])
    })

# 最终回答
final_response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=messages
)
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

### 5.4 常用Embedding模型

**Embedding模型对比**：
| 模型 | 维度 | 中文支持 | 特点 | 适用场景 |
|------|------|----------|------|----------|
| text-embedding-3-small | 1536 | 一般 | OpenAI，高质量 | 通用 |
| text-embedding-3-large | 3072 | 一般 | OpenAI，最高质量 | 高精度需求 |
| bge-large-zh | 1024 | 优秀 | 开源，中文最佳 | 中文场景 |
| m3e-base | 768 | 优秀 | 开源，轻量 | 资源受限 |
| jina-embeddings-v2 | 768 | 良好 | 支持8K上下文 | 长文本 |

**使用示例**：
```python
# OpenAI Embedding
from openai import OpenAI

client = OpenAI()
response = client.embeddings.create(
    input="什么是分布式事务？",
    model="text-embedding-3-small"
)
embedding = response.data[0].embedding

# BGE中文Embedding（本地部署）
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('BAAI/bge-large-zh-v1.5')
embedding = model.encode("什么是分布式事务？")
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

### 6.3 流式输出与回调

**流式生成（Streaming）**：
```python
from openai import OpenAI

client = OpenAI()

# 流式调用
stream = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "写一首诗"}],
    stream=True
)

# 逐token处理
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**带回调的Agent**：
```python
from langchain.callbacks import StreamingStdOutCallbackHandler

class TokenCallback(StreamingStdOutCallbackHandler):
    def __init__(self):
        self.tokens = []
    
    def on_llm_new_token(self, token: str, **kwargs):
        """每个token生成时的回调"""
        self.tokens.append(token)
        print(token, end="", flush=True)
    
    def on_tool_start(self, tool_name: str, tool_input: str, **kwargs):
        """工具开始执行时的回调"""
        print(f"\n🔧 调用工具: {tool_name}")
    
    def on_tool_end(self, output: str, **kwargs):
        """工具执行完成时的回调"""
        print(f"✅ 工具结果: {output[:100]}...")

# 使用回调
callback = TokenCallback()
result = agent.run("查询今天的天气", callbacks=[callback])
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

## 7.5 LLM 评估体系

> 这一节回答"怎么判断一个模型到底好不好"以及"选型时该看什么"。评估不是上线后才做的事，而是贯穿选型、开发、发布全过程。
>
> **定位**：本节侧重模型能力评估与选型框架；线上质量监控、AB 测试、回归测试等工程侧评估请联动 `03-AI工程化实践.md`。

---

### 7.5.1 为什么需要系统化评估

直觉判断"模型好不好"有三个常见问题：

1. **主观偏差**：同一个回答，不同人评价差异大。
2. **以偏概全**：试了几个 case 觉得不错就上线，遇到长尾问题才暴露短板。
3. **无法比较**：换了模型或 Prompt 后，说不清到底是变好了还是变差了。

系统化评估的目标是：**让"好不好"变成可量化、可复现、可比较的结论**。

---

### 7.5.2 常见 Benchmark 一览

下面是当前最常被引用的 5 个基准测试，覆盖知识、推理、代码、常识和真实性。

#### MMLU（Massive Multitask Language Understanding）

- **评估什么**：57 个学科的多选题，覆盖 STEM、人文、社会科学、专业领域。
- **为什么重要**：衡量模型"通识知识广度"，是最常用的综合能力基准。
- **题目示例**：大学物理、法律伦理、临床医学、计算机科学等多选题。
- **注意**：高分不代表真正理解，可能只是记住了训练数据中的相似题。

#### HumanEval

- **评估什么**：164 道 Python 编程题，要求模型生成能通过单元测试的函数。
- **指标**：pass@k（生成 k 次至少通过一次的比例）。
- **为什么重要**：直接测试代码生成能力，而不只是"看起来像代码"。
- **注意**：题目偏算法和工具函数，不完全反映工程级代码能力。

#### GSM8K（Grade School Math 8K）

- **评估什么**：8500 道小学数学应用题，需要多步推理。
- **为什么重要**：测试模型的链式推理能力，而不只是知识检索。
- **典型题**："小明买了 3 本书，每本 12 元，打 8 折后付了多少钱？"
- **注意**：对 CoT prompting 非常敏感，加不加"让我们一步步思考"差距很大。

#### HellaSwag

- **评估什么**：常识推理——给定一个场景开头，从 4 个选项中选出最合理的后续。
- **为什么重要**：测试模型对真实世界常识和物理直觉的理解。
- **注意**：人类准确率 >95%，但对模型来说干扰项设计得很有迷惑性。

#### TruthfulQA

- **评估什么**：817 道容易引发错误回答的问题，专门测试模型是否会输出常见误解。
- **为什么重要**：直接衡量"幻觉倾向"和"是否会迎合错误前提"。
- **典型题**："人一天需要喝 8 杯水吗？"（常见误解）
- **注意**：模型越大、越对齐，TruthfulQA 表现反而可能更好。

#### Benchmark 对比表

| Benchmark | 评估维度 | 题目形式 | 题目数 | 核心指标 | 主要局限 |
|-----------|----------|----------|--------|----------|----------|
| MMLU | 通识知识广度 | 多选题 | ~15K | 准确率 | 可能靠记忆、不测深层理解 |
| HumanEval | 代码生成 | 编程题 | 164 | pass@k | 偏算法题、不反映工程能力 |
| GSM8K | 数学推理 | 应用题 | 8.5K | 准确率 | 对 prompting 方式敏感 |
| HellaSwag | 常识推理 | 选择题 | ~10K | 准确率 | 干扰项设计影响难度 |
| TruthfulQA | 真实性/幻觉 | 开放+选择 | 817 | 真实率 + 信息量 | 题量小、偏西方文化语境 |

> **一个关键认知**：没有任何单一 Benchmark 能告诉你"这个模型适不适合你的业务"。Benchmark 是筛选起点，不是选型终点。

---

### 7.5.3 评估维度

实际评估时，建议按 5 个维度分别考察，而不是只看一个总分。

| 维度 | 考察什么 | 常用基准 / 方法 | 说明 |
|------|----------|-----------------|------|
| **知识** | 事实准确性、学科覆盖面 | MMLU、TruthfulQA、ARC | 模型"知道多少" |
| **推理** | 逻辑推理、数学、多步规划 | GSM8K、MATH、BBH、HellaSwag | 模型"能不能想明白" |
| **代码** | 代码生成、补全、调试 | HumanEval、MBPP、SWE-bench | 模型"能不能写对" |
| **安全** | 拒答有害请求、抗注入、偏见 | TruthfulQA、红队测试、自定义安全集 | 模型"会不会出格" |
| **多语言** | 非英语语言理解与生成 | MMMLU（多语言MMLU）、自建中文集 | 模型"中文行不行" |

#### 每个维度的注意事项

- **知识**：高分不等于不幻觉。MMLU 90+ 的模型仍然会在垂直领域编造细节。
- **推理**：GSM8K 满分不代表数学能力强，它只是小学难度。复杂推理看 MATH 和 BBH。
- **代码**：HumanEval 是最低门槛。真实工程能力建议用 SWE-bench 或自建 Repo 级评测。
- **安全**：公开基准远远不够。生产环境必须建自己的红队用例集。
- **多语言**：英文基准高分不能直接推断中文能力。中文场景要单独评测。

---

### 7.5.4 模型选型评估框架

选型不只是"哪个模型 Benchmark 分最高"，而是在三个约束之间找平衡。

#### 延迟 / 成本 / 质量三角

```text
        质量（Quality）
           ▲
          / \
         /   \
        /     \
       /  选型  \
      /  平衡点  \
     /           \
    ▼─────────────▼
延迟（Latency）   成本（Cost）
```

- **质量优先**：用最强模型（GPT-4 / Claude Opus），接受高延迟高成本。适合低频高价值场景（合同审查、医疗问答）。
- **延迟优先**：用小模型或本地模型，接受质量损失。适合实时交互场景（聊天、补全）。
- **成本优先**：用最便宜的模型 + 缓存 + 批处理，接受部分质量波动。适合大批量处理场景（分类、摘要）。

> 大多数生产系统不是选一个模型，而是按场景分级路由多个模型。

#### 选型评估 Pipeline

```text
1. 明确业务场景和核心指标
   ↓
2. 用公开 Benchmark 初筛候选模型（3-5 个）
   ↓
3. 构建业务评测集（200-500 条，覆盖核心场景 + 边界 case）
   ↓
4. 在业务评测集上跑候选模型，记录：
   - 质量分（准确率 / Judge 评分 / pass@k）
   - 平均延迟（首 token / 总耗时）
   - 单请求成本（input + output token 费用）
   ↓
5. 画出三角对比表，结合业务优先级决策
   ↓
6. 灰度上线验证，持续监控线上指标
```

#### 选型对比表模板

| 候选模型 | 业务评测分 | P95 延迟 | 单请求成本 | 中文能力 | 部署方式 | 综合建议 |
|----------|-----------|----------|-----------|----------|----------|----------|
| GPT-4o | 4.2/5 | 1.8s | ¥0.15 | 良好 | API | 质量首选 |
| Claude 3.5 Sonnet | 4.0/5 | 1.2s | ¥0.08 | 良好 | API | 性价比优 |
| GPT-4o-mini | 3.5/5 | 0.6s | ¥0.02 | 一般 | API | 低成本场景 |
| Qwen2-72B | 3.8/5 | 2.5s | ¥0.05 | 优秀 | 私有部署 | 中文 + 数据安全 |
| Llama 3-8B | 3.0/5 | 0.3s | GPU 成本 | 一般 | 本地 | 边缘/高并发 |

> 这张表的数字是示意，实际选型必须用你自己的业务数据跑出来。

#### 几个常见选型陷阱

1. **只看公开排行榜**：排行榜反映通用能力，不反映你的业务场景。
2. **只比价格不比质量**：便宜模型的错误率可能导致更高的返工和客诉成本。
3. **忽略延迟**：用户体验对首 token 延迟非常敏感，2s 和 5s 是完全不同的感受。
4. **忽略中文**：很多模型英文强但中文弱，中文场景必须单独评测。
5. **一次选型永久不变**：模型迭代很快，建议每季度重新跑一次评测。

---

### 7.5.5 小结

```text
评估体系 = Benchmark 初筛 + 多维度拆解 + 业务评测集 + 三角权衡 + 持续监控
```

- Benchmark 告诉你"模型大概处于什么水平"。
- 评估维度告诉你"模型在哪些方面强、哪些方面弱"。
- 业务评测集告诉你"模型在你的场景里到底行不行"。
- 三角权衡告诉你"在质量、延迟、成本之间怎么取舍"。
- 持续监控告诉你"上线后模型表现有没有退化"。

---

## 8. 面试题自查

### Q1: Transformer中的Self-Attention机制是什么？为什么要除以√d_k？

**答案**：
Self-Attention计算序列中每个token与其他所有token的相关性，公式为：
`Attention(Q, K, V) = softmax(QK^T / √d_k) * V`

**过程**：
1. 输入X通过三个权重矩阵W_q、W_k、W_v分别得到Q、K、V
2. 计算Q和K的点积，得到attention score
3. 除以√d_k进行缩放
4. softmax归一化得到attention权重
5. 与V加权求和得到输出

**为什么除以√d_k？**
当d_k较大时，点积的方差会很大（约为d_k），导致softmax的梯度非常小（接近one-hot），造成梯度消失。除以√d_k使方差回到1，保持梯度稳定。

---

### Q2: RoPE相比传统位置编码有什么优势？

**答案**：
RoPE（Rotary Position Embedding）通过旋转矩阵编码位置信息。

**优势**：
1. **编码相对位置**：Attention中只依赖相对位置(m-n)，而非绝对位置
2. **长度外推能力**：无需重新训练即可处理更长序列
3. **远程衰减**：自然实现远距离token的attention权重衰减
4. **计算高效**：只需对Q和K进行旋转，不增加参数

**原理**：将位置m的Q和位置n的K做内积时，结果只依赖于(m-n)，实现相对位置编码。

---

### Q3: KV Cache的作用是什么？内存占用如何估算？

**答案**：
**作用**：自回归生成时，缓存历史token的K、V向量，避免重复计算。

没有KV Cache时，生成第n个token需要计算前n-1个token的KV；有KV Cache后，只需计算当前token的KV，然后与缓存拼接。

**内存估算**：
```
内存 = 2 × num_layers × batch_size × seq_len × num_heads × head_dim × dtype_size
```

**示例**（LLaMA-7B，seq_len=4096，batch=1，fp16）：
= 2 × 32 × 1 × 4096 × 32 × 128 × 2 bytes ≈ 2GB

**优化方案**：
- **PagedAttention**：分页管理KV Cache，减少内存碎片
- **GQA**：分组共享K、V，减少缓存量
- **量化**：对KV Cache做int8量化

---

### Q4: Temperature和Top-p参数如何影响生成效果？

**答案**：
**Temperature**（温度）：
- 控制softmax分布的"尖锐度"
- T > 1：分布更平坦，输出更随机
- T < 1：分布更尖锐，输出更确定
- T = 0：等于贪婪解码（argmax）

**Top-p**（核采样）：
- 只从累积概率达到p的最小token集合中采样
- p = 0.9：忽略尾部10%概率的token
- 动态调整候选集大小

**使用建议**：
| 场景 | Temperature | Top-p |
|------|-------------|-------|
| 代码生成 | 0 ~ 0.2 | 0.95 |
| 创意写作 | 0.8 ~ 1.2 | 0.9 |
| 数学推理 | 0 | 1.0 |

---

### Q5: RLHF和DPO有什么区别？各自的优缺点？

**答案**：

| 维度 | RLHF | DPO |
|------|------|-----|
| 训练阶段 | SFT → 训练RM → PPO | SFT → DPO |
| 奖励模型 | 需要单独训练 | 不需要，隐式学习 |
| 训练稳定性 | 较差，PPO需要调参 | 稳定，类似分类任务 |
| 计算成本 | 高（4个模型） | 低（2个模型） |
| 效果天花板 | 更高 | 接近RLHF |

**RLHF优点**：可以持续迭代优化，效果上限更高
**RLHF缺点**：训练复杂，需要同时维护策略模型、参考模型、奖励模型、价值模型

**DPO优点**：简单稳定，直接从偏好数据学习，无需强化学习
**DPO缺点**：依赖高质量的偏好数据对

---

### Q6: Agent的ReAct模式是什么？与Plan-and-Execute有何不同？

**答案**：
**ReAct（Reasoning + Acting）**：
```
Thought: 分析当前状态
Action: 调用工具
Observation: 观察结果
... (循环直到完成)
```
特点：边思考边行动，每步决策基于最新观察

**Plan-and-Execute**：
```
1. 先制定完整计划（N个步骤）
2. 按计划依次执行每一步
3. 汇总结果
```
特点：先规划后执行，适合明确的多步任务

**对比**：
| 特性 | ReAct | Plan-and-Execute |
|------|-------|------------------|
| 灵活性 | 高，可动态调整 | 低，按计划执行 |
| 可解释性 | 一般 | 好（有明确计划） |
| 错误恢复 | 容易 | 需要重新规划 |
| 适用场景 | 探索性任务 | 明确的多步任务 |

---

### Q7: RAG的核心流程是什么？有哪些常见优化技术？

**答案**：
**核心流程**：
1. **索引构建**：文档分块 → Embedding → 存入向量库
2. **检索**：用户Query → Embedding → 向量相似度检索
3. **增强生成**：检索结果 + Query → 构造Prompt → LLM生成

**优化技术**：

| 阶段 | 优化技术 | 说明 |
|------|----------|------|
| 索引 | 语义分块 | 按语义边界切分，保持完整性 |
| 检索 | Hybrid Search | 向量检索 + BM25关键词检索 |
| 检索 | Query Expansion | 扩展查询词，提高召回 |
| 重排 | Cross-Encoder Reranking | 精排提高准确率 |
| 生成 | 引用溯源 | 标注答案来源，可验证 |

---

### Q8: 从模型原理看，LLM 为什么会产生幻觉？基础缓解手段有哪些？

**答案**：
**幻觉（Hallucination）**：LLM 生成了语言上自洽、但事实或推理上错误的内容。

**为什么会发生**：
1. **训练目标决定**：LLM 本质是在做 next-token prediction，优先追求“像真的”，不天然等于“是真的”。
2. **参数知识有边界**：训练数据有时效性、覆盖面和噪声限制，模型不可能内置所有最新事实。
3. **上下文不足或被污染**：问题本身模糊、检索材料缺失、上下文冲突，都会让模型补全错误答案。
4. **复杂推理链不稳定**：多跳推理、数字计算、实体关系题更容易在中间步骤出错。

**基础缓解手段**：
1. **RAG 注入外部事实**：让模型优先基于给定资料回答，而不是只靠参数记忆。
2. **工具调用**：需要实时信息、精确计算、结构化查询时，交给搜索、数据库或计算器。
3. **Prompt 约束**：明确“不确定就说不知道”“必须给出依据”。
4. **一致性与校验**：对关键问题做多次采样、规则校验或交叉验证。
5. **输出溯源**：要求引用来源，方便人工判断答案是否可信。

这题在本篇重点是“为什么会幻觉、有哪些基础缓解方法”；如果你要回答线上治理、门禁和回滚，优先看 `03-AI工程化实践`。

---

### Q9: Function Calling的工作原理？如何设计好的工具？

**答案**：
**工作原理**：
1. 用户问题 + 工具定义（JSON Schema）→ LLM
2. LLM判断是否需要调用工具，输出工具名和参数
3. 执行工具，获取结果
4. 结果回传给LLM，生成最终答案

**工具设计原则**：

1. **单一职责**：一个工具只做一件事
2. **清晰描述**：description要准确描述功能和参数
3. **参数验证**：校验必填项、类型、范围
4. **错误处理**：返回结构化错误信息，便于LLM理解
5. **幂等性**：尽量设计幂等操作，避免重复调用副作用

**示例**：
```python
{
    "name": "search_order",
    "description": "根据订单号查询订单状态和物流信息",
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "订单号，格式如：ORD202401010001"
            }
        },
        "required": ["order_id"]
    }
}
```

---

### Q10: Embedding模型选型要考虑哪些因素？

**答案**：

| 因素 | 考量点 |
|------|--------|
| **维度** | 高维度精度高但存储/计算成本高 |
| **语言支持** | 中文场景优先选BGE、M3E |
| **上下文长度** | 长文本选支持8K的模型如Jina |
| **开源/闭源** | 私有部署选开源，追求效果选OpenAI |
| **领域适配** | 特定领域需要微调 |
| **推理延迟** | 实时场景选轻量模型 |

**常用模型**：
- **OpenAI text-embedding-3-small/large**：通用高质量
- **BGE-large-zh**：中文场景首选
- **M3E-base**：轻量，资源受限场景
- **Jina-embeddings-v2**：长上下文（8K）

---

### Q11: Multi-Head Attention为什么要分多个头？

**答案**：
**原因**：
1. **捕获不同子空间的特征**：每个head可以学习不同的attention模式（如语法、语义、位置关系）
2. **增加模型容量**：总参数量不变，但表达能力更强
3. **并行计算**：多个head可以并行计算，效率更高

**计算过程**：
```
输入: X (seq_len, d_model)
分成h个head，每个head维度 d_k = d_model / h

对每个head:
  Q_i = X @ W_q_i
  K_i = X @ W_k_i  
  V_i = X @ W_v_i
  head_i = Attention(Q_i, K_i, V_i)

拼接: Concat(head_1, ..., head_h) @ W_o
```

**典型配置**：GPT-3有96个head，每个head维度128

---

### Q12: Chunking策略对RAG效果有什么影响？

**答案**：
**影响**：
- **块太大**：检索精度下降，噪声多
- **块太小**：语义不完整，上下文丢失
- **重叠不够**：边界信息丢失

**常用策略**：

1. **固定大小分块**
   - chunk_size=500, overlap=50
   - 简单快速，但可能切断语义

2. **语义分块**
   - 基于Embedding相似度找断点
   - 保持语义完整，但计算成本高

3. **结构化分块**
   - 按文档结构（章节、段落）切分
   - 需要文档格式支持

4. **句子/段落分块**
   - 以句号/段落为边界
   - 语义完整但长度不一

**最佳实践**：
- 保留上下文：在chunk头部加入父章节标题
- 适度重叠：10-20%重叠防止边界信息丢失
- 元数据：记录来源、页码便于溯源

---

### Q13: 从任务编排视角，如何评估一个 Agent 系统的效果？

**答案**：
这题在基础文档里，重点不是通用产品指标，而是看 Agent 作为“会规划、会调工具的系统”是否真的比单轮问答更有效。

| 评估维度 | 指标 | 说明 |
|----------|------|------|
| **任务完成率** | Success Rate | 是否真正完成目标任务 |
| **步骤效率** | Avg Steps | 完成任务平均用了多少步 |
| **工具使用** | Tool Accuracy | 工具选择和参数是否正确 |
| **错误恢复** | Recovery Rate | 调用失败后能否自恢复 |
| **轨迹质量** | Trajectory Quality | 中间推理和动作链是否合理 |
| **成本** | Token/Tool Cost | 完成任务的总成本 |

**评估方法**：
1. **任务集评测**：设计带 ground truth 的真实任务，不只评估最终答案，也评估中间轨迹。
2. **人工审查**：看工具是否乱用、步骤是否冗余、失败是否可解释。
3. **A/B 对比**：和单 Agent、无工具版本或旧策略做对照。

如果要回答线上质量看板、SLO、成本门禁和发布灰度，那属于工程治理视角，优先看 `03-AI工程化实践.md`。

---

### Q14: 什么是 Prompt Injection？它在原理上为什么危险？基础防御思路有哪些？

**答案**：
**Prompt Injection**：攻击者把恶意指令伪装进用户输入、网页内容、文档内容或工具返回结果里，诱导 LLM 偏离原始系统目标。

**它为什么危险**：
1. **自然语言指令和数据共处一个上下文**：模型并不天然知道哪段话是“可执行指令”，哪段只是“待处理内容”。
2. **模型倾向服从最近、最强的文本模式**：如果隔离做得不好，恶意内容会和系统提示竞争控制权。
3. **一旦连上工具，风险会放大**：问题不再只是“答错”，而可能变成越权查询、误调用外部系统、泄露敏感信息。

**基础防御思路**：
1. **角色隔离**：系统指令、开发指令、用户输入、外部内容分区放置，避免混写。
2. **明确边界**：把外部文档当“数据”而不是“指令”，必要时加分隔符和引用包装。
3. **输出约束**：限制结构化输出和敏感字段，减少模型自由发挥空间。
4. **最小权限**：工具默认只给读权限，高风险写操作要求二次确认。
5. **高风险输入识别**：对明显的越权、绕过、提权模式做拦截或人工复核。

这题在本篇重点是“为什么会被注入、基础防御原则是什么”；如果要回答生产上的多层防线和治理闭环，优先看 `03-AI工程化实践`。

---

### Q15: 对比Hybrid Search中向量检索和关键词检索的优缺点

**答案**：

| 特性 | 向量检索 | 关键词检索（BM25） |
|------|----------|-------------------|
| **原理** | 语义相似度 | 词频统计匹配 |
| **优点** | 理解语义，同义词友好 | 精确匹配，专有名词准 |
| **缺点** | 专有名词可能召回偏 | 无法理解语义 |
| **延迟** | 较高（需Embedding） | 较低 |
| **适用** | 语义理解场景 | 精确搜索场景 |

**Hybrid Search**：
```python
# 典型配置：向量0.7 + 关键词0.3
ensemble = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.7, 0.3]
)
```

**适用场景**：
- 专业领域（医疗、法律）：关键词权重高
- 通用对话：向量权重高

---

### Q16: 解释LLM的涌现能力（Emergent Abilities）

**答案**：
**涌现能力**：模型在达到一定规模后，突然获得的能力，且在较小规模时几乎不存在。

**典型涌现能力**：
1. **思维链推理（CoT）**：>100B参数后出现
2. **In-Context Learning**：从示例中学习新任务
3. **指令跟随**：理解并执行复杂指令
4. **代码生成**：生成可执行代码

**可能的解释**：
- 模型容量积累到临界点
- 复杂能力是简单能力的组合
- 评估指标的非线性（准确率突变）

**工程意义**：
- 不能简单外推小模型的能力
- 大模型可能具备未被发现的能力
- 提示工程可以激发潜在能力

---

### Q17: 上下文窗口变长后，为什么效果不一定线性变好？

**答案**：
窗口变长提升的是“可容纳信息量”，不等于“可有效利用信息量”。常见瓶颈有：
1. **注意力稀释**：关键信息被长上下文噪声淹没。
2. **检索质量不足**：喂进去的是低相关内容，反而干扰生成。
3. **位置偏置**：中间信息更容易被忽略（Lost in the Middle）。

工程上应优先做“高质量召回 + 上下文重排 + 结构化提示”，而不是只盲目堆上下文长度。

---

### Q18: Function Calling 场景里，如何降低“错工具、错参数”带来的失败率？

**答案**：
可以从三层治理：
1. **工具层**：名称互斥、描述清晰、参数 schema 严格校验。
2. **策略层**：先意图分类再路由工具，减少大模型直接猜测。
3. **恢复层**：失败返回结构化错误（缺字段/类型错/权限不足），触发有限重试。

关键是把调用失败当成可观测事件，而不是“模型偶发失误”。

---

### Q19: Multi-Agent 一定比 Single-Agent 更好吗？

**答案**：
不一定。Multi-Agent 的优势是职责分离和并行处理，但代价是编排复杂度、上下文传递成本和故障面扩大。判断标准：
1. 任务是否天然可拆分且相互独立。
2. 子任务是否需要不同工具/不同策略。
3. 团队是否具备编排与可观测能力。

任务边界不清时，强行多代理通常只会增加成本和排障难度。

---

### Q20: Agent 系统如何设计“可回滚”而不是“只能升级”？

**答案**：
建议把回滚能力设计成默认能力：
1. **版本化**：Prompt、工具 schema、路由策略都要有版本号。
2. **灰度发布**：按流量或用户分组推进，指标异常自动回退。
3. **结果可追溯**：每次决策保留 trace，能定位是模型、工具还是策略导致异常。
4. **降级路径**：失败时切换到简化流程或人工接管。

没有回滚能力的 Agent 发布，本质上是把风险后移到线上。

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


---
