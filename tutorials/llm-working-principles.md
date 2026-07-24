# LLM 工作原理

本篇涵盖大语言模型从底层架构到实际使用的核心概念：模型如何生成文本（文字接龙原理）、底层引擎是什么（Transformer 架构）、如何与模型交互（Prompt/Completion）、以及模型运行时的性能指标（Inference）。理解这些概念，才能对 AI 工具建立正确的认知和预期。

## 一、Generative AI 与 AIGC：先厘清两个名词

**Generative AI（生成式人工智能）** 是指能够根据输入创造新内容（文本、图片、音视频、代码等）的 AI 系统。[¹](#ref1)

**AIGC（AI-Generated Content）** 直译为"人工智能生成内容"，是 Generative AI 的产出物的统称——换句话说，Generative AI 是技术，AIGC 是这个技术生成的内容。

> 常见误解：把 AIGC 理解为"一种新技术"。实际上它只是内容生产范式的描述，不特指某种算法。

## 二、大模型的工作原理——"文字接龙"

![自回归“文字接龙”逐 Token 生成](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/llm-autoregressive_1784688564.png)

大模型的本质是一个**自回归语言模型（Autoregressive LM）**：给定前面所有 Token，预测下一个 Token 的概率分布，选出概率最高（或按策略采样）的那个 Token 输出，然后把它追加到输入序列末尾，继续预测……如此循环直到遇到结束标记。[¹](#ref1) [²](#ref2)

```
输入: "你好吗？"
→ 预测下一个 Token: "我" (概率最高)
→ 输入变成: "你好吗？我"
→ 预测下一个 Token: "很" 
→ ...
→ 直到输出结束标记 <EOS>
```

ATA 文章用了一个非常直观的比喻：**大模型就是一个极其复杂的"文字接龙"游戏。**[¹](#ref1)

### 2.1 自回归生成的关键特征

1. **逐 Token 生成**：模型每次只输出一个 Token，不会"一口气想好整段话"
2. **单向性**：已输出的 Token 不会被模型回头修改（除非应用层做了 retry）
3. **概率性**：输出不是唯一确定的，同样的 Prompt 可能得到不同 Completion（取决于 Temperature 等采样参数）
4. **无状态**：模型本身没有记忆，"多轮对话"靠把历史消息全部重新输入来实现

### 2.2 大模型的训练流程概览

![大模型训练三阶段：预训练 → SFT → RLHF](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/llm-training-pipeline_1784688576.png)

理解工作原理也需了解模型是如何被训练出来的：

| 阶段 | 过程 | 数据规模 |
|---|---|---|
| **Pre-training（预训练）** | 在海量互联网文本上做"下一个 Token 预测"，学习语言知识 | TB 级文本 |
| **SFT（监督微调）** | 用人工编写的高质量问答对进行微调，学习"对话"格式 | 数万~数十万条 |
| **RLHF/DPO（对齐）** | 通过人类偏好反馈，让模型输出更安全、有用、符合预期 | 数万条偏好对 |

预训练阶段消耗最多的资源（数千 GPU 训练数周到数月），但它只是让模型学会了"接龙"能力。真正让模型变得"好用"的是后面的 SFT 和 RLHF 阶段。[⁵](#ref5)

## 三、Prompt 与 Completion

在与 LLM 交互时，有一对核心概念：[¹](#ref1) [²](#ref2)

| 术语 | 含义 | 类比 |
|---|---|---|
| **Prompt** | 用户输入给模型的文本（问题/指令/上下文） | 考试题目 |
| **Completion** | 模型根据 Prompt 生成的输出文本 | 考生答案 |

在 Chat API 中，Prompt 进一步细分为：
- **System Prompt**：开发者设定的"元指令"，定义角色和规则
- **User Prompt**：用户的具体问题或指令
- **Assistant**：模型之前的回复（多轮对话场景）

> 注意：Completion 是模型**逐 Token 生成**的，不是一次性"想好"再输出。

### 3.1 实际 API 调用中的体现

```python
from openai import OpenAI
client = OpenAI(api_key="...", base_url="https://dashscope.aliyuncs.com/compatible-mode/v1")

response = client.chat.completions.create(
    model="qwen-plus",
    messages=[
        {"role": "system", "content": "你是一位诗人"},  # System Prompt
        {"role": "user", "content": "写一首关于秋天的五言绝句"}  # User Prompt
    ]
)
# response.choices[0].message.content → 这就是 Completion
```

在流式输出模式下，Completion 是一个 Token 一个 Token"流"出来的，前端展现为"打字机效果"。[²](#ref2)

我们已经知道模型在做什么（文字接龙），接下来看看它是怎么做到的（Transformer 架构）。

## 四、Transformer 架构——Encoder 与 Decoder

现代 LLM 的底层引擎都基于 **Transformer** 架构，由 Google 团队在 2017 年论文《Attention Is All You Need》中提出。[³](#ref3)

### 4.1 高层结构：两大核心组件

原始 Transformer 由两大组件构成：

| 组件 | 职责 | 核心机制 |
|---|---|---|
| **Encoder（编码器）** | 读取并理解输入序列，生成上下文表示 | Self-Attention（自注意力） |
| **Decoder（解码器）** | 基于 Encoder 输出 + 已生成 Token，逐步生成输出序列 | Masked Self-Attention + Cross-Attention |

两者都由多层堆叠的注意力模块组成，但工作方式不同：[³](#ref3) [⁴](#ref4)

- **Encoder**：双向注意力，能同时看到序列中所有位置（适合"理解"任务）
- **Decoder**：因果注意力（Causal/Masked），只能看到当前位置及之前的 Token（适合"生成"任务）

> Encoder 就像一个阅读理解者，可以反复看整篇文章来理解意思；Decoder 就像一个翻译者，只能看着已经写出的译文逐词翻译。

### 4.2 三种变体架构

不同的大模型根据任务侧重选择了不同的 Transformer 变体：

| 架构类型 | 代表模型 | 适合任务 |
|---|---|---|
| **Encoder-only** | BERT, RoBERTa | 文本分类、NER、语义理解 |
| **Decoder-only** | GPT 系列, LLaMA, Qwen, Claude | 文本生成、对话、代码 |
| **Encoder-Decoder** | T5, BART, 原始 Transformer | 翻译、摘要、Seq2Seq 任务 |

> 当前主流的 LLM（GPT-4、Claude、Qwen、DeepSeek 等）几乎全部采用 **Decoder-only** 架构。

### 4.3 Self-Attention 是核心

无论 Encoder 还是 Decoder，核心创新都是 **Self-Attention（自注意力）** 机制——让模型在处理每个 Token 时，能"看到"并加权关注序列中所有（或允许范围内的）其他 Token，从而捕获长距离依赖关系。[³](#ref3)

简化公式：
```
Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

其中 Q（Query）、K（Key）、V（Value）均由输入经过线性变换得到。

> 直觉解读：Q 就像在问"我在找什么信息"，K 是"我这里有什么信息"，V 是"我实际提供什么信息"。模型通过比较 Q 和 K 的相似度来决定关注哪些位置。

## 五、Inference（推理）

在 LLM 语境中，**Inference（推理）** 指的是：**使用已训练好的模型，对新输入进行前向计算并生成输出的过程。**[¹](#ref1)

> 这里的"推理"是工程意义上的"模型推理"（forward pass），**不是**逻辑意义上的"因果推理/逻辑推理"。

| 阶段 | 含义 | 资源消耗 |
|---|---|---|
| **Training（训练）** | 用海量数据调整模型参数（权重） | 极高（数千 GPU 运行数周） |
| **Inference（推理）** | 用训练好的模型处理新的输入并生成输出 | 相对较低（API 调用即可） |

日常使用 ChatGPT、通义千问等对话时，每次提问-回答都是一次 Inference 过程。API 按 Token 计费，本质上就是在为 Inference 的算力买单。

### 5.1 Inference 的关键性能指标

| 指标 | 含义 | 典型值 |
|---|---|---|
| **TTFT（Time to First Token）** | 从发送请求到收到第一个 Token 的延迟 | 0.5–3 秒 |
| **TPS（Tokens Per Second）** | 模型每秒生成的 Token 数 | 30–100 tokens/s |
| **Throughput** | 单位时间内系统能处理的总请求数 | 取决于部署规模 |

> 当你向 ChatGPT 提问后，通常要等 1-3 秒才开始出字——这就是 TTFT。之后文字以每秒 30-100 个 Token 的速度流式输出——这就是 TPS。

### 5.2 Inference 与 Training 的核心区别

| 维度 | Training | Inference |
|---|---|---|
| 目的 | 学习/调整模型参数 | 使用固定参数生成输出 |
| 计算方向 | 前向 + 反向传播 | 仅前向传播 |
| 数据 | 训练集（已标注） | 用户的实时输入 |
| 频率 | 一次（或少数几次） | 持续进行（每次用户交互） |
| 硬件需求 | 顶配 GPU 集群 | 可用较低配置 GPU / 专用推理芯片 |

## 常见误解

**关于 Prompt/Completion 的命名**：
- ❌ "Input / Output"——虽然意思接近，但不是 LLM 领域的标准术语
- ❌ "Query / Response"——这是搜索引擎的术语

**关于 Transformer 核心组件**：
- ❌ "CNN 和 RNN"——Transformer 正是为了替代 RNN 而提出的
- ❌ "Attention 和 Feed-Forward"——这是 Transformer 内部子层结构，不是高层组件

**关于 Generative AI 工作原理**：
- ❌ "从数据库中检索答案"——那是搜索引擎/RAG，不是生成
- ❌ "真正理解了语义再回答"——模型没有"真正理解"，只是统计模式

**关于 Inference 推理**：
- ❌ "训练模型参数的过程"——那是 Training
- ❌ "模型进行逻辑因果推理"——那是 Reasoning，不是 Inference

**关于 AIGC**：
- ❌ "AI General Computing"——不存在这个缩写
- ❌ "一种特定的模型架构"——AIGC 是内容产出概念，不是技术架构


## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** 钱红凯(趋势). 《AI实践之路：LLM、RAG、MCP、Agent、A2A，掌握AI相关基础原理》. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*

<a id="ref3"></a>**[3]** Vaswani, A. et al. *Attention Is All You Need*. NeurIPS 2017. https://arxiv.org/abs/1706.03762

<a id="ref4"></a>**[4]** Wikipedia. *Attention Is All You Need*. https://en.wikipedia.org/wiki/Attention_Is_All_You_Need

<a id="ref5"></a>**[5]** 陈熳熳(念珷). 《AI基础知识(1)：概念篇》. ATA, 2025-06-26. https://ata.atatech.org/articles/11020427672 *（内网）*
