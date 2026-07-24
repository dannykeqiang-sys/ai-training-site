# Skill 机制与 MCP 对比

Skill（技能）和 MCP（Model Context Protocol）是 AI 编码工具中两个互补的扩展机制。MCP 解决"工具能不能接"（统一协议），Skill 解决"工具怎么用好"（行为规范）。本篇聚焦两者的核心差异和互补关系。

## 一、Skill vs MCP：核心差异

![Skill 与 MCP 的核心差异](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/skill-vs-mcp_1784689134.png)

两者解决的问题完全不同，是**互补关系**：[¹](#ref1)

| 维度 | MCP | Skill |
|---|---|---|
| **解决的问题** | "能不能接"——统一工具接入标准 | "怎么用好"——定义何时用、怎么用、输出什么格式 |
| **本质** | 通信协议（Protocol） | 行为规范（Instruction） |
| **类比** | 给新员工开通系统账号和工具权限 | 给新员工一本员工手册和 SOP |
| **加载方式** | **急切加载**：启动时加载所有工具描述 | **渐进式加载**：按需分层加载 |
| **Token 消耗** | 高（工具越多越贵） | 低（即使 20 个 Skill 初始也仅千余 tokens） |
| **包含的内容** | 工具名称、描述、输入输出 Schema | 任务流程、判断规则、输出格式、参考资料引用 |
| **谁定义** | 工具/服务开发者 | 应用开发者 / 使用者 |
| **是否执行操作** | 是——实际调用外部工具 | 否——只指导 Agent 如何行动（可引用 MCP 工具） |

### 互补关系图示

```
┌─────────────────────────────────────────┐
│  Skill（员工手册）                        │
│  "收到 API 错误时：                      │
│   1. 先调用 health-check 工具           │
│   2. 根据状态码查阅错误文档              │
│   3. 输出格式：问题 + 根因 + 修复步骤"   │
│                    │                     │
│                    ▼ 引用                 │
│  MCP Tool: health-check（实际执行能力）   │
└─────────────────────────────────────────┘
```

> 💡 **没有 MCP，Skill 只能"纸上谈兵"（有手册但没有工具）；没有 Skill，MCP 工具"不知所措"（有工具但不知道什么时候该用）。**

## 二、Skill 的核心价值：渐进式加载

Skill 相比 MCP 最精妙的设计差异在于**渐进式加载机制**：[¹](#ref1)

### 三层加载结构

| 层级 | 加载时机 | 加载内容 | Token 消耗 |
|---|---|---|---|
| **L1** | Agent 启动时 | 所有 Skill 的 `name` + `description`（仅索引） | 极少（20 个 Skill 约 1000-2000 tokens） |
| **L2** | 用户任务匹配到某个 Skill 时 | 该 Skill 的完整指令内容 | 中等（建议 < 500 行） |
| **L3** | Skill 指令引用外部文件时 | 参考文档 | 按需（仅在需要时加载） |

```
Agent 启动：加载 20 个 Skill 的 name + description ≈ 1500 tokens
    │
    ▼ 用户问："帮我处理这个 API 错误"
匹配到 "api-error-handler" Skill → 加载完整指令 ≈ 800 tokens
    │
    ▼ Skill 指令："当错误码为 5xx 时，读取 references/server-errors.md"
按需加载 server-errors.md ≈ 500 tokens
```

> 对比 MCP：MCP 是**急切加载**，Agent 启动时就把所有 Server 的所有 Tool 描述全部加入上下文。如果接入 10 个 Server、50 个 Tool，启动即消耗大量 Token。[¹](#ref1)

### Token 预算管理：Skill 的优势

在实际项目中，Token 预算是硬约束。Skill 的渐进式加载让你可以**用有限的 Token 覆盖更多的能力**：

**典型场景：一个中型项目配置 20 个 Skill + 5 个 MCP Server**

| 机制 | 启动时消耗 | 任务执行时消耗 | 总消耗 |
|---|---|---|---|
| **纯 MCP 方案** | 50 个 Tool 描述 ≈ 8000 tokens | 每次调用工具 ≈ 500 tokens | 高（8000 + 调用次数×500） |
| **Skill + MCP 混合** | 20 个 Skill 索引 ≈ 1500 tokens | 激活 1-2 个 Skill ≈ 800-1600 tokens + 工具调用 | 低（1500 + 激活 Skill + 调用） |

**关键洞察**：Skill 让你可以"预定义 100 种能力"但"每次只激活最相关的 1-2 种"，而 MCP 必须"把所有工具的说明书都放在桌面上"。

## 三、何时用 Skill、何时用 MCP、何时两者结合

![Skill / MCP 选型决策树](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/skill-mcp-decision_1784689124.png)

### 决策框架

```
你的需求是什么？
    │
    ├─ "Agent 需要调用外部工具/API"
    │   └─ 用 MCP（这是 MCP 的核心能力）
    │
    ├─ "Agent 需要在特定场景下遵循特定流程"
    │   └─ 用 Skill（这是 Skill 的核心能力）
    │
    ├─ "Agent 需要在特定场景下调用特定工具，并遵循特定流程"
    │   └─ Skill + MCP（Skill 指导流程，MCP 提供工具）
    │
    └─ "Agent 需要动态选择多个工具完成复杂任务"
        └─ Skill 编排 + MCP 执行（Skill 作为"指挥官"）
```

### 真实场景对比

**场景 1：代码审查**

| 方案 | 实现方式 | 问题 |
|---|---|---|
| **纯 MCP** | 接入 `read_file`、`grep`、`lint` 等工具，Agent 自行决定审查流程 | Agent 可能遗漏关键检查项，审查质量不稳定 |
| **纯 Skill** | 定义"代码审查 SOP"，但没有工具 | Skill 无法实际执行文件读取和代码分析 |
| **Skill + MCP** | Skill 定义审查流程（先检查安全、再检查性能、最后检查规范），引用 MCP 工具执行 | Agent 按标准流程审查，质量稳定可控 |

**场景 2：API 错误诊断**

| 方案 | 实现方式 | 问题 |
|---|---|---|
| **纯 MCP** | 接入 `curl`、`log_viewer`、`health_check` 工具 | Agent 不知道诊断顺序，可能反复试错 |
| **Skill + MCP** | Skill 定义诊断流程："先查状态码→再查日志→再查配置→给出修复建议"，引用 MCP 工具 | 诊断效率高，输出格式统一 |

**场景 3：数据库迁移**

| 方案 | 实现方式 | 问题 |
|---|---|---|
| **纯 MCP** | 接入 `db_query`、`db_migrate` 工具 | 风险极高，Agent 可能跳过安全检查直接执行 |
| **Skill + MCP** | Skill 强制要求："必须先备份→必须在测试环境验证→必须人工确认→才能执行迁移" | 安全可控，关键步骤不可跳过 |

> 💡 **经验法则**：如果任务"步骤固定、风险较高、需要标准化输出"，优先用 Skill 编排；如果任务"工具导向、步骤灵活"，可以只用 MCP。

### 权衡：Skill 编排的投入产出

Skill 编排有明确的收益，但也有设计和维护成本。以下场景值得权衡：

| 场景 | Skill 编排的收益 | 需要权衡的因素 |
|---|---|---|
| **探索性任务**（如"帮我看看这个代码库有什么问题"） | 可以定义探索方法论，避免遗漏 | 任务目标开放，流程难以标准化，可能需要更灵活的 Skill 设计 |
| **一次性简单操作**（如文件编码转换） | 可以避免常见坑（如 BOM 头丢失） | 任务简单，Skill 设计成本可能高于直接操作 |
| **高度个性化任务**（如每人标准不同的代码审查） | 可以提供基础框架，用户再自定义覆盖 | 统一流程可能无法满足所有偏好，需要 Skill 留足灵活空间 |

> 💡 **经验法则**：Skill 的设计成本与"场景的标准化程度"正相关——越容易总结出通用流程的场景，Skill 的投入产出比越高。

## 四、Skill 的定义

**Skill（技能）** 是 AI 编码工具中的**预定义能力模块**——它不是单条 Prompt，而是围绕特定任务、工具调用流程和输出规范的**结构化行为设计**。[¹](#ref1)

> 一句话定义：Skill 是给 Agent 的"专属员工手册"，告诉它在特定场景下该怎么做、用什么工具、输出什么格式。

### 触发机制：Model-driven Activation

Skill 的触发**完全依赖 `description` 字段**，由模型自主判断当前任务是否匹配：[¹](#ref1)

- 不是关键词硬编码匹配
- 模型根据用户意图与 description 的语义相关性决定是否激活
- description 写作建议：使用祈使语气（"Use this skill when..."），聚焦用户意图

## 常见误解


- ❌ "Skill 就是 Prompt 模板"——Skill 还包含工具调用流程、判断规则、参考文件引用等

- ❌ "Skill 让模型更聪明"——Skill 不改变模型能力，只优化上下文使用效率

- ❌ "Skill 是 MCP 的替代品"——两者解决不同问题，互补而非替代
- ❌ "MCP 比 Skill 更先进"——两者在不同维度发挥作用，无优劣之分

## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》第 11 节 Skill. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** Anthropic. *Agent Skills in the SDK – Claude Code Docs*. https://code.claude.com/docs/en/agent-sdk/skills

<a id="ref3"></a>**[3]** Lee, H. *Claude Agent Skills: A First Principles Deep Dive*. 2025-10-26. https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/
