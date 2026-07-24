# Agent 工具调用与安全

Function Calling 让模型从"只会说"变成"能做事"，工具设计质量直接决定 Agent 能否正确理解和使用自身能力。同时，Agent 拥有行动能力后，安全问题从"输出不当内容"升级为"执行不当操作"——注入攻击、权限失控和缺乏隔离是三大核心威胁。

## 一、Function Calling

![Function Calling 完整时序](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/function-calling-flow_1784689068.png)

### 1.1 什么是 Function Calling

**Function Calling** 是现代 LLM API 提供的结构化机制，允许开发者向模型声明可用工具，模型判断何时需要调用工具，并以结构化格式输出调用指令。[1](#ref1)

> **关键理解**：LLM 本身不执行函数。它只是输出"调用请求"（包含函数名和参数），由应用层代码负责实际执行并将结果返回。

### 1.2 工具定义格式

现代 Function Calling API 使用**结构化格式**定义工具——包括工具名称（name）、描述（description）和参数规范（parameters）。模型根据这些定义来判断何时以及如何调用工具。

### 1.3 工具调用的完整流程

```
1. 开发者传入工具定义列表
2. 用户提问："今天北京天气怎么样？"
3. LLM 判断需要调用 get_weather，输出 tool_calls
4. 应用层执行实际的天气 API 调用
5. 将工具结果返回给 LLM
6. LLM 基于工具结果生成最终回答
```

### 1.4 单一职责原则

**每个工具只做一件事，且把这件事做好。**[3](#ref3) [5](#ref5)

**反例**：`manage_database`（同时负责增删改查）
**正例**：`db_query`、`db_insert`、`db_update`、`db_delete`（四个独立工具）

设计指南：命名明确、参数精简（3-5 个以内）、边界清晰、组合优于聚合。

### 1.5 工具结果的返回与上下文影响

工具执行结果的返回方式会显著影响 Agent 的后续推理：

| 返回模式 | 说明 | 适用场景 |
|---|---|---|
| **直接返回** | 工具结果直接作为 tool_result 返回给 LLM | 结果简短、结构化（如天气数据、查询结果） |
| **摘要返回** | 对大型结果进行摘要或截取关键部分 | 搜索结果、日志分析、长文档 |
| **引用返回** | 返回文件路径或 ID，LLM 按需读取 | 大型文件、数据库查询、图片生成 |

**上下文窗口管理**：工具结果会占用上下文窗口。频繁调用返回大量数据的工具可能导致早期对话内容被截断。实践中可通过 `max_tokens` 参数限制结果大小，或在工具设计时就考虑返回精简数据。

**错误处理**：工具执行失败时，应返回明确的错误信息（而非抛异常让应用崩溃）。LLM 需要理解失败原因才能决定下一步行动（重试、换工具、还是告知用户）。

| 模式 | 适用场景 |
|---|---|
| **并行** | 多个查询相互独立、无数据依赖 |
| **串行** | 存在数据依赖或逻辑依赖 |

### 1.6 工具描述的重要性

工具的 `description` 是 LLM 决策的**唯一依据**——模型完全靠 description 判断何时调用、如何调用、调用哪个工具。好的 description 应说清能力、边界、输入和输出。

> Function Calling 赋予了 Agent 行动能力，但这种能力也带来了新的安全风险。接下来我们讨论 Agent 安全的核心问题。

## 二、Agent 安全

![Agent 四大安全风险与防线](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/agent-security-risks_1784689053.png)

### 2.1 为什么 Agent 安全更严峻

标准 LLM 最坏情况是输出不当内容。但 **Agent 拥有行动能力**——能调用工具、执行代码、修改文件。一旦被操控，后果从"输出垃圾"升级为"实际破坏"。

OWASP LLM Top 10 (2025) 的 **LLM06: Excessive Agency** 专门针对此风险。

### 2.2 Prompt Injection 在 Agent 中的风险

| 攻击类型 | 注入位置 | Agent 特有风险 |
|---|---|---|
| **直接注入** | 用户输入 | Agent 可能据此调用危险工具 |
| **间接注入** | 工具返回值、检索文档 | Agent 读取被污染数据后执行恶意指令 |
| **跨工具注入** | 工具 A 的输出传给工具 B | 形成攻击链 |

**间接注入是 Agent 时代最危险的攻击面**：Agent 主动读取外部内容（网页/邮件/工具返回值），每个数据源都可能被注入恶意指令。

### 2.3 真实攻击场景

Agent 面临的典型攻击场景包括：

**1. 工具描述注入**
攻击者在 MCP Server 的工具 `description` 字段中嵌入隐藏指令（如 HTML 注释），诱导 Agent 执行恶意操作。2025 年安全研究人员已在多个开源 MCP Server 中发现此类漏洞。

**2. RAG 知识库投毒**
恶意用户向 RAG 知识库注入包含隐藏指令的文档。后续所有检索到该文档的 Agent 都可能被诱导执行非预期操作。

**3. 外部内容注入**
攻击者在 GitHub Issue、网页、邮件等 Agent 可能读取的外部内容中嵌入恶意指令。如果 Agent 未对内容进行安全过滤，可能执行恶意请求。

**4. 跨工具链式攻击**
Agent 将工具 A 的输出传递给工具 B 时，工具 A 返回内容中的恶意指令在工具 B 的上下文中被"激活"，形成攻击链。每个工具单独看都"正常"，但组合起来形成攻击路径。

> 💡 **关键认知**：Agent 的攻击面远大于标准 LLM。每个工具调用、每次外部内容读取都是潜在的攻击入口。防御需要覆盖整个工具调用链路。

### 2.4 Confirmation Gates（确认门）

在 Agent 执行高风险操作之前，**必须暂停并请求人类确认**。

| 风险等级 | 操作类型 | 是否需要确认门 |
|---|---|---|
| **高** | 删除文件、发送邮件、金融交易 | **必须** |
| **中** | 创建/修改文件 | 推荐 |
| **低** | 读取文件、搜索信息 | 不需要 |

### 2.5 最小权限原则

**Agent 只应被赋予完成当前任务所必需的最小权限和工具集**。

| 层面 | 实施方式 |
|---|---|
| **工具数量** | 只注册必需的工具 |
| **工具权限** | 只需读就不给写 |
| **操作范围** | 文件操作限制在指定目录 |
| **网络访问** | 白名单制 |

### 2.6 代码沙盒（Sandboxing）

Agent 生成的代码必须在隔离环境中执行，防止：文件系统破坏、数据泄露、系统接管、资源耗尽。

| 方案 | 适用场景 |
|---|---|
| **Docker 容器** | 通用代码执行，大多数团队的首选 |
| **gVisor / Firecracker** | 高安全要求（金融、医疗） |
| **专用沙盒服务（E2B）** | 希望快速接入、不想自建基础设施 |

### 2.7 综合防御策略（Defense in Depth）

```
第 1 层：输入过滤（拦截已知攻击模式）
第 2 层：最小权限（限制可用工具和权限范围）
第 3 层：确认门（高风险操作必须人工批准）
第 4 层：沙盒隔离（代码在隔离环境中执行）
第 5 层：输出验证（校验 Agent 输出的合规性）
第 6 层：监控告警（异常行为实时检测）
```

## 参考资料

<a id="ref1"></a>**[1]** OpenAI. *Function Calling*. https://developers.openai.com/api/docs/guides/function-calling

<a id="ref2"></a>**[2]** 钱红凯(趋势). 《AI实践之路》第四章 MCP · Function Calling demo. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*

<a id="ref3"></a>**[3]** Anthropic. *Tool Use: Define Tools*. https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools

<a id="ref4"></a>**[4]** OWASP Gen AI Security Project. *LLM01:2025 Prompt Injection*. https://genai.owasp.org/llmrisk/llm01-prompt-injection/

<a id="ref5"></a>**[5]** Anthropic. *Building Effective Agents*. 2024-12. https://www.anthropic.com/research/building-effective-agents
