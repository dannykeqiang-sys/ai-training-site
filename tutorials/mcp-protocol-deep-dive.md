# MCP 协议详解

MCP（Model Context Protocol）是 AI 工具生态中最重要的协议之一。本篇从架构模型和工具调用流程两个层面，帮你建立对 MCP 的完整认知。

## 一、MCP 是什么

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 于 2024 年 11 月推出的一种**开放标准协议**，旨在统一大语言模型（LLM）与外部数据源、工具之间的通信方式。[¹](#ref1) [²](#ref2)

> 一句话定义：MCP 是 AI 世界的"USB-C 接口"——让工具开发者只需实现一次协议，即可被所有支持 MCP 的 AI 应用（Claude、Cursor、Cline 等）调用。[³](#ref3)

### 为什么需要 MCP？

在 MCP 出现之前，AI 应用与外部工具的集成面临三大痛点：[²](#ref2) [⁴](#ref4)

| 痛点 | 说明 |
|---|---|
| **协议碎片化** | OpenAI Function Calling、Google Tool Use、Anthropic Tool Use 各自有不同的 API 规范，工具每换一个平台都要重写适配代码 |
| **信息孤岛** | LLM 无法直接访问本地文件、数据库、实时 API 等外部信息 |
| **安全与交互不统一** | 不同平台对工具授权、参数校验、错误处理的实现各异 |

MCP 通过定义一套**统一的客户端-服务器架构**，一次性解决了上述问题。

## 二、MCP 架构三层模型

![MCP 架构：主机 - 客户端 - 服务器](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch2/images/mcp-architecture_1784685549.png)

MCP 采用**Host → Client → Server** 三层架构：[¹](#ref1) [³](#ref3)

```
┌─────────────────────────────────────────────────────┐
│  MCP Host（宿主应用）                                │
│  如 Claude Desktop / Cursor / Cline / IDE           │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │ MCP Client │  │ MCP Client │  │ MCP Client │   │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │
└────────┼────────────────┼────────────────┼──────────┘
         │                │                │
   ┌─────▼──────┐  ┌─────▼──────┐  ┌─────▼──────┐
   │ MCP Server │  │ MCP Server │  │ MCP Server │
   │  天气查询   │  │  数据库    │  │  文件系统   │
   └────────────┘  └────────────┘  └────────────┘
```

### 2.1 三个角色定义

| 角色 | 定义 | 实例 |
|---|---|---|
| **Host（宿主）** | 承载 AI 交互的终端应用，负责创建和管理 MCP Client 实例 | Claude Desktop、Cursor、Cline、Qoder |
| **Client（客户端）** | 由 Host 为每个 Server 连接创建的专用通信实例，负责协议协商和消息路由 | Host 内部组件，一个 Host 可管理多个 Client |
| **Server（服务器）** | 实际提供工具/资源/提示词的服务进程，可运行在本地或远程 | 天气查询 Server、GitHub Server、数据库 Server |

> 你可以把 Host 想象成一个浏览器，Client 就像浏览器为每个网站打开的一个标签页——每个标签页独立与对应的网站通信。

### 2.2 Server 暴露的三类能力（Primitives）

MCP Server 可以向 Client 暴露三种类型的能力：[¹](#ref1)

| 能力类型 | 用途 | 示例 |
|---|---|---|
| **Tools（工具）** | 可执行的函数/操作，由模型决定何时调用 | `query_weather(city)`, `search_database(sql)` |
| **Resources（资源）** | 可读取的数据源，类似 GET 请求 | 文件内容、数据库记录、API 响应 |
| **Prompts（提示模板）** | 预定义的 Prompt 模板，便于复用 | 代码审查模板、翻译模板 |

### 2.3 AI 编程工具的角色

在 MCP 架构中，**AI 编程工具（如 Claude Code、Cursor、Cline）同时扮演 Host 和 Client 角色**：[¹](#ref1) [³](#ref3)

- 作为 **Host**：它是用户直接使用的应用界面，管理 AI 交互生命周期
- 内含 **Client**：为每个连接的 MCP Server 维护一个 Client 实例
- **不是 Server**：编程工具本身不向其他应用暴露工具，而是消费其他 Server 提供的工具

## 三、MCP 在 Agent 工具调用中的完整流程

一次完整的 MCP 工具调用流程如下：[⁴](#ref4)

```
用户："今天北京天气如何？"
    │
    ▼
[Agent/Host] 组装 messages + tools 参数，调用 LLM
    │
    ▼
[LLM] 分析问题，返回 finish_reason: "tool_calls"
      tool_calls: [{ name: "mcp_query_weather_tool", arguments: { cityName: "北京" } }]
    │
    ▼
[Agent] 通过 MCP Client 调用对应 MCP Server 的工具
    │
    ▼
[MCP Server] 执行实际查询（调用天气 API），返回结果
    │
    ▼
[Agent] 将工具返回结果以 tool role 追加到 messages，再次调用 LLM
    │
    ▼
[LLM] 基于工具结果生成自然语言回答：
      "今天北京天气晴朗，当前温度为 6℃。"
```

> 关键认知：**MCP 工具不是被 LLM 直接调用的**——LLM 只输出"我想调用什么工具、参数是什么"的指令，**由 Agent（Host）执行实际调用**，然后把结果反馈给 LLM。[⁴](#ref4)

## 四、MCP 的传输协议

![MCP 三种传输方式对比](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch2/images/mcp-transports_1784685600.png)

MCP 支持多种传输协议，适用于不同场景：[¹](#ref1) [⁵](#ref5)

### 4.1 三种传输方式对比

| 传输方式 | 通信机制 | 适用场景 | 优缺点 |
|---|---|---|---|
| **stdio** | 标准输入/输出（进程间通信） | 本地 MCP Server，Host 直接启动 Server 进程 | ✅ 简单、低延迟、无需网络配置<br/>❌ 仅限本地、Host 需管理 Server 生命周期 |
| **SSE（Server-Sent Events）** ⚠️ | HTTP 长连接 + 事件流 | 远程 MCP Server，Client 主动连接 | ✅ 支持远程、Server 可独立部署<br/>❌ 需要网络连接、Server 需暴露端口<br/>⚠️ **已废弃（2025-03）**，新项目建议使用 Streamable HTTP |
| **Streamable HTTP** | 标准 HTTP 请求/响应 | 无状态场景，Serverless 部署 | ✅ 无状态、易扩展、兼容性好<br/>❌ 不支持流式推送、每次请求独立 |

### 4.2 如何选择传输方式

```
决策树：
├─ Server 运行在本地（与 Host 同一台机器）？
│  └─ 是 → 使用 stdio（最简单、性能最好）
├─ Server 需要远程访问？
│  └─ 是 → 使用 Streamable HTTP（SSE 已废弃，不推荐新项目使用）
└─ Server 部署在 Serverless 环境（如 AWS Lambda）？
   └─ 是 → 使用 Streamable HTTP
```

### 4.3 实际案例

| 场景 | 传输方式 | 配置方式 |
|---|---|---|
| 本地文件系统 Server | stdio | `"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]` |
| 远程数据库 Server | Streamable HTTP | `"url": "https://api.example.com/mcp/database"` |
| Serverless 天气查询 | Streamable HTTP | `"url": "https://api.example.com/mcp/weather"` |

## 五、MCP Server 开发注意事项

开发 MCP Server 时，除了遵循通用的工具设计原则（详见《Agent 工具调用与安全》），还需注意 MCP 特有的问题：

| 关注点 | MCP 特有考虑 | 建议 |
|---|---|---|
| **跨平台兼容性** | 同一 Server 可能被多个 Host（Claude、Cursor、Cline）调用 | 避免依赖特定 Host 的特性，保持协议层中立 |
| **资源声明** | MCP 支持声明 Resources（只读数据源）和 Prompts（模板） | 静态数据用 Resources，可执行操作用 Tools |
| **能力协商** | Host 和 Server 在连接时协商支持的能力 | 明确声明 Server 支持的能力列表，避免运行时错误 |
| **生命周期管理** | stdio 模式下 Host 管理 Server 进程，SSE/HTTP 模式下 Server 独立运行 | 根据部署模式设计合理的启动/停止逻辑 |

> 关于工具设计的通用原则（单一职责、参数精简、最小权限等），详见教程《Agent 工具调用与安全》。

## 六、MCP 与其他协议的对比

MCP 不是唯一的工具调用协议，了解其与其他方案的区别有助于选型：

| 协议/方案 | 标准化程度 | 跨平台支持 | 适用场景 |
|---|---|---|---|
| **MCP** | 开放标准（Anthropic 主导） | ✅ Claude、Cursor、Cline、Qoder 等 | 需要统一工具生态、跨平台复用 |
| **OpenAI Function Calling** | 私有实现 | ❌ 仅 OpenAI 平台 | 已深度集成 OpenAI、无需跨平台 |
| **LangChain Tools** | 框架内标准 | ⚠️ 仅 LangChain 生态 | 使用 LangChain 框架开发 |
| **自定义 API** | 无标准 | ❌ 需逐个适配 | 特殊需求、遗留系统 |

**何时选择 MCP**：
- 希望工具被多个 AI 应用复用（开发一次，到处使用）
- 需要标准化的工具接入流程
- 团队使用多种 AI 编程工具（Claude + Cursor + Cline）

## 常见误解


- ❌ "MCP 是一种大模型"——MCP 是协议，不是模型
- ❌ "MCP 是 Function Calling 的别名"——Function Calling 是各平台私有实现，MCP 是统一标准

- ❌ "AI 编程工具是 MCP Server"——它是 Host/Client 端，不是工具提供者

- ❌ "接入 MCP 后模型变得更聪明"——MCP 只提供工具通道，不改变模型能力


## 参考资料

<a id="ref1"></a>**[1]** Model Context Protocol. *Architecture Overview*. https://modelcontextprotocol.io/docs/concepts/architecture

<a id="ref2"></a>**[2]** 刘立成(文辺). 《关于 AI 的一些新名词和概念》第三节 MCP. ATA, 2025-04-18. https://ata.atatech.org/articles/11020405255 *（内网）*

<a id="ref3"></a>**[3]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》第 9 节 MCP. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref4"></a>**[4]** 钱红凯(趋势). 《AI实践之路：LLM、RAG、MCP、Agent、A2A，掌握AI相关基础原理》第四节 MCP. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*

<a id="ref5"></a>**[5]** Anthropic. *Introducing the Model Context Protocol*. 2024-11-25. https://www.anthropic.com/news/model-context-protocol
