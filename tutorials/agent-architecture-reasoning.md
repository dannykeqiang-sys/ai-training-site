# Agent 架构与推理模式

AI Agent 通过感知环境、自主规划、调用工具和观察反思，将大语言模型从"一问一答"升级为能自主完成复杂任务的智能系统。ReAct、Plan-and-Execute 和自我反思是当前最主流的三种 Agent 推理与行动架构模式。

## 一、什么是 AI Agent

**AI Agent（智能体）** 是一种能够自主决策并执行任务的智能系统。它以 LLM 为核心"大脑"，通过感知环境、制定计划、调用工具、观察结果，形成持续运行的循环，直到完成目标。[1](#ref1) [2](#ref2)

### Agent 与标准 LLM 的核心区别

| 维度 | 标准 LLM | AI Agent |
|---|---|---|
| **交互模式** | 单轮问答（stateless） | 多轮循环、持续执行 |
| **自主性** | 被动响应 Prompt | 主动规划、自主决策 |
| **工具使用** | 无（纯文本输出） | 可调用外部 API、数据库 |
| **记忆** | 仅当次 Context Window | 短期 + 长期记忆 |
| **错误恢复** | 无 | 观察结果 → 反思 → 调整 |

> 一句话：**标准 LLM 像一本百科全书——你翻到哪页就看哪页；Agent 像一个员工——你下达目标，它自己拆解任务、找工具、干活、汇报。**

## 二、Agent 核心架构

![Agent 核心架构：LLM 大脑 + 规划 + 记忆 + 工具](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/agent-core-arch_1784688967.png)

一个完整的 Agent 系统由三大模块组成：[1](#ref1) [2](#ref2)

```
┌─────────────────────────────────────────────┐
│                  AI Agent                    │
│   ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│   │ Planning │  │  Memory  │  │  Tools  │  │
│   │ (规划)   │  │  (记忆)  │  │ (工具)  │  │
│   └────┬─────┘  └────┬─────┘  └────┬────┘  │
│        └──────────────┼──────────────┘       │
│              ┌────────▼────────┐             │
│              │  LLM（核心大脑）│             │
│              └────────┬────────┘             │
│              ┌────────▼────────┐             │
│              │ Observation &   │             │
│              │ Reflection      │             │
│              └─────────────────┘             │
└─────────────────────────────────────────────┘
```

### 规划（Planning）

将复杂目标拆解为可执行子任务。常见方式包括任务分解、Chain-of-Thought、ReAct、Plan-and-Execute。

### 记忆（Memory）

| 类型 | 说明 | 实现方式 |
|---|---|---|
| **短期记忆** | 当前任务/对话中的上下文信息 | Context Window 中的内容 |
| **长期记忆** | 跨会话的持久化信息 | 向量数据库或传统数据库 |

> 短期记忆 = 当前对话会话中的上下文信息（Context），随会话结束而消失。

### 工具调用（Tools）

Agent 通过 Function Calling 调用外部工具。关键点：**LLM 本身不能直接执行工具**，它只能输出"工具调用指令"，由外层系统（Harness）实际执行并将结果返回。

### 任务分解（Task Decomposition）

将复杂任务拆分为更小、更简单的子任务：降低每步出错概率、支持并行化、便于错误定位。

## 三、ReAct：推理与行动的交织

![ReAct 循环：思考 → 行动 → 观察](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/react-loop_1784688974.png)

### 3.1 什么是 ReAct

**ReAct**（Reasoning + Acting）让 LLM 在完成任务时交替进行**推理**和**行动**，每一步产出三个交织的组件：[3](#ref3)

| 组件 | 说明 |
|---|---|
| **Thought（思考）** | 推理当前状态、决定下一步策略 |
| **Action（行动）** | 执行具体操作（如调用搜索工具） |
| **Observation（观察）** | 从环境获得的反馈（如搜索结果） |

### 3.2 ReAct 执行示例

```
用户：阿里巴巴的创始人是谁？他的出生年份是？

Thought 1: 我需要先查询阿里巴巴的创始人
Action 1:  Search["阿里巴巴 创始人"]
Observation 1: 阿里巴巴由马云于 1999 年创立

Thought 2: 接下来查他的出生年份
Action 2:  Search["马云 出生年份"]
Observation 2: 马云，1964 年 9 月 10 日出生

Thought 3: 已收集到所有信息
Action 3:  Finish["阿里巴巴创始人是马云，出生于 1964 年"]
```

### 3.3 ReAct vs 纯 CoT

| 维度 | 纯 CoT | ReAct |
|---|---|---|
| **本质** | 仅推理，不与外界交互 | 推理 + 行动 + 观察三者交替 |
| **信息来源** | 仅依赖模型内部知识 | 可通过 Action 获取外部信息 |
| **幻觉风险** | 高——不确定时会编造 | 低——通过外部验证纠正 |
| **适用场景** | 数学推理、逻辑题 | 信息检索、多步任务 |

> **一句话**：ReAct = CoT + 行动能力。CoT 只会"想"，ReAct 既会"想"也会"做"。

## 四、Plan-and-Execute 架构

### 4.1 基本思想

将**规划**与**执行**分离：[4](#ref4) [5](#ref5)

- **Planner（规划器）**：一次性生成完整的任务计划
- **Executor（执行器）**：逐步执行计划中的每个步骤

```
┌──────────────┐         ┌──────────────┐
│   Planner    │────────▶│   Executor   │
│  (生成计划)  │         │  (逐步执行)  │
└──────────────┘         └──────┬───────┘
       ▲                        │
       │     反馈/重新规划       │
       └────────────────────────┘
```

### 4.2 与 ReAct 的对比

| 维度 | ReAct | Plan-and-Execute |
|---|---|---|
| **规划粒度** | 每步即时决策（短视） | 先全局规划再逐步执行 |
| **调整灵活性** | 高——每步可即时调整 | 中——需要"重新规划" |
| **适合任务** | 信息检索、探索性任务 | 目标明确、步骤可预见的多步任务 |

### 4.3 Planner 与 Executor 分离的优势

1. **全局视角**：避免 ReAct 的"贪心"决策
2. **职责单一**：Planner 专注"做什么"，Executor 专注"怎么做"
3. **成本优化**：Planner 用强模型（如 GPT-4），Executor 用轻量模型
4. **可审计性**：完整计划在执行前可供人类审核

## 五、自我反思（Self-Reflection）

Agent 在执行任务后，**对自身行为和结果进行回顾和评估**，从中总结经验教训：[6](#ref6)

```
执行任务 → 观察结果 → 反思（成功/失败原因） → 生成改进策略 → 下次执行时应用
```

**Reflexion 框架**的核心设计是"语言强化学习"——不更新模型权重，而是将经验以自然语言的形式存储，在后续尝试的 Prompt 中注入这些"经验教训"。

**反思结论注入 Prompt 示例**：

```
【上次尝试的反思】
- 第 1 次尝试：使用了冒泡排序（O(n²)），不满足性能要求
- 失败原因：未优先考虑时间复杂度约束
- 改进策略：遇到排序需求时，优先选择 O(n log n) 算法

当前任务：请实现一个对大规模数组排序的函数，要求时间复杂度不超过 O(n log n)。
```

## 六、Workflow vs Agent：何时选择 Agent

**关键原则：能用 Workflow 解决的问题不要用 Agent。**[3](#ref3)

| 维度 | Workflow（工作流） | Agent（智能体） |
|---|---|---|
| **控制方式** | 预定义的固定流程 | LLM 动态决策 |
| **执行路径** | 确定性 | 非确定性 |
| **可预测性** | 高——容易测试 | 低——需要更多 护栏 |

**何时必须用 Agent**：
- 任务步骤**事先无法完全确定**
- 需要**多次工具交互**
- 任务涉及**开放式探索**
- 需要**错误恢复和重试策略**

## 参考资料

<a id="ref1"></a>**[1]** 刘立成(文辺). 《关于 AI 的一些新名词和概念》第四章 Agent. ATA, 2025-04-18. https://ata.atatech.org/articles/11020405255 *（内网）*

<a id="ref2"></a>**[2]** 蹇志坤(义坤). 《AI 核心概念大串联》第 10 节 Agent. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref3"></a>**[3]** Anthropic. *Building Effective Agents*. 2024-12. https://www.anthropic.com/research/building-effective-agents

<a id="ref4"></a>**[4]** LangChain. *Plan-and-Execute Agents*. https://www.langchain.com/blog/planning-agents

<a id="ref5"></a>**[5]** Yao, S., et al. *ReAct: Synergizing Reasoning and Acting in Language Models*. ICLR 2023. https://arxiv.org/abs/2210.03629

<a id="ref6"></a>**[6]** Shinn, N., et al. *Reflexion: Language Agents with Verbal Reinforcement Learning*. NeurIPS 2023. https://arxiv.org/abs/2303.11366
