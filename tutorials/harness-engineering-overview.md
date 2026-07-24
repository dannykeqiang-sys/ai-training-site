# Harness Engineering 概览

在构建 AI Agent 产品时，一个常见的误区是将精力全部投入模型选型，却忽视了围绕模型的工程基础设施。Harness Engineering 正是关注这部分"冰山之下"的工程学科——它定义了 Agent 系统中除模型以外的一切，是决定 Agent 能否在生产环境中稳定可靠工作的关键。

## 一、一句话定义

**Harness（线束/支架）** 是 Agent 系统中**除模型本身以外的所有基础设施、约束和运行逻辑的总和**。它包括系统提示词、工具定义、执行沙盒、上下文管理、反馈回路、编排逻辑等一切让模型"能用起来"的工程部分。[¹](#ref1) [²](#ref2)

> LangChain 的定义原文："A harness is every piece of code, configuration, and execution logic that isn't the model itself."[²](#ref2)

**理解 Harness 的隐喻**：Harness 原意为马具或线束，是将动力（马/引擎）转化为有用功的中间装置。没有马具，再强壮的马也无法拉车；没有线束，引擎的电力无法驱动车轮。在 Agent 中，Harness 就是将模型的智能转化为实际可用功能的工程基础设施——模型再聪明，没有 Harness 也只是一个"有想法但没有手脚"的大脑。

## 二、核心等式：Agent = Model + Harness

![核心等式：Agent = Model + Harness](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/agent-model-harness_1784689205.png)

这是 Harness Engineering 最核心的思想框架：[¹](#ref1) [²](#ref2)

```
Agent = Model + Harness
```

| 组成部分 | 职责 | 比喻 |
|---|---|---|
| **Model** | 提供认知能力（理解、推理、生成） | 赛车的引擎 |
| **Harness** | 管理状态、调用工具、安全约束、编排流程 | 赛车的底盘 + 转向系统 + 刹车 + 导航 |

**关键洞察**：模型提供智能，Harness 让智能变得可用（"The model contains the intelligence and the harness makes that intelligence useful"）。业界经验表明，Agent 产品的大部分工程量在 Harness 而非模型本身——从工具链集成到安全护栏，从上下文管理到错误重试，这些工程工作远超模型接入本身。[²](#ref2)

### 具体职责划分

| 谁负责 | 具体例子 |
|---|---|
| **Model 负责** | 理解用户意图、规划解题步骤、生成代码、判断何时该调用工具 |
| **Harness 负责** | 把用户消息格式化后传给模型、实际执行工具调用、管理对话历史、在输出中检测危险操作并拦截、失败后自动重试、记录全程日志 |

> 换句话说：Model 做"思考"，Harness 做"一切其他事情"。一个没有 Harness 的 Model 就像一个有想法但没有手脚、没有记忆的大脑——它能想，但做不了任何事。

## 三、Harness Engineering 与 Prompt/Context Engineering 的关系

三者是包含关系，层层递进：[¹](#ref1) [³](#ref3)

```
┌─────────────────────────────────────────────┐
│  Harness Engineering（最外层）              │
│  ┌───────────────────────────────────────┐  │
│  │  Context Engineering                  │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Prompt Engineering             │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

| 层级 | 范围 | 关注什么 |
|---|---|---|
| **Prompt Engineering** | 单次提示词的措辞技巧 | "怎么写一段好 Prompt" |
| **Context Engineering** | 模型每次接收到的完整信息（Prompt + 历史 + 检索 + 工具输出） | "给模型喂什么信息" |
| **Harness Engineering** | 整个运行时系统（Context + 工具 + 沙盒 + 编排 + 反馈回路 + 安全） | "怎么让整个 Agent 稳定可靠地工作" |

> 简单说：Prompt Engineering 是写好一句话；Context Engineering 是组织好一整页材料；Harness Engineering 是搭好整个工作台。

## 四、Harness 的核心组件

![Harness 的核心组件](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/harness-components-matched-v2_1784705412.png)

一个完整的 Agent Harness 通常包含以下组件：[²](#ref2) [³](#ref3) [⁴](#ref4)

| 组件 | 说明 | 示例 |
|---|---|---|
| **System Prompt** | 系统级行为指令 | 角色定义、输出格式约束、安全规则 |
| **Tools & MCP Servers** | 模型可调用的外部能力 | 代码执行器、搜索 API、数据库查询 |
| **Execution Sandbox** | 隔离的代码执行环境 | Docker 容器、E2B 沙盒 |
| **Orchestration Logic** | 多步/多 Agent 编排 | 路由器、子 Agent 调度、重试策略 |
| **Memory & State** | 持久化状态管理 | 短期（Context）+ 长期（向量库 / KV 存储） |
| **Deterministic Middleware** | 确定性的前/后处理 | 输入过滤、输出格式校验、护栏 |
| **Feedback Loops** | 执行后反馈信号 | 测试运行结果、Linter 输出、用户评分 |

### 快速概览：七句话理解七个组件

1. **System Prompt** 是 Agent 的"出厂设置"，定义角色、规则和输出格式。
2. **Tools & MCP Servers** 是 Agent 的"手脚"，赋予它搜索、执行、查询等外部能力。
3. **Execution Sandbox** 是 Agent 的"安全屋"，让生成的代码在隔离环境中运行而不影响宿主系统。
4. **Orchestration Logic** 是 Agent 的"大脑调度器"，决定何时调用谁、失败了怎么办。
5. **Memory & State** 是 Agent 的"记忆系统"，管理短期上下文和跨会话的长期知识。
6. **Deterministic Middleware** 是 Agent 的"安检门"，用确定性规则过滤输入、校验输出。
7. **Feedback Loops** 是 Agent 的"纠错机制"，把执行结果反馈给 Agent 形成自我修正循环。

### 各组件详细说明

**System Prompt**：是 Harness 中最基础的 Feedforward Control。它定义了 Agent 的角色身份、行为边界和输出规范。好的 System Prompt 是结构化的（用 Markdown/XML 组织），而非散文式的自然语言段落。

**Tools & MCP Servers**：赋予 Agent 感知和影响外部世界的能力。工具设计遵循单一职责原则，每个工具只做一件事。MCP 提供了标准化的工具接入协议。

**Execution Sandbox**：Agent 生成的代码必须在隔离环境中执行，防止恶意/错误代码影响宿主系统。这是安全性的核心保障。

**Orchestration Logic**：决定 Agent 的"控制流"——是单步执行还是多步循环？失败时重试几次？何时升级给人工？多个子 Agent 如何协调？

**Memory & State**：短期记忆即当前 Context Window 内的信息；长期记忆通过向量数据库或文件系统持久化跨会话的知识。

**Deterministic Middleware**：在模型输入/输出两端的确定性处理逻辑。包括 Guardrails（安全护栏）、Schema 验证、PII 脱敏等。这些是"硬规则"，不受模型概率性的影响。

**Feedback Loops**：将执行结果回传给 Agent，形成自我修正循环。经典案例：运行测试 → 测试失败 → 把错误信息反馈给 Agent → Agent 修复代码 → 再次运行测试。

这些内容在题库中有直接考查。

## 五、Harness 对基准分数的影响（实证）

以 SWE-bench 为例：[⁵](#ref5) [⁶](#ref6)

| 现象 | 说明 |
|---|---|
| **同一模型 + 不同 Harness → 分数差异巨大** | 在 SWE-bench 上，仅更换 Agent 的 Harness（提示词策略、工具链、重试机制）而不更换底层模型，解题率可以从 20% 跳到 50%+ |
| **排行榜实质在比 Harness** | 业界普遍认为 SWE-bench 排行榜很大程度上衡量的是 Harness 的工程质量而非纯模型能力 |
| **Harness 优化的边际效益 > 换模型** | 对于已有合格底座模型的团队，投资 Harness 优化的 ROI 通常高于升级模型 |

### 具体案例

SWE-bench Verified 排行榜上，使用相同底座模型（如 Claude 3.5 Sonnet）的不同 Agent 系统，分数差异可达 20 个百分点以上。这些差异完全来自 Harness 的设计差异：

- **搜索策略**：有的 Agent 用关键词搜索定位代码，有的用语义搜索，有的两者混合
- **编辑工具**：有的逐行编辑，有的整文件替换，有的用 diff 模式
- **验证循环**：有的改完就交，有的自动跑测试并修复直到通过
- **上下文管理**：有的一次性灌入所有文件，有的按需逐步加载

> 💡 **启示**：不要只盯着模型选型，Harness Engineering 才是产品竞争力的真正护城河。

## 六、Harness Engineering 作为新兴学科

Harness Engineering 正在成为一个独立的工程学科，它融合了：[³](#ref3) [⁴](#ref4)

- **控制论**：Feedforward（前馈）+ Feedback（反馈）控制模型
- **软件工程**：模块化设计、测试驱动、可观测性
- **DevOps/SRE**：持续集成、监控告警、灰度发布
- **安全工程**：最小权限、沙盒隔离、护栏

其目标是回答一个核心问题：**如何让 AI Agent 在生产环境中可靠、安全、可控地工作？**

### 入门建议

如果你刚开始接触 Harness Engineering，以下是三个具体的起步方向：

1. **从 System Prompt 工程化入手**：把团队项目的 System Prompt 从散文式描述改为结构化格式（Markdown/XML 分节），加入明确的角色定义、行为边界和输出格式约束。这是投入最小、见效最快的 Harness 优化。

2. **为 Agent 建立基本的 Feedback Loop**：在 Agent 生成代码后自动运行测试或 Linter，将结果反馈给 Agent。哪怕只是"跑测试 → 把失败信息塞回 Prompt"这一步，也能显著提升 Agent 的可靠性。

3. **用 SWE-bench 做 Harness 对比实验**：选择一个开源 Agent 框架（如 LangGraph、SWE-Agent），在 SWE-bench 的一个子集上对比不同 Harness 配置（如搜索策略、编辑工具）的解题率，直观感受 Harness 对结果的影响。

## 七、实际产品中的 Harness Engineering

以当今主流 AI 编码 Agent 为例，来看 Harness 的各组件如何落地：[²](#ref2) [⁴](#ref4)

| 产品 | Harness 特色 |
|---|---|
| **Claude Code** | CLAUDE.md 作为项目级 Feedforward；Agent Skills 作为可复用行为模块；Bash/文件工具链；Confirmation Gate 要求高危操作确认 |
| **Cursor** | .cursorrules 定义编码规范（Feedforward）；Composer 模式做多步编排；内置 Linter 反馈循环 |
| **Devin** | 完整的沙盒开发环境（VM 级隔离）；浏览器 + 终端 + 编辑器多工具协同；Plan-Execute-Verify 编排逻辑 |
| **SWE-Agent** | 基于 ACI（Agent-Computer Interface）设计的工具集；自定义 Shell 环境；搜索/编辑/测试工具链 |

这些产品的共同点：**模型可能是同一个（如都用 Claude 3.5 Sonnet），但 Harness 的差异决定了产品体验和任务完成率的巨大差别。**

### 一个具象化的 Harness 工作流示例

以 Claude Code 处理一个 Bug Fix 任务为例：

```
1. [Feedforward] 加载 CLAUDE.md 项目规范 + 用户请求
2. [Model]       理解 bug 描述，规划修复步骤
3. [Tool Call]   读取相关源文件（Bash tool: cat src/xxx.ts）
4. [Model]       生成修复代码
5. [Tool Call]   写入修改（Edit tool）
6. [Feedback]    运行测试（Bash tool: npm test）→ 2 个用例失败
7. [Model]       分析测试失败原因
8. [Tool Call]   修改代码
9. [Feedback]    再次运行测试 → 全部通过
10. [Gate]       展示 diff 给用户确认
```

这 10 步中，只有第 2、4、7 步是 Model 在"思考"，其余全是 Harness 在工作。

## 常见误解

- ❌ 不是 Prompt Engineering 的同义词（Prompt 只是 Harness 的子集）

## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》§10.3 Harness Engineering. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** LangChain. *The Anatomy of an Agent Harness*. 2025. https://www.langchain.com/blog/the-anatomy-of-an-agent-harness

<a id="ref3"></a>**[3]** Truong Pham. *Harness Engineering: The Emerging Discipline of Making AI Agents Reliable*. Dev.to, 2025. https://dev.to/truongpx396/harness-engineering-the-emerging-discipline-of-making-ai-agents-reliable-42gf

<a id="ref4"></a>**[4]** Anthropic. *Building Effective Agents*. 2024. https://www.anthropic.com/research/building-effective-agents

<a id="ref5"></a>**[5]** SWE-bench. *SWE-bench Leaderboards*. https://www.swebench.com/

<a id="ref6"></a>**[6]** Epoch AI. *What skills does SWE-bench Verified evaluate?*. 2025. https://epoch.ai/publications/what-skills-does-swe-bench-verified-evaluate
