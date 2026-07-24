# Fine-tuning、PEFT 与 LoRA

Fine-tuning（微调）是让预训练模型适应特定任务的核心手段，但全量微调的高昂成本让很多团队望而却步。PEFT（参数高效微调）和 LoRA 的出现改变了这一局面——用极少的计算资源就能达到接近全量微调的效果。本文从全量微调的挑战出发，讲解 SFT 的训练流程、PEFT 的思想和 LoRA 的核心原理，帮助你做出合理的技术选型。

## 一、Fine-tuning（微调）的定义与核心目标

### 1.1 什么是 Fine-tuning

**Fine-tuning（微调）** 是指在一个**已经预训练好的大语言模型**基础上，使用特定领域或任务的数据集**继续训练**，从而让模型学会新的行为模式、风格或专业能力的过程。[1](#ref1) [2](#ref2)

```
预训练模型（通用能力） + 领域/任务数据 → 继续训练 → 微调后模型（专用能力）
```

核心特征：**Fine-tuning 会更新模型的权重参数**。这区别于 Prompting（不改权重，只改输入）和 RAG（不改权重，只加外部知识）。

### 1.2 Fine-tuning 的核心目标

Fine-tuning 的目标不是让模型"记住新知识"（这是 RAG 的强项），而是让模型：[1](#ref1) [2](#ref2)

| 目标 | 说明 | 例子 |
|---|---|---|
| **学会新的行为模式** | 让模型以特定方式响应 | 始终输出结构化 JSON |
| **掌握特定风格** | 让模型用指定的语气/格式写作 | 模仿某公司的客服话术 |
| **提升特定任务精度** | 在窄领域获得超越通用模型的表现 | 法律文书分类、医学实体提取 |
| **减少推理成本** | 把 Few-shot 中的示例"内化"到权重里 | 省去每次调用都要带 10 条示例的 Token 消耗 |

> 一句话：**Fine-tuning 是通过继续训练来更新模型权重，使模型在特定任务或领域上获得更好的表现。**

## 二、SFT（有监督微调）：Fine-tuning 的主流形式

### 2.1 什么是 SFT

**SFT（Supervised Fine-Tuning，有监督微调）** 是使用**人工标注的指令-回答对**对预训练模型进行微调的过程。它是当前大语言模型从"能续写文本"进化到"能遵循指令对话"的关键步骤。[1](#ref1) [7](#ref7)

```
预训练模型（只会续写文本） + 指令-回答数据 → SFT 训练 → 能遵循指令的对话模型
```

> 一句话：**SFT 是 Fine-tuning 在 LLM 领域最主流的应用形式。** 当我们说"微调一个 LLM"时，绝大多数情况指的就是 SFT。

### 2.2 SFT 在 LLM 训练流程中的位置

现代大语言模型的开发通常经历三个阶段，SFT 是承上启下的关键环节：[7](#ref7) [8](#ref8)

```
┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
│ Pre-training │ ──→ │     SFT     │ ──→ │ RLHF / DPO       │
│  预训练      │     │  有监督微调  │     │  人类偏好对齐     │
│             │     │             │     │                  │
│ 学会语言能力 │     │ 学会遵循指令 │     │ 学会人类价值观    │
│ 获得世界知识 │     │ 学会对话格式 │     │ 拒绝有害请求      │
└─────────────┘     └─────────────┘     └──────────────────┘
```

| 阶段 | 训练数据 | 目标 | 产出 |
|------|---------|------|------|
| **Pre-training** | 海量无标注文本（万亿 Token） | 学习语言和世界知识 | Base 模型（如 LLaMA-Base） |
| **SFT** | 人工标注的指令-回答对（万~百万条） | 学会遵循指令、对话交互 | Instruct/Chat 模型（如 ChatGPT） |
| **RLHF/DPO** | 人类偏好排序数据 | 对齐人类价值观，拒绝有害请求 | 最终发布模型 |

> **关键理解**：你在日常使用的 ChatGPT、Claude、Qwen-Chat 等对话模型，都是经过 SFT 训练的。没有 SFT，模型只是一个"文本续写器"，无法进行有结构的对话。

### 2.3 SFT 的数据格式

SFT 的核心是**高质量的指令-回答数据**。典型的数据格式如下：

**单轮对话**：
```json
{
  "instruction": "将以下文本翻译为英文",
  "input": "今天天气真好",
  "output": "The weather is really nice today"
}
```

**多轮对话**：
```json
{
  "conversations": [
    {"role": "system", "content": "你是一个专业的Python开发助手"},
    {"role": "user", "content": "如何实现单例模式？"},
    {"role": "assistant", "content": "Python中实现单例模式有多种方式..."}
  ]
}
```

### 2.4 SFT 数据质量的关键原则

SFT 的效果高度依赖数据质量，而非数据数量。核心原则：[1](#ref1) [7](#ref7)

| 原则 | 说明 | 反面案例 |
|------|------|---------|
| **高质量 > 大数量** | 1000 条精心标注的数据可能胜过 10 万条噪声数据 | 用 GPT 批量生成未审核的数据做 SFT |
| **分布一致** | 训练数据的分布应接近实际使用场景 | 训练数据全是简单问题，线上遇到复杂问题 |
| **多样性** | 覆盖各种任务类型、难度级别和边界情况 | 只有翻译任务的数据，缺少推理、创作等 |
| **人工审核** | 每条数据都应有专业人员审核确认 | 完全依赖自动化标注，无人工校验 |

> **经验法则**：LIMA 论文（"Less Is More for Alignment"）证明，仅用 1000 条精选数据做 SFT，就能训练出质量接近 GPT-4 的模型。这说明数据质量远比数量重要。[8](#ref8)

### 2.5 SFT 的实施方式：Full Fine-tuning vs PEFT

SFT 描述的是"用什么数据训练"（指令-回答对），而 Full Fine-tuning 和 PEFT 描述的是"怎么训练"（更新多少参数）。两者是正交的概念：

| 组合方式 | 说明 |
|---------|------|
| **SFT + Full Fine-tuning** | 用指令数据做全量参数更新，资源充足时追求最优效果 |
| **SFT + LoRA/QLoRA** | 用指令数据做低秩适配，资源有限时最推荐的组合 |
| **SFT + Adapter** | 用指令数据训练插入的瓶颈网络，需要多任务切换 |

> **实践推荐**：对于大多数团队，**SFT + LoRA/QLoRA** 是最佳起步方案——用少量高质量指令数据做 LoRA 微调，在单张消费级 GPU 上就能完成。

## 三、Fine-tuning vs Prompting：什么时候用哪个

### 3.1 决策框架

![Fine-tuning vs Prompting 决策路径](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/finetune-vs-prompt-decision_1784688694.png)

OpenAI 官方给出的判断准则：[1](#ref1) [2](#ref2)

| 场景特征 | 推荐方案 | 理由 |
|---|---|---|
| 任务可以用**自然语言清楚描述** | Prompting | 好的 Prompt 就够了，无需训练 |
| 需要**大量示例**才能让模型理解格式/风格 | Fine-tuning | 把示例内化到权重，省去每次调用的 Token |
| 需要模型**严格遵循**特定输出格式 | Fine-tuning | 训练后模型更稳定地输出期望格式 |
| 需要**降低延迟**（减少 Prompt 长度） | Fine-tuning | 短 Prompt 即可触发正确行为 |
| 知识**频繁变动** | Prompting + RAG | 不需要反复重训 |
| 有**高质量标注数据**且任务固定 | Fine-tuning | 数据质量保证训练效果 |

### 3.2 典型场景对照

| 场景 | 选 Prompting | 选 Fine-tuning |
|---|---|---|
| "用某律所的固定格式写合同" | — | 风格/格式内化到权重 |
| "把用户反馈分类为 Bug/Feature/Question" | Few-shot 可以但 Token 浪费 | 训练后零 Prompt 即可分类 |
| "模型总是输出 Markdown 但我需要纯 JSON" | Prompt 约束不稳定 | 训练强化 JSON 输出 |

**当需要模型学会某种稳定的行为模式/输出格式，且有高质量训练数据时，选 Fine-tuning；当任务可以通过指令描述清楚且知识可能变化时，选 Prompting。**

## 四、全量微调的挑战与 PEFT 的直觉

传统的 **Full Fine-tuning（全量微调）** 是更新模型的**全部参数**。对于现代大模型，这面临巨大挑战：[3](#ref3)

| 挑战 | 说明 |
|---|---|
| **GPU 显存需求巨大** | 7B 模型全量微调需要 ~56GB 显存（参数 + 梯度 + 优化器状态） |
| **训练成本高** | 大模型全量训练需要多卡并行，费用从数千到数十万美元不等 |
| **存储成本高** | 每个任务产生一个完整的模型副本（7B 模型约 14GB） |
| **灾难性遗忘** | 过度训练可能导致模型丧失原有的通用能力 |

既然全量微调太贵，自然的想法是：能不能只动一小部分参数就达到差不多的效果？**PEFT（参数高效微调）**的核心洞察是：微调时模型真正需要改变的"信息量"远小于模型的总参数量。就像你不需要重写整本教科书来修正一个章节——只需要在关键位置做标注就够了。

## 五、LoRA：最主流的 PEFT 方法

**LoRA（Low-Rank Adaptation）** 是目前最广泛使用的 PEFT 方法，由微软于 2021 年提出。它的核心思想是：微调时权重的变化量 ΔW 是**低秩**的，不需要更新完整的权重矩阵。[5](#ref5)

### 5.1 核心机制

![全量微调与 LoRA 的对比](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/lora-mechanism_1784688682.png)

对于预训练权重矩阵 W₀（维度 d × k），LoRA **冻结** W₀ 不动，额外注入两个小矩阵的乘积来近似权重更新：

```
原始前向传播:  h = W₀ · x
LoRA 前向传播:  h = W₀ · x + (B · A) · x
                        ↑
                   ΔW = B · A（低秩近似）

其中：
  A: r × k 矩阵（降维）
  B: d × r 矩阵（升维）
  r << min(d, k)（r 远小于 d 和 k）
```

**参数量对比**（以 LLaMA-7B 的 attention 层为例，d = k = 4096）：

| 方法 | 参数量 | 说明 |
|------|--------|------|
| Full Fine-tuning | d × k = 16,777,216 | 更新完整权重矩阵 |
| LoRA (r=8) | d×r + r×k = 65,536 | 仅为全量的 **0.39%** |
| LoRA (r=16) | d×r + r×k = 131,072 | 仅为全量的 **0.78%** |

> 一句话：**LoRA 用两个小矩阵的乘积来近似完整的权重更新，训练参数量降低 1-2 个数量级，效果接近全量微调。**

### 5.2 为什么低秩就够了

LoRA 论文的关键发现：微调时权重变化矩阵 ΔW 虽然维度很大，但其**内在秩（intrinsic rank）很低**——大部分信息集中在少数几个方向上。[5](#ref5) 这意味着用低秩矩阵就能很好地近似 ΔW，就像一张照片虽然有几百万像素，但压缩后仍然能保留主要信息。

### 5.3 关键超参数

| 超参数 | 含义 | 推荐值 | 说明 |
|--------|------|--------|------|
| **r (rank)** | 低秩矩阵的秩 | 8 ~ 64 | 越大表达能力越强，但参数更多。简单任务 r=8 通常足够 |
| **α (alpha)** | 缩放系数 | 通常为 r 的 2 倍 | 控制 LoRA 更新的幅度，实际缩放为 α/r |
| **target_modules** | 对哪些层注入 LoRA | q_proj, v_proj | 最少注入 attention 的 Q、V 投影层；追求效果可加全部线性层 |
| **dropout** | LoRA 层的 dropout 率 | 0.05 ~ 0.1 | 防止过拟合 |

### 5.4 推理时的合并

LoRA 的一个重要优势：**训练完成后，可以将 LoRA 权重合并回原始模型，推理时零额外延迟**。

```
训练时:  h = W₀ · x + (B · A) · x    （两路计算）
推理时:  h = (W₀ + B · A) · x         （合并为一个矩阵，一次计算）
```

这也意味着可以为不同任务训练不同的 LoRA 适配器，推理时按需加载切换——比存储多个完整模型便宜得多（每个 LoRA 适配器仅几 MB ~ 几十 MB）。

### 5.5 QLoRA：进一步降低显存

**QLoRA** 在 LoRA 基础上引入了 4-bit 量化：将冻结的预训练权重 W₀ 量化为 4-bit 存储，只对 LoRA 的 A、B 矩阵做全精度训练。[5](#ref5)

| 方法 | 7B 模型训练显存 | 效果 |
|------|----------------|------|
| Full Fine-tuning | ~56 GB | 基准 |
| LoRA (r=16, fp16) | ~16 GB | 接近全量微调 |
| QLoRA (r=16, 4-bit) | ~6 GB | 接近 LoRA，可在单张消费级 GPU 上训练 |

> **实践推荐**：对于资源有限的团队，**SFT + QLoRA** 是最佳起步方案——在单张 24GB 显存的消费级 GPU（如 RTX 4090）上即可完成 7B 模型的微调。

## 六、技术路线选择总览

```
┌─────────────────────────────────────┐
│  技术路线选择                        │
│                                     │
│  需要新知识？ → RAG（外挂知识库）    │
│  需要新行为/风格？                  │
│    有 GPU 资源 → Full Fine-tuning   │
│    资源有限   → PEFT (LoRA/QLoRA)   │
│  任务简单可描述？ → Prompting       │
└─────────────────────────────────────┘
```

### 组合使用：RAG + Fine-tuning

RAG 和 Fine-tuning 并不是互斥的选择，在实际生产中经常组合使用。典型的做法是：先用 Fine-tuning（如 LoRA）让模型学会特定领域的表达风格、输出格式和专业术语，再用 RAG 动态注入最新的事实性知识。这样既能利用 Fine-tuning 在行为模式上的优势（稳定的格式输出、领域化的表达），又能利用 RAG 在知识更新上的灵活性（无需重新训练即可更新知识库）。对于企业级 AI 应用，"LoRA 微调 + RAG"是目前最推荐的组合方案——用少量高质量数据做 LoRA 微调来规范模型行为，用 RAG 来补充频繁变动的知识。

## 常见误解

**关于 Fine-tuning**：
- ❌ "Fine-tuning 是从零训练模型"——那是 Pre-training，Fine-tuning 是在预训练模型基础上继续训练
- ❌ "Fine-tuning 总是比 Prompting 好"——取决于场景：稳定格式/风格用 FT，知识常变用 Prompting

**关于 SFT**：
- ❌ "SFT 就是让模型记住新知识"——那是 RAG 的强项，SFT 是教模型新的**行为模式**
- ❌ "SFT 数据越多越好"——LIMA 论文证明 1000 条精选数据就足够

**关于 PEFT**：
- ❌ "PEFT 不更新任何权重"——它更新的是注入的适配器参数，只是冻结大部分原始权重

## 参考资料

<a id="ref1"></a>**[1]** OpenAI. *Fine-tuning best practices*. https://developers.openai.com/api/docs/guides/fine-tuning-best-practices

<a id="ref2"></a>**[2]** OpenAI. *Model optimization*. https://developers.openai.com/api/docs/guides/model-optimization

<a id="ref3"></a>**[3]** Hugging Face. *PEFT: State-of-the-art Parameter-Efficient Fine-Tuning*. https://github.com/huggingface/peft

<a id="ref5"></a>**[5]** Hu, E. J. et al. *LoRA: Low-Rank Adaptation of Large Language Models*. arXiv:2106.09685, 2021. https://arxiv.org/abs/2106.09685

<a id="ref7"></a>**[7]** Ouyang, L. et al. *Training language models to follow instructions with human feedback*. NeurIPS, 2022. https://arxiv.org/abs/2203.02155

<a id="ref8"></a>**[8]** Zhou, C. et al. *LIMA: Less Is More for Alignment*. NeurIPS, 2023. https://arxiv.org/abs/2305.11206

