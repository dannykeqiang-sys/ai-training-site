# Embedding、多模态与 RLHF

本篇介绍 LLM 的三个延伸维度：Embedding 是让模型理解语义的基础技术（也是 RAG 的核心）；RLHF 是让模型变得"好用"的训练方法；多模态是模型能力的扩展方向。理解这三者，能帮助你建立从"模型能做什么"到"模型怎么做得更好"的完整认知。

## 一、Embedding（向量嵌入）

### 1.1 定义

计算机只认数字，所以要把文本变成数字。但 Tokenizer 生成的 Token ID 只是编号，没有语义含义（比如 ID=1024 和 ID=1025 不一定代表意思相近的词）。

**Embedding（向量嵌入）** 是把每个 Token（或整段文本）变成一组能表达语义的浮点数向量——语义越接近的词，向量越接近。映射后，语义相似的内容在向量空间中距离更近。[¹](#ref1) [²](#ref2)

```
"国王" → [0.21, -0.45, 0.89, ..., 0.12]  (1536 维向量)
"王后" → [0.19, -0.43, 0.91, ..., 0.15]  (距离很近)
"汽车" → [-0.67, 0.33, -0.12, ..., 0.78] (距离很远)
```

> **经典语义运算**：`vector("国王") - vector("男人") + vector("女人")` 的结果与 `vector("王后")` 非常接近。这说明向量空间确实捕捉到了"性别"这个语义关系——Embedding 不只是数字编号，而是真正编码了语义。

### 1.2 为什么需要 Embedding

![Embedding 将语义映射到向量空间](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/embedding-vector-space_1784688698.png)

计算机无法直接理解"猫"和"狗"在语义上比"猫"和"飞机"更接近。Embedding 的作用就是：

| 需求 | Embedding 如何解决 |
|---|---|
| **语义理解** | 将语义关系编码为向量间的数学距离 |
| **相似度计算** | 余弦相似度（cosine similarity）衡量两段文本的语义接近程度 |
| **高效检索** | 向量数据库可在百万级文档中毫秒级找到最相关的内容 |
| **降维表示** | 将非结构化数据转为统一格式，便于下游任务处理 |

### 1.3 Embedding 在 LLM 中的两重角色

**角色一：模型内部的 Token Embedding**

每个 Token ID 进入模型时，首先被一个 Embedding 层转换为连续向量，模型的所有计算都在这个向量空间中进行。

**角色二：外部的文本 Embedding（用于 RAG 等）**

通过专门的 Embedding 模型（如 OpenAI text-embedding-3-small、通义 text-embedding-v3）将整段文本编码为一个向量，用于：[²](#ref2) [³](#ref3)
- RAG 中的语义检索
- 文本分类/聚类
- 推荐系统
- 异常检测

### 1.4 Embedding 在 RAG 中的关键地位

```
文档 → 分块(Chunking) → Embedding 模型 → 向量 → 存入向量数据库
                                                    ↑
用户问题 → Embedding 模型 → 向量 → 相似度检索 ────┘
                                    ↓
                            Top-K 相关片段 → 拼入 Prompt → LLM 生成答案
```

向量化后，"语义相似的文本距离更近"这一特性使得 RAG 能准确找到与用户问题最相关的知识片段。[¹](#ref1)

### 1.5 主流 Embedding 模型一览

| 模型 | 厂商 | 向量维度 | 特点 |
|---|---|---|---|
| text-embedding-3-small | OpenAI | 1536 | 性价比高，通用场景 |
| text-embedding-3-large | OpenAI | 3072 | 精度更高，适合高要求检索 |
| text-embedding-v3 | 阿里（通义） | 1024/2048 | 中文优化，国内首选 |
| bge-large-zh | BAAI（智源） | 1024 | 开源，中文表现优异 |
| Cohere embed-v3 | Cohere | 1024 | 多语言支持好 |

### 1.6 相似度计算方法

两个向量之间的"距离"常用以下方法度量：

| 方法 | 公式含义 | 适用场景 |
|---|---|---|
| **余弦相似度** | 衡量方向一致性（-1 到 1） | 最常用，适合文本语义匹配 |
| **欧氏距离** | 衡量空间中的绝对距离 | 适合需要考虑向量长度的场景 |
| **点积** | 方向 + 长度综合考量 | 部分向量数据库默认使用 |

## 二、RLHF（基于人类反馈的强化学习）

### 2.1 定义

**RLHF（Reinforcement Learning from Human Feedback）** 是一种利用人类偏好数据，通过强化学习来优化语言模型行为的训练方法。它是让 LLM 从"能说话"进化到"说人话"的关键技术。[⁴](#ref4) [⁵](#ref5)

### 2.2 为什么需要 RLHF

预训练后的 LLM 虽然具备强大的语言能力，但存在严重问题：
- 可能输出有害/有毒内容
- 不遵循用户指令（答非所问）
- 输出冗长、重复、格式混乱
- 表现出不受控的偏见

简单的监督微调（SFT）可以部分解决，但人类对"什么是好回答"的判断很难用简单规则描述。RLHF 的核心洞察是：**让人类直接告诉模型"哪个回答更好"，比定义"好回答的规则"容易得多。**[⁴](#ref4)

### 2.3 RLHF 三步流程

![RLHF 三步流程示意](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/rlhf-three-steps_1784688704.png)

OpenAI 在 InstructGPT 论文中系统描述了 RLHF 的三个阶段：[⁴](#ref4) [⁵](#ref5)

> **通俗类比**：可以类比为培训新员工：(1) 先看老员工怎么做（SFT）；(2) 让主管学会打分（训练 Reward Model）；(3) 让新员工反复练习，主管给反馈（PPO 优化）。

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: 监督微调（SFT）                                       │
│   收集人工编写的高质量问答对 → 微调基础模型                       │
├─────────────────────────────────────────────────────────────┤
│ Step 2: 训练奖励模型（Reward Model）                           │
│   对同一问题，模型生成多个回答 → 人类标注员对回答排序             │
│   → 用排序数据训练一个"打分器"（RM），学会预测人类偏好              │
├─────────────────────────────────────────────────────────────┤
│ Step 3: 强化学习优化（PPO）                                    │
│   用 RM 作为奖励信号 → 通过 PPO 算法优化 LLM                    │
│   → 模型学会生成"人类更喜欢"的回答                               │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 RLHF 的效果

InstructGPT（1.3B 参数 + RLHF）在人类评估中优于 GPT-3（175B 参数无 RLHF），证明：**对齐方法的重要性可以超过参数规模。**[⁴](#ref4)

### 2.5 RLHF 的后继技术

| 技术 | 改进点 |
|---|---|
| **DPO（Direct Preference Optimization）** | 跳过 RM 训练，直接用偏好数据优化 LLM，更简单稳定 |
| **GRPO（Group Relative Policy Optimization）** | DeepSeek-R1 使用，无需 RM，通过组内相对排名计算奖励 |
| **Constitutional AI（CAI）** | Anthropic 提出，用 AI 自身反馈替代部分人类标注 |

> **了解 RLHF 对你使用 LLM 的意义**：当你觉得模型"太保守"或"太啰嗦"时，这其实是 RLHF 对齐的结果。不同模型的 RLHF 策略不同，导致"性格"差异——例如 Claude 倾向于谨慎和详尽，GPT-4 倾向于直接和简洁。理解这一点，有助于你根据任务需求选择合适的模型。

## 三、多模态 AI 模型

### 3.1 定义

**多模态 AI 模型（Multimodal AI Model）** 是能够同时处理和理解**多种类型输入数据**（文本、图像、音频、视频等）的 AI 模型。[⁶](#ref6) [⁷](#ref7)

> 类比：人类天生是"多模态"的——我们同时用眼睛看、耳朵听、嘴巴说来理解世界。多模态 AI 试图赋予模型类似的能力。

### 3.2 从单模态到多模态的演进

| 阶段 | 代表 | 能力 |
|---|---|---|
| 纯文本 LLM | GPT-3, LLaMA | 只能处理文字 |
| 视觉语言模型 | GPT-4V, Qwen-VL | 理解图片 + 文字 |
| 全模态模型 | GPT-4o, Gemini 1.5 | 文字 + 图片 + 音频 + 视频，统一处理 |

### 3.3 多模态的技术实现思路

1. **各模态独立编码**：使用专门的编码器将图片/音频转换为向量表示
   - 图片：Vision Encoder（如 ViT）
   - 音频：Audio Encoder（如 Whisper）
2. **统一向量空间**：将不同模态的向量映射到同一个语义空间
3. **Transformer 统一处理**：在统一空间中用同一个 Transformer 做跨模态推理

### 3.4 实际应用场景

| 场景 | 输入模态 | 输出模态 | 示例 |
|---|---|---|---|
| 图片理解 | 图片 + 文字 | 文字 | "这张图里有什么？" |
| 图片生成 | 文字 | 图片 | DALL-E、Midjourney |
| 语音对话 | 音频 | 音频/文字 | GPT-4o 语音模式 |
| 视频理解 | 视频 + 文字 | 文字 | "总结这段视频的内容" |
| 文档理解 | PDF/图表 + 文字 | 文字 | 分析合同扫描件 |

### 3.5 多模态模型的 Token 消耗

图片也会消耗 Token。例如通义 qwen-vl 模型中，**每 28x28 像素约等于 1 个 Token**。一张 1024x1024 的图片约消耗 1300 Tokens。[¹](#ref1)

## 常见误解


- ❌ "Embedding 就是分词"——分词是 Tokenizer 的工作，Embedding 是将 Token 映射为向量
- ❌ "Embedding 只用于训练阶段"——推理时同样需要 Embedding

- ❌ "RLHF 是一种预训练方法"——它是预训练之后的对齐/微调阶段
- ❌ "RLHF 不需要人类参与"——名字里的 HF 就是 Human Feedback
- ❌ "RLHF 让模型学习新知识"——RLHF 主要调整模型的行为偏好，不注入新事实知识

- ❌ "多模态模型只能输入文字，输出图片"——多模态指输入输出都可以是多种形式
- ❌ "多模态就是多语言"——多模态指数据形式多样，多语言指自然语言多样，两者不同
- ❌ "所有 LLM 都是多模态的"——纯文本 LLM（如早期 GPT-3）不具备多模态能力

## 参考资料

<a id="ref1"></a>**[1]** 钱红凯(趋势). 《AI实践之路：LLM、RAG、MCP、Agent、A2A》RAG 及图片识别章节. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*

<a id="ref2"></a>**[2]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》第 5 节 RAG. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref3"></a>**[3]** OpenAI. *Vector Embeddings*（官方 Embeddings 指南）. https://developers.openai.com/api/docs/guides/embeddings

<a id="ref4"></a>**[4]** Ouyang, L. et al. *Training language models to follow instructions with human feedback* (InstructGPT). arXiv, 2022. https://arxiv.org/abs/2203.02155

<a id="ref5"></a>**[5]** Hugging Face. *Illustrating Reinforcement Learning from Human Feedback (RLHF)*. 2022. https://huggingface.co/blog/rlhf

<a id="ref6"></a>**[6]** OpenAI. *Hello GPT-4o*（多模态能力介绍）. 2024-05-13. https://openai.com/index/hello-gpt-4o/

<a id="ref7"></a>**[7]** 刘立成(文辺). 《关于 AI 的一些新名词和概念》LLM 大语言模型分类. ATA, 2025-04-18. https://ata.atatech.org/articles/11020405255 *（内网）*
