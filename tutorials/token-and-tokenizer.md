# Token 与 Tokenizer

Token 是理解大语言模型的起点——它决定了模型能"看到"什么、花多少钱、能处理多长的文本。Tokenizer 则是完成"文字到数字"转换的核心组件，不同 Tokenizer 的设计直接影响模型的理解能力和效率。

计算机只能处理数字，所以任何文字进入大模型之前，都要先被翻译成数字。

## 一、一句话定义

- **Token（词元）** 是大语言模型处理文本的最小单元。模型不直接理解文字，所有输入和输出都要先经过 Tokenizer 在「文字 ↔ 整数 ID」之间互译，模型实际上只在和这串整数打交道。
- **Tokenizer（分词器）** 是执行这种互译的组件，由**切分算法**和**词表**两部分组成。同一个文本经过不同 Tokenizer 会产生不同的 Token 序列，因此 Tokenizer 的选择直接决定了模型的输入质量。

> Token 的中文译名「**词元**」已在行业内逐步推广使用。[¹](#ref1)

## 二、Tokenizer 的工作原理

![Tokenizer 分词流程：文本 → Token → Token ID](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/token-tokenizer-flow-matched-v2_1784705361.png)

Tokenizer 把一段文本变成数字，分两步：

1. **切分（Segmentation）**：把句子拆成若干最小片段。
   - 例：「你好吗」可能被切成 `你` / `好吗` 两个 Token。
   - 例：英文 `"unbelievable"` 在 BPE 算法下可能被切成 `un` / `believ` / `able` 三个 subword。
2. **映射（Mapping）**：每个 Token 对应词表里一个唯一整数 **Token ID**。模型读到的是这串 ID 列表。

### 2.1 主流分词算法

| 算法 | 原理 | 代表模型 | 特点 |
|------|------|---------|------|
| **BPE**（Byte-Pair Encoding） | 从单字节出发，反复合并最高频相邻对，直到达到词表大小 | GPT 系列、LLaMA、Qwen | 不会出现未登录词（最坏退化到字节级） |
| **WordPiece** | 类似 BPE，但以最大化语言模型似然度（而非频率）为合并标准 | BERT、DistilBERT | 适合理解型任务，未登录词用 `##` 前缀标记 |
| **SentencePiece** | 直接在原始字节流上训练，不依赖空格做预分词 | T5、mBART、部分多语言模型 | 天然适合中文、日文等无空格语言 |
| **Unigram** | 从大词表出发，逐步剪枝低概率子词，保留最精简的词表 | AlBERT、部分 T5 变体 | 提供多种切分方案，选概率最高的 |


### 2.2 词表（Vocabulary）

词表是 Tokenizer 的核心资产，记录了所有已知 Token 与其 ID 的映射关系。词表大小直接影响模型的表现：

| 词表大小 | 优势 | 劣势 |
|---------|------|------|
| 小（~3 万） | 模型参数少、推理快 | 常见词被频繁切碎，表达效率低 |
| 中（~5-10 万） | 平衡点，多数主流模型的选择 | — |
| 大（~15 万+） | 高频短语直接对应单个 Token，表达紧凑 | Embedding 矩阵变大，内存和计算开销增加 |

**特殊 Token**：词表中除了自然语言的子词，还包含一批有特殊功能的 Token：

| 特殊 Token | 作用 |
|-----------|------|
| `[BOS]` / `<s>` | 序列起始标记 |
| `[EOS]` / `</s>` | 序列结束标记，模型看到它就知道输出该停了 |
| `[PAD]` | 填充标记，把不等长的序列补齐到相同长度以便批量处理 |
| `[UNK]` | 未登录词标记，BPE/SentencePiece 通常不需要（可退化到字节级） |

### 2.3 编码与解码

Tokenizer 的工作是可逆的：

```
编码（Encode）：文本 → Token 序列 → ID 列表
   "你好世界" → ["你", "好", "世界"] → [1234, 5678, 9012]

解码（Decode）：ID 列表 → Token 序列 → 文本
   [1234, 5678, 9012] → ["你", "好", "世界"] → "你好世界"
```

模型的推理过程就是：接收编码后的 ID 列表 → 逐 Token 预测下一个 ID → 解码回文本。理解了这一点，就能明白为什么模型的"上下文窗口"是以 Token 数来衡量的——它实际上是在限制一次能处理的 ID 序列长度。

## 三、不同模型，Tokenizer 和 Token 数都不一样

同一段文本，在不同模型里被切出的 Token 数可能差别很大，因为每家用的 Tokenizer 和词表不同：

| 模型 | Tokenizer | 词表大小 | 中文友好度 |
|---|---|---|---|
| GPT 系列 | tiktoken（BPE 变体，cl100k / o200k） | ~10 万 / ~20 万 | 一般，中文常被切成 1.5 字/Token |
| Claude 系列 | Anthropic 自研 Tokenizer | 未公开 | 一般 |
| 通义 Qwen 系列 | QwenTokenizer（BPE 变体） | ~15 万 | 较好，中文密度高 |
| BERT 系列 | WordPiece | ~3 万 | 一般，未登录词用 `##` 标记 |
| T5 系列 | SentencePiece | ~3.2 万 | 较好，直接在字节流上训练 |

**经验估算**（来自 ATA 文章）[¹](#ref1)：

**英文**：1 Token ≈ 0.75 个英文单词

**中文**：1 Token ≈ 1.5 个汉字

**示例**：60 个汉字 ≈ 40 Tokens

可以用 OpenAI 官方的 [Tokenizer 可视化工具](https://platform.openai.com/tokenizer) 实际查看任意文本的切分结果。[⁴](#ref4)

> **Tokenizer 选择的实际影响**：同样一段 100 字的中文文本，用 GPT-4 的 tiktoken 可能切成 ~67 个 Token，而用 Qwen 的 Tokenizer 可能只需要 ~40 个 Token。这意味着使用中文优化的模型不仅 Token 消耗更低、费用更省，而且在同样的上下文窗口中能放入更多内容。

## 四、为什么 Token 这件事很重要

![相同含义文本的 Token 消耗对比](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/token-efficiency-compare_1784688569.png)

Token 数直接决定三件事：

1. **成本**：API 计费基本都按 Token 计（输入/输出分别计价）。
2. **速度**：每个 Token 都要做一次前向计算，输出延迟基本和输出 Token 数线性相关。
3. **上下文窗口（Context Window）**：模型一次最多能处理的 Token 数有上限（如 Qwen-Plus 128K、Kimi 200 万字），超出会被截断或丢弃。[¹](#ref1)

> **费用体感示例**：假设一次普通对话约 2000 input tokens + 500 output tokens，以 GPT-4o-mini 的价格（输入 $0.15/M tokens，输出 $0.60/M tokens）计算，单次对话约 $0.0006（不到半分钱）。但如果使用 GPT-4o（输入 $2.50/M，输出 $10.00/M），同样一次对话约 $0.01，100 次对话就是 1 美元。Token 消耗看似微小，规模化后差异巨大。

> 💡 容易忽略的事实：多模态模型中，**图片也会消耗 Token**。例如通义 qwen-vl 模型，每 28×28 像素 ≈ 1 个 Token，一张 1024×1024 的图就要消耗约 1300 Tokens。[⁵](#ref5) 这意味着如果你在上下文中放了 5 张图，光图片就占了约 6500 Tokens 的窗口空间，费用也会相应增加。

## 五、Tokenizer 的常见陷阱与边界情况

即使理解了 Tokenizer 的基本原理，实际使用中仍有一些容易踩坑的地方：

### 5.1 数字和代码的 Token 化效率

**问题**：数字和代码在不同 Tokenizer 下的效率差异很大。

```
示例：数字 "1234567890"

GPT-4 (tiktoken):
  → 可能切成 ["12", "34", "56", "78", "90"] = 5 Tokens
  
Qwen:
  → 可能切成 ["1234", "5678", "90"] = 3 Tokens

某些旧模型:
  → 可能切成 ["1", "2", "3", "4", "5", "6", "7", "8", "9", "0"] = 10 Tokens
```

**影响**：如果你的应用大量处理数字（如财务报表、数据分析），Tokenizer 的效率直接影响成本和速度。

**实践建议**：如果应用涉及大量代码或数字处理，优先选择对这些内容优化过的 Tokenizer（如 GPT-4 的 tiktoken 对代码有专门优化）。

### 5.2 特殊字符和 Emoji

**问题**：特殊字符和 Emoji 的 Token 化方式不一致，可能消耗更多 Token。

**实践建议**：在处理包含大量 Emoji 的文本时，先用 Tokenizer 工具预估 Token 数，避免超出上下文窗口。

### 5.3 多语言混合文本

**问题**：中英文混合文本的 Token 化效率通常低于纯中文或纯英文。Tokenizer 在语言边界处往往需要额外的 Token 来标记语言切换。

**实践建议**：如果可能，尽量保持文本语言一致。混合语言场景下，Token 预算要留更多余量。

### 5.4 Tokenizer 的"幻觉"切分

**问题**：某些专业术语可能被意外切碎（如 "Transformer" 被切成 ["Trans", "form", "er"]），导致语义碎片化。

**实践建议**：对于领域专业术语，可以在 System Prompt 中明确定义，帮助模型理解。

## 六、Token 使用的监控与优化

在生产系统中，监控和优化 Token 使用是控制成本的关键：

### 6.1 监控指标

| 指标 | 目标值 | 说明 |
|---|---|---|
| **平均单次调用 Token 数** | < 5000 | 输入 + 输出总和 |
| **输入/输出比例** | 2:1 ~ 5:1 | 输入通常多于输出 |
| **重试率** | < 5% | 因 Token 超限导致的重试 |
| **上下文利用率** | > 70% | 实际使用的 Token / 上下文窗口 |

### 6.2 优化策略

| 策略 | 方法 | 效果 |
|---|---|---|
| **Prompt 压缩** | 用简洁的关键词替代冗长描述（如"代码审查专家。检查：bug、性能、安全、风格"） | Token 减少 50-70% |
| **输出长度控制** | API 调用中设置 `max_tokens` 限制输出长度（需权衡截断风险） | 避免输出失控 |
| **上下文裁剪** | 多轮对话只保留最近 N 轮（如 `messages[-20:]`） | 防止无限累积 |
| **缓存复用** | 利用 Prompt Caching（如 Anthropic 的 `cache_control`）复用重复的 System Prompt | 节省重复 Token |

**常见"Token 爆炸"场景**：用户发送超长文本、多轮对话无限累积、工具返回结果未裁剪、重试逻辑无限制。

**效果**：如果 System Prompt 有 1000 Tokens，每天调用 1000 次，缓存可以节省 100 万 Tokens/天。

### 6.3 Token 计数工具

| 阶段 | 工具 |
|---|---|
| **开发阶段** | OpenAI Tokenizer（在线可视化）、`tiktoken` Python 库、Anthropic Token Counter |
| **生产阶段** | 使用 `tiktoken` 在发送前预估 Token 数，超限触发裁剪逻辑 |

### 6.4 成本预算与告警

在生产系统中设置 Token 使用告警（如达到日预算 80% 时警告，超限时降级到更便宜的模型），避免意外的高额账单。

## 常见误解

- ❌ "Token 就是单词"——英文里一个单词常被切成多个 subword
- ❌ "Token 就是一个汉字"——中文常常 2-3 字合并为 1 个 Token
- ❌ "所有模型用同一个 Tokenizer"——每家模型的 Tokenizer 和词表都不同
- ❌ "Tokenizer 只负责切分文本"——它还负责将 Token 映射为 ID（编码）和将 ID 还原为文本（解码）

## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** Hugging Face. *Byte-Pair Encoding tokenization*. LLM Course Chapter 6.5. https://huggingface.co/learn/llm-course/chapter6/5

<a id="ref3"></a>**[3]** Wikipedia. *Byte-pair encoding*. https://en.wikipedia.org/wiki/Byte-pair_encoding

<a id="ref4"></a>**[4]** OpenAI. *Tokenizer*（在线可视化工具）. https://platform.openai.com/tokenizer

<a id="ref5"></a>**[5]** 钱红凯(趋势). 《AI实践之路：LLM、RAG、MCP、Agent、A2A，掌握AI相关基础原理》. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*
