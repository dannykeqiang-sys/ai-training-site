# Skill 机制详解

Skill 是 Claude Code 中最精妙的扩展机制之一。如果说 MCP 给 Agent 装上了"工具箱"，那么 Skill 就是给 Agent 配了一本"员工手册"——它不增加新工具，但教会 Agent 在什么场景、按什么流程、用什么格式来完成工作。本篇从机制原理到实践编写，帮你深入理解 Skill 的设计哲学和使用方法。

## 一、为什么需要 Skill

在引入 Skill 之前，AI 编码工具面临三个典型问题：

| 问题 | 表现 |
|---|---|
| **工作流程不标准** | 同一个团队的不同成员，让 Agent 做"代码审查"的流程各不相同，质量参差不齐 |
| **输出质量不稳定** | Agent 自由发挥时，有时遗漏关键步骤，有时输出格式不符合团队规范 |
| **Token 浪费严重** | 为了让 Agent "记住"流程，每次对话都要重复输入大段指令 |

**Skill（技能）** 正是为了解决这些问题而设计的——它是 Agent 的**预定义能力模块**，围绕特定任务定义结构化的行为规范。[¹](#ref1)

> 一句话理解：Skill 是给 Agent 的"专属员工手册"，告诉它在特定场景下**该怎么做、用什么工具、输出什么格式**。

## 二、Skill 的本质：行为指令而非工具

这是理解 Skill 最重要的一点：**Skill 不是工具（不执行操作），而是行为规范（指导 Agent 如何行动）**。[¹](#ref1) [²](#ref2)

### 类比理解

| 概念 | 现实类比 | 在 Agent 中的角色 |
|---|---|---|
| **MCP** | 给新员工开通系统账号、配置工具权限 | 提供可调用的外部工具（文件读写、API 调用、数据库查询等） |
| **Skill** | 给新员工一本员工手册和 SOP | 定义在特定场景下如何使用工具、遵循什么流程、输出什么格式 |

### Skill 的内容构成

一个 Skill 通常包含以下要素：[¹](#ref1)

| 要素 | 说明 | 示例 |
|---|---|---|
| **任务流程** | 按步骤定义 Agent 应执行的操作序列 | "第一步：检查错误类型；第二步：查阅日志；第三步：生成修复建议" |
| **判断规则** | 定义条件分支和决策逻辑 | "如果是 4xx 错误，检查客户端参数；如果是 5xx 错误，检查服务端日志" |
| **输出格式** | 规范 Agent 的输出结构 | "输出格式：问题描述 + 根因分析 + 修复步骤 + 预防措施" |
| **参考资料引用** | 指向外部文档或文件 | "当错误码为 5xx 时，参考 `references/server-errors.md`" |

### 关键区别

```
MCP Tool（工具）：
  - 有实际的执行能力（读文件、调 API、查数据库）
  - 通过 JSON Schema 定义输入输出
  - 被调用后返回执行结果

Skill（技能）：
  - 没有执行能力，只有行为指导
  - 通过自然语言定义流程和规则
  - 被激活后注入到 Agent 的上下文中
  - 可以引用 MCP 工具来实际执行操作
```

> 💡 **Skill 本身不会让 Agent "能做"新事情，但会让 Agent "更好地做"已有事情。** 就像一个经验丰富的员工和新员工的区别——工具相同，但工作流程和判断力不同。

## 三、三层加载架构

![Skill 三层渐进式加载架构](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/skill-three-layers_1784689077.png)

Skill 最核心的设计创新是**渐进式加载（Progressive Disclosure）**——不是一次性把所有指令塞入上下文，而是分层、按需加载。[³](#ref3) [²](#ref2)

### 架构概览

| 层级 | 名称 | 加载时机 | 加载内容 | 大小参考 |
|---|---|---|---|---|
| **L1** | 索引层（Frontmatter） | Agent 启动时 | 所有 Skill 的 `name` + `description`，聚合到 `<available_skills>` 列表 | 系统默认预算上限 15,000 字符 |
| **L2** | 指令层（SKILL.md） | 用户任务匹配到某个 Skill 时 | 该 Skill 的完整指令内容 | 建议 < 5000 词（约 800 行）[³](#ref3) |
| **L3** | 参考层（Helper Resources） | Skill 指令引用外部文件时 | `references/`（文本，消耗 Token）或 `assets/`（二进制，不消耗 Token）[³](#ref3) | 按需加载 |

> 💡 **`references/` vs `assets/` 的区别**：`references/` 目录下的文本文件通过 `Read` 工具加载到上下文中，**消耗 Token**（如一个 10KB 的 Markdown 文件）；`assets/` 目录下的文件仅通过路径引用，**不加载到上下文**，不消耗 Token（如 HTML 模板、图片等）。[³](#ref3)

### 加载过程详解

以下用一个"API 错误诊断" Skill 为例，演示三层加载的完整过程：

```
阶段一：Agent 启动（L1 索引层加载）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
加载所有 Skill 的索引：

  Skill 1: api-error-handler  — "Diagnose and resolve API errors..."
  Skill 2: code-review        — "Perform structured code review..."
  Skill 3: db-migration       — "Safely execute database migrations..."
  ...（共 20 个 Skill）

  总消耗 ≈ 1500 tokens
  ─────────────────────────────────────
  此时 Agent 只知道"有哪些 Skill 可用"，
  但不知道每个 Skill 的具体指令。


阶段二：用户提问（L2 指令层加载）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
用户："帮我看看这个 API 返回 502 错误"
  │
  ▼ 模型语义匹配 → 激活 "api-error-handler" Skill
  │
  加载该 Skill 的完整指令：

  "## API 错误诊断流程
   1. 确认错误类型（4xx / 5xx）
   2. 调用 health-check 工具检查服务状态
   3. 根据状态码分支处理：
      - 4xx：检查请求参数和认证信息
      - 5xx：读取 references/server-errors.md
   4. 输出格式：[问题] + [根因] + [修复步骤]"

  本次加载 ≈ 800 tokens


阶段三：按需引用（L3 参考层加载）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Skill 指令中提到"当错误码为 5xx 时，读取 references/server-errors.md"
  │
  ▼ Agent 判断当前是 502 错误（5xx），需要加载参考文件
  │
  加载 references/server-errors.md：

  "502 Bad Gateway: 通常由上游服务不可用引起..."
  "503 Service Unavailable: 服务过载或维护中..."

  本次加载 ≈ 500 tokens
```

**总消耗**：1500（L1） + 800（L2） + 500（L3） = 2800 tokens

如果采用 MCP 式的急切加载（把所有 Skill 的完整指令全部加载），20 个 Skill × 800 tokens/Skill = 16000 tokens——**差距超过 5 倍**。[³](#ref3)

### 与 MCP 加载方式的对比

```
MCP（急切加载）：
  启动时 → 加载所有 Server 的所有 Tool 描述 → 全部放入上下文
  问题：工具越多，Token 消耗越大，上下文越拥挤

Skill（渐进式加载）：
  启动时 → 仅加载索引（name + description）
  匹配时 → 加载匹配 Skill 的完整指令
  需要时 → 加载引用的参考文件
  优势：用有限 Token 覆盖更多能力
```

## 四、触发机制：Model-driven Activation

![Skill 由模型根据描述自主触发](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/skill-activation_1784689076.png)

Skill 的触发机制是**模型驱动的自主激活（Model-driven Activation）**，这与传统的关键词匹配或规则引擎有本质区别。[¹](#ref1) [²](#ref2)

### 纯 LLM 推理，非算法路由

这是最关键的一点：**系统不使用嵌入向量、分类器或模式匹配来决定激活哪个 Skill**。[³](#ref3)

工作原理：
1. 所有可用 Skill 的 name + description 被聚合成文本列表，嵌入到 Skill 工具的 prompt 字段中
2. 模型通过原生语言理解能力阅读这个列表
3. 模型将用户意图与 Skill description 进行语义匹配
4. 模型在**前向传播（forward pass）过程中**做出决策，激活最匹配的 Skill

> 💡 **这意味着 Skill 的选择完全发生在 Transformer 的前向传播中，而非应用层代码。** 没有正则表达式、没有关键词匹配、没有基于机器学习的意图检测——这是纯粹的 LLM 推理。[³](#ref3)

### description 写作的重要性

因为触发完全依赖 description，所以 **description 的质量直接决定了 Skill 能否被正确激活**。

| 类型 | 示例 | 问题 / 优势 |
|---|---|---|
| ❌ 过于模糊 | `"Handle errors"` | 几乎所有涉及错误的请求都可能匹配，导致误触发 |
| ❌ 过于技术化 | `"Process HTTP 4xx/5xx status codes via diagnostic pipeline"` | 用户用自然语言描述问题时可能无法匹配 |
| ✅ 聚焦用户意图 | `"Diagnose and resolve API errors, including HTTP 4xx and 5xx responses, by analyzing logs and suggesting fixes"` | 既覆盖用户常见表述，又明确了 Skill 的边界 |
| ✅ 祈使语气 | `"Use this skill when the user encounters API errors and needs systematic diagnosis and resolution"` | 明确告诉模型何时激活，降低误判率 |

### description 写作建议

根据实践经验，好的 description 应遵循以下原则：[²](#ref2)

1. **使用祈使语气**：以 `"Use this skill when..."` 开头
2. **聚焦用户意图**：描述用户想做什么，而非技术实现细节
3. **包含典型触发场景**：列出常见的用户表述方式
4. **明确边界**：说明 Skill 不处理什么（避免与其他 Skill 冲突）

```markdown
# ✅ 好的 description 示例
---
description: >
  Use this skill when the user encounters API errors (HTTP 4xx/5xx)
  and needs systematic diagnosis. Covers: identifying error types,
  checking service health, analyzing logs, and generating fix suggestions.
  Does NOT cover: network connectivity issues or client-side JavaScript errors.
---

# ❌ 差的 description 示例
---
description: Error handling module
---
```

## 五、如何编写一个高质量的 Skill

### 5.1 Skill 的文件结构

在 Claude Code 中，Skill 以 Markdown 文件形式存放在项目的 `.claude/skills/` 目录下：[²](#ref2)

```
.claude/
└── skills/
    ├── api-error-handler.md     ← L2 指令文件（SKILL.md）
    ├── code-review.md
    ├── db-migration.md
    ├── references/              ← L3 参考文件（文本，消耗 Token）
    │   ├── server-errors.md
    │   └── review-checklist.md
    ├── scripts/                 ← L3 辅助脚本
    │   └── lint-check.sh
    └── assets/                  ← L3 资源文件（不消耗 Token）
        └── report-template.html
```

一个 Skill 文件的基本结构：

```markdown
---
description: >
  Use this skill when the user needs to diagnose and resolve
  API errors, including HTTP 4xx and 5xx responses.
---

# API 错误诊断

## 诊断流程

1. **确认错误类型**
   - 检查 HTTP 状态码
   - 区分客户端错误（4xx）和服务端错误（5xx）

2. **收集上下文信息**
   - 调用 `health-check` 工具确认服务状态
   - 读取最近的日志记录

3. **按类型分支处理**
   - **4xx 错误**：检查请求参数、认证信息、权限配置
   - **5xx 错误**：读取 `references/server-errors.md` 查找已知问题

4. **输出诊断报告**
   - 问题描述
   - 根因分析
   - 修复步骤
   - 预防建议
```

### Frontmatter 字段参考

Skill 文件通过 YAML frontmatter 声明元数据。[³](#ref3)

| 字段 | 是否必需 | 用途 |
|---|---|---|
| `description` | ✅ 必需 | 模型选择 Skill 的主要依据（L1 索引层内容） |
| `allowed-tools` | 可选 | 逗号分隔的工具列表，支持通配符（如 `Bash(git:*)`），激活时自动授权 |
| `model` | 可选 | 覆盖当前模型（如 `"claude-opus-4-20250514"` 或 `"inherit"`） |
| `disable-model-invocation` | 可选 | 设为 `true` 时，模型不会自动激活该 Skill，仅支持用户通过 `/skill-name` 手动调用 |

> ⚠️ **Skill 必须包含 `description` 字段才会出现在 `<available_skills>` 列表中**，否则模型无法感知到该 Skill 的存在。[³](#ref3)

### 5.2 instructions 的设计原则

Skill 的指令部分（即正文内容）是 Agent 实际遵循的操作指南，应遵循以下原则：

| 原则 | 说明 | 反例 |
|---|---|---|
| **清晰** | 每个步骤都有明确的动作和判断条件 | "检查服务是否正常"（什么叫"正常"？） |
| **可执行** | Agent 能根据指令直接行动 | "理解用户需求"（太抽象，无法执行） |
| **有边界** | 明确 Skill 不处理的情况 | 没有说明 Skill 的适用范围 |
| **有输出规范** | 定义预期的输出格式 | 不规定格式，每次输出结构不同 |

### 5.3 常见反模式

| 反模式 | 问题 | 改进建议 |
|---|---|---|
| **过于宽泛** | `"处理所有类型的错误"` → 指令过于笼统，无法有效指导 | 聚焦特定场景，如"处理 API 5xx 错误" |
| **指令过长** | 单个 Skill 超过 800 行（约 5000 词），L2 加载消耗过多 Token [³](#ref3) | 拆分为多个 Skill，或将详细内容放入 L3 参考文件 |
| **缺乏边界** | 没有说明不适用的场景，导致误触发 | 在 description 中明确排除条件 |
| **硬编码具体值** | `"当错误码是 502 时..."` → 无法覆盖其他 5xx 错误 | 使用分类和规则，而非穷举具体值 |

## 六、Skill 的生命周期

一个 Skill 从创建到执行，经历以下阶段：

```
① 创建
   编写 Skill 文件（.md 格式），定义 description 和 instructions
       │
       ▼
② 注册
   将文件放入 .claude/skills/ 目录
   → Agent 下次启动时自动识别
       │
       ▼
③ 发现与索引加载（L1）
   Agent 启动时从四个来源并行扫描 Skill：[³](#ref3)
   - 用户级：~/.claude/commands/
   - 项目级：.claude/skills/
   - 插件提供的 Skill
   - 内置 Skill
   → 聚合到 <available_skills> 列表，Agent 知道"有哪些 Skill 可用"
       │
       ▼
④ 按需激活（L2 + L3）
   模型通过 LLM 推理匹配用户意图 → 加载完整指令和参考文件
   → 注入两条 user 消息：[³](#ref3)
     - 元数据消息（对用户可见）："<command-message>The "xxx" skill is loading</command-message>"
     - 指令消息（对用户隐藏）：完整的 SKILL.md 内容
   → Agent 知道"该怎么做"
       │
       ▼
⑤ 执行
   Agent 按照 Skill 指令执行任务（可使用预授权的 MCP 工具）
   → 输出符合规范的结果
```

### 跨会话持久化

Skill 文件存储在项目的 `.claude/skills/` 目录中，具有**跨会话持久性**：[²](#ref2)

- Skill 文件随项目代码一起管理，可以通过 Git 进行版本控制
- 每次启动新会话时，Agent 会重新加载 Skill 索引
- 修改 Skill 文件后，下一次会话自动生效，无需额外配置

> 💡 **这意味着 Skill 可以作为团队共享的"知识资产"**——把最佳实践写成 Skill，提交到代码仓库，团队所有成员的 Agent 都能遵循相同的工作流程。

## 七、Token 预算管理

在实际项目中，上下文窗口的 Token 预算是硬约束。Skill 的渐进式加载在 Token 管理上有显著优势。[³](#ref3)

### 量化对比

**场景：一个中型项目配置 20 个 Skill + 5 个 MCP Server（共 50 个 Tool）**

| 方案 | 启动时 Token 消耗 | 任务执行时 Token 消耗 |
|---|---|---|
| **纯 MCP**（50 个 Tool） | ≈ 8000 tokens（所有 Tool 描述急切加载） | 每次调用 Tool ≈ 500 tokens |
| **Skill 索引**（20 个 Skill） | ≈ 1500 tokens（仅 name + description） | 激活 1-2 个 Skill ≈ 800-1600 tokens |

### 如何用有限 Token 覆盖更多能力

Skill 的渐进式加载让你可以**"预定义多、每次激活少"**：

```
策略：
├─ 可以定义 50+ 个 Skill（索引消耗 ≈ 3000-4000 tokens）
├─ 每次任务只激活 1-2 个最相关的 Skill（≈ 800-1600 tokens）
└─ 参考文件按需加载（≈ 500 tokens/文件）

结果：
  覆盖 50+ 种场景，每次任务实际消耗 ≈ 2000-4000 tokens
```

对比 MCP：如果 50 个 Tool 全部急切加载，仅启动就要 ≈ 8000 tokens，且无法"选择性加载"。

### 实践建议

| 建议 | 说明 |
|---|---|
| **description 精简** | 每个 Skill 的 description 控制在 1-3 句话，避免索引层消耗过多 Token |
| **Skill 数量合理** | 建议 20-50 个 Skill，超过 50 个时索引层 Token 消耗明显增加 |
| **善用 L3 参考层** | 将不常用的详细文档放入参考文件，而非全部写在指令层 |
| **避免指令冗余** | 多个 Skill 之间的公共步骤可以抽取为共享参考文件 |

## 八、常见误解

- ❌ **"Skill 就是 Prompt 模板"**——Prompt 模板是单次使用的文本片段；Skill 是包含任务流程、判断规则、输出格式、参考文件引用的结构化行为模块，且具备渐进式加载能力 [¹](#ref1)

- ❌ **"Skill 让模型更聪明"**——Skill 不改变模型本身的推理能力，它只是优化了上下文的使用效率，让模型在特定场景下表现更稳定 [¹](#ref1)

- ❌ **"Skill 是 MCP 的替代品"**——两者解决完全不同的问题：MCP 解决"能不能接"（工具接入），Skill 解决"怎么用好"（行为规范）。最佳实践是两者结合使用 [¹](#ref1)

- ❌ **"Skill 越多越好"**——每个 Skill 的 description 都会消耗索引层 Token。过多的 Skill 不仅增加 Token 消耗，还可能因 description 语义重叠导致模型误判激活哪个 Skill [³](#ref3)

## 延伸阅读

Skill 与 MCP 的详细对比（包括加载方式、Token 消耗、适用场景等维度的对比分析），以及何时用 Skill、何时用 MCP 的决策框架，详见 [Skill 机制与 MCP 对比](./skill-mechanism-vs-mcp)。

## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》第 11 节 Skill. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** Anthropic. *Skills – Claude Code Documentation*. https://docs.anthropic.com/en/docs/claude-code/skills

<a id="ref3"></a>**[3]** Lee, H. *Claude Agent Skills: A First Principles Deep Dive*. 2025-10-26. https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/
