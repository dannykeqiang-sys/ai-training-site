# Temperature 与上下文窗口

Temperature 和 Context Window 是使用 LLM API 时最常遇到的两个参数概念。前者决定模型"怎么说"（输出的随机性），后者决定模型"能记住多少"（处理的容量上限）。理解这两个概念，是合理调用 LLM API、构建稳定应用的基础。

## 一、Temperature 参数

![Temperature 对词概率分布的影响](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/temperature-distribution_1784688564.png)

### 1.1 定义

**Temperature** 是调用 LLM API 时的一个采样参数，用于控制模型输出的**随机性/创造性程度**。[¹](#ref1) [²](#ref2)

直觉理解：Temperature 就像一个"创造力旋钮"——

| Temperature 值 | 效果 | 适用场景 |
|---|---|---|
| **0**（或极低） | 几乎总是选概率最高的 Token，输出**确定性强**、可复现 | 数据提取、分类、事实问答 |
| **0.3–0.7** | 适度随机，兼顾准确与多样 | 通用对话、代码生成 |
| **1.0** | 标准采样（模型原始概率分布） | 一般创作 |
| **>1.0**（如 1.5–2.0） | 低概率 Token 被选中的可能性大幅增加，输出**更随机/更有创意** | 头脑风暴、诗歌创作 |

### 1.2 数学原理

模型输出的是每个候选 Token 的 logit（未归一化分数）。生成概率时需经过 softmax：

> **通俗理解 logit**：logit 可以理解为模型对每个候选 Token 的"原始打分"——分数越高，模型越倾向于选择这个 Token。Temperature 的作用就是调整这些分数之间的差距：Temperature 低时，高分和低分的差距被放大，模型几乎只选最高分的 Token；Temperature 高时，差距被缩小，低分 Token 也有更多机会被选中。

```
P(token_i) = exp(logit_i / T) / Σ exp(logit_j / T)
```

其中 **T** 就是 Temperature：[²](#ref2)
- T → 0：softmax 退化为 argmax，确定性选最大 logit
- T = 1：标准 softmax
- T → ∞：趋向均匀分布，每个 Token 被选中的概率趋于相等

### 1.3 与 Top-p（Nucleus Sampling）的关系

OpenAI 官方文档建议：**Temperature 和 Top-p 通常只调一个，不要同时修改。**[¹](#ref1)

| 参数 | 控制方式 | 默认值 |
|---|---|---|
| **Temperature** | 缩放整个概率分布的"陡峭程度" | 1.0 |
| **Top-p** | 只从累积概率前 p% 的 Token 中采样 | 1.0 |

两者都可以降低输出的随机性，但机制不同。实际工程中，常见做法是固定 Top-p=1，只调 Temperature。

### 1.4 实践建议

```python
# 精确任务（JSON 提取、分类）
response = client.chat.completions.create(
    model="qwen-plus",
    temperature=0,
    messages=[...]
)

# 创意任务（广告文案、诗歌）
response = client.chat.completions.create(
    model="qwen-plus",
    temperature=1.2,
    messages=[...]
)
```

### 1.5 常见任务的 Temperature 推荐值

| 任务类型 | 推荐 Temperature | 理由 |
|---|---|---|
| JSON/结构化数据提取 | 0 | 需要确定性输出，不允许随机变化 |
| 代码生成 | 0–0.3 | 代码需要语法正确，但可接受少量变化 |
| 事实问答（QA） | 0–0.3 | 准确性优先 |
| 通用对话 | 0.5–0.7 | 兼顾自然和准确 |
| 翻译 | 0.3–0.5 | 可接受表达差异，但语义必须准确 |
| 营销文案/创意写作 | 0.8–1.2 | 需要多样性和创造力 |
| 头脑风暴 | 1.0–1.5 | 鼓励发散思维 |

### 1.6 其他常用采样参数

除 Temperature 和 Top-p 外，API 还提供以下常用参数：

| 参数 | 作用 | 典型值 |
|---|---|---|
| **max_tokens** | 限制输出的最大 Token 数 | 1–4096+ |
| **frequency_penalty** | 降低已出现 Token 的再次出现概率（减少重复） | 0–2.0 |
| **presence_penalty** | 鼓励模型提及新话题（增加多样性） | 0–2.0 |
| **stop** | 指定停止序列，模型生成到该序列即停止 | 自定义字符串 |

## 二、Context Window（上下文窗口）

### 2.1 定义

**Context Window（上下文窗口）** 是模型一次 Inference 能处理的**最大 Token 数量上限**（包含输入 + 输出）。[³](#ref3) [⁴](#ref4)

> 类比：Context Window 就是模型的"临时记忆体容量"。超出窗口的信息会被截断或丢弃，模型完全"看不见"那些超出部分。

### 2.2 主流模型的窗口大小

| 模型 | Context Window | 约等于中文字数 |
|---|---|---|
| GPT-4o | 128K tokens | ~9 万字 |
| Claude 3.5 Sonnet | 200K tokens | ~14 万字 |
| Qwen-Plus | 128K tokens | ~9 万字 |
| Kimi | 200 万字（约 2.8M tokens） | ~200 万字 |
| Gemini 1.5 Pro | 1M tokens | ~70 万字 |

数据来源：各厂商官方文档，截至 2025 年。[³](#ref3)

### 2.3 Context 包含哪些内容

Context 不只是"用户的问题"，而是模型这次 Inference 所接收到的**所有信息之和**：[³](#ref3)

```
Context = System Prompt
        + 历史对话记录
        + RAG 检索到的知识片段
        + 工具描述（MCP/Function Calling）
        + 当前用户问题
        + 模型已生成的回复部分
```

### 2.4 窗口限制的实际影响

1. **多轮对话越聊越"健忘"**：当历史消息累积超过窗口，早期对话会被截断，模型表现得像"忘了之前说过什么"
2. **长文档处理受限**：一份 50 页的 PDF 可能超出某些模型的窗口
3. **成本线性增长**：输入 Token 越多，API 费用越高、延迟越大

### 2.5 "Lost in the Middle" 问题

研究发现，即使内容在窗口范围内，模型对 Context **中间部分**的信息召回率也显著低于开头和结尾。[³](#ref3) [⁵](#ref5)

> Anthropic 2025 年的研究建议：将最关键的信息放在 Context 的**开头或结尾**，而非中间。

### 2.6 应对窗口限制的工程策略

| 策略 | 说明 |
|---|---|
| **RAG** | 不把所有文档塞进 Context，只检索相关片段 |
| **对话摘要压缩** | Agent 定期对历史对话做摘要，压缩 Token 占用 |
| **滑动窗口** | 只保留最近 N 轮对话 |
| **选用大窗口模型** | 需要长文档分析时选 Kimi/Gemini 等超长上下文模型 |

### 2.7 实际场景中的 Token 预算分配

![上下文窗口的 Token 预算分配示意](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/context-window-budget-matched_1784704952.png)

以一个典型的 RAG 客服场景为例，使用 128K Context Window 的模型：

```
┌─────────────────────────────────────────────────────────┐
│ 128K Token 总预算分配                                     │
├─────────────────────────────────────────────────────────┤
│ System Prompt（角色定义 + 规则）        ≈  500 tokens    │
│ 工具描述（MCP/Function Calling）        ≈  2,000 tokens  │
│ RAG 检索结果（Top-5 片段）             ≈  5,000 tokens  │
│ 历史对话记录（最近 10 轮）             ≈  3,000 tokens  │
│ 当前用户问题                           ≈  100 tokens    │
│ ─────────────────────────────────────────               │
│ 输入总计                               ≈ 10,600 tokens  │
│ 留给模型输出                           ≈ 4,000 tokens   │
│ 剩余空间（安全余量）                    ≈ 116,472 tokens │
└─────────────────────────────────────────────────────────┘
```

> 虽然 128K 看起来很大，但实际业务中 RAG 片段多、历史对话长、工具定义复杂时，Token 消耗增长很快，务必做好预算规划。

### 2.8 两个概念的核心对比

| 维度 | Temperature | Context Window |
|---|---|---|
| 本质 | 采样策略参数 | 容量硬限制 |
| 影响什么 | 输出的随机性/多样性 | 模型能"看到"多少信息 |
| 可调范围 | 0–2.0（连续值） | 固定值（由模型架构决定） |
| 调高的代价 | 可能产生离谱输出 | 不可调高（超出即截断） |
| 调低的代价 | 输出单一/重复 | N/A |
| 是否影响成本 | 不影响 | 输入 Token 越多越贵 |
| 是否影响质量 | 间接影响（太高太低都不好） | 直接影响（关键信息被截断则质量骤降） |

## 常见误解

**关于 Temperature 参数**：
- ❌ "控制模型的响应速度"——Temperature 不影响速度
- ❌ "控制输出文本的长度"——长度由 max_tokens 参数控制
- ❌ "控制模型的知识范围"——Temperature 不改变模型知道什么，只改变它如何选择输出

**关于 Context Window**：
- ❌ "Context Window 越大，模型越聪明"——窗口大小是容量限制，不影响智力水平
- ❌ "Context Window 可以无限扩展"——受硬件算力和架构限制
- ❌ "只有用户输入占用 Context Window"——System Prompt、历史消息、模型输出都会占用

## 参考资料

<a id="ref1"></a>**[1]** OpenAI. *API Reference — Chat Completions*（temperature, top_p 参数说明）. https://platform.openai.com/docs/api-reference/chat/create

<a id="ref2"></a>**[2]** OpenAI Community. *Cheat Sheet: Mastering Temperature and Top_p in ChatGPT API*. https://community.openai.com/t/cheat-sheet-mastering-temperature-and-top-p-in-chatgpt-api/172683

<a id="ref3"></a>**[3]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》第 3-4 节 Context/Context Window. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref4"></a>**[4]** Learn Prompting. *Understanding Temperature, Top P, and Maximum Length in LLMs*. https://learnprompting.org/docs/intermediate/configuration_hyperparameters

<a id="ref5"></a>**[5]** Liu, N. et al. *Lost in the Middle: How Language Models Use Long Contexts*. arXiv, 2023. https://arxiv.org/abs/2307.03172

