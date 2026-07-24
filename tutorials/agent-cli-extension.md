# Agent CLI 命令扩展

Agent 连接外部世界不只靠 MCP 协议——CLI 命令执行是另一条同样重要的通道。当你需要 `git push`、`docker build`、`kubectl apply` 时，没有哪个 MCP Server 比直接执行命令行工具更直接。本篇聚焦 Agent 通过 Bash/Shell 工具执行 CLI 命令来调用外部系统的扩展机制，帮助你理解这种"万能后备"方案的原理、实践和安全模型。

> 一句话理解：CLI 扩展是 Agent 的"万能后备"——没有专用协议时，用命令行解决。

## 一、什么是 CLI 命令扩展

**CLI 命令扩展** 是指 Agent 通过内置的 **Bash/Shell 工具** 执行命令行工具（`gh`、`aws`、`docker`、`kubectl`、`curl` 等）来与外部系统交互的机制。它是 Tool Use 的一种具体实现形式——模型通过 Function Calling 协议调用 Shell 工具，工具的实现是执行 Shell 命令并返回 `stdout`/`stderr`。[¹](#ref1) [²](#ref2)

### 与其他扩展机制的关系

在第一章「协议与扩展」分组中，我们已经学习了两种扩展机制：

| 扩展机制 | 本质 | 核心教程 |
|---|---|---|
| **MCP（标准协议）** | 统一的工具接入协议，结构化的 Client-Server 通信 | [MCP 协议深度解析](./mcp-protocol-deep-dive.md) |
| **Skill（行为指令）** | 预定义的行为规范，指导 Agent 何时用、怎么用工具 | [Skill 机制与 MCP 对比](./skill-mechanism-vs-mcp.md) |
| **CLI 命令扩展（本篇）** | 通过 Shell 工具直接执行命令行工具 | 本篇 |

三者不是替代关系，而是**互补关系**：MCP 提供标准化工具接入，Skill 提供行为规范，CLI 命令提供"万能后备"——当没有专用 MCP Server 时，Agent 仍能通过执行命令行工具完成任务。

### 主流框架的支持情况

CLI 命令扩展已被主流 Agent 框架广泛支持：

| 框架/产品 | Shell 工具名称 | 定位 | 来源 |
|---|---|---|---|
| **OpenAI Agents SDK** | `ShellTool` | 归类为 "Local/runtime execution tools"，支持本地执行和托管容器执行两种模式 | [openai-agents-python](https://github.com/openai/openai-agents-python/blob/main/docs/tools.md) [¹](#ref1) |
| **LangChain** | `ShellToolMiddleware` | 包含持久化 Shell 会话、多种执行策略（Host/Docker/Codex Sandbox）、输出截断和超时控制、PII 检测 | [langchain shell_tool.py](https://github.com/langchain-ai/langchain/blob/main/libs/langchain_v1/langchain/agents/middleware/shell_tool.py) [²](#ref2) |
| **Claude Code** | 内置 Bash 工具 | 支持沙箱配置（`sandbox.enabled`、`autoAllowBashIfSandboxed`、`allowedDomains` 等），同时支持 MCP 工具和 Bash 命令执行 | [claude-code](https://github.com/anthropics/claude-code) [³](#ref3) |

## 二、Agent 如何执行 CLI 命令

### 2.1 技术流程

CLI 命令执行遵循标准的 Agent Loop（详见 [Agent 执行循环](./agent-loop.md)），每轮循环包含四个阶段：

```
用户："帮我查看这个项目的 Git 提交历史"
    │
    ▼
[Agent 推理] 分析问题 → 决定使用 Bash 工具执行 git log 命令
    │
    ▼
[构造命令] 生成命令字符串："git log --oneline -20"
    │
    ▼
[调用 Bash 工具] Harness 执行 Shell 命令，捕获 stdout/stderr
    │
    ▼
[解析结果] 将命令输出（20 行提交记录）作为 tool_result 返回给 LLM
    │
    ▼
[Agent 推理] 基于命令输出，生成自然语言总结返回给用户
```

关键点：**LLM 本身不执行命令**——它只输出"我想执行什么命令"的指令，由 Harness（Agent 宿主）负责实际执行，然后把结果反馈给 LLM。这与 MCP 工具调用的模式完全一致（参见 [MCP 协议深度解析](./mcp-protocol-deep-dive.md) 第三节），区别仅在于工具的**实现方式**：MCP 工具通过协议调用 Server，Shell 工具通过子进程执行命令。

### 2.2 真实案例：Claude Code 的 Bash 工具

Claude Code 是 Agent 使用 CLI 命令的典型实现。在 [Agent 执行循环](./agent-loop.md) 中展示过一个完整的 Bug Fix 流程，其中 Bash 工具被多次调用：[⁴](#ref4) [⁵](#ref5)

```
1. [Feedforward] 加载 CLAUDE.md 项目规范 + 用户请求
2. [Model]       理解 bug 描述，规划修复步骤
3. [Tool Call]   读取相关源文件（Bash tool: cat src/xxx.ts）    ← CLI 调用
4. [Model]       生成修复代码
5. [Tool Call]   写入修改（Edit tool）
6. [Feedback]    运行测试（Bash tool: npm test）→ 2 个用例失败   ← CLI 调用
7. [Model]       分析测试失败原因
8. [Tool Call]   修改代码
9. [Feedback]    再次运行测试 → 全部通过                         ← CLI 调用
10. [Gate]       展示 diff 给用户确认
```

这个 10 步流程中，第 3、6、9 步都是通过 Bash 工具执行 CLI 命令——第 3 步用 `cat` 读取文件，第 6 和第 9 步用 `npm test` 运行测试。CLI 命令贯穿了 Agent 工作的始终。

### 2.3 持久化 Shell 会话 vs 每次独立执行

Shell 命令的执行方式有两种：

| 模式 | 说明 | 优缺点 |
|---|---|---|
| **每次独立执行** | 每条命令启动一个新的子进程，执行完即销毁 | ✅ 隔离性好、无状态污染<br/>❌ 无法保持环境状态（如 `cd` 后目录不保留） |
| **持久化 Shell 会话** | 维护一个长期运行的 Shell 进程，多条命令在同一会话中执行 | ✅ 保持环境状态（目录、环境变量）<br/>❌ 状态污染风险、需要管理会话生命周期 |

LangChain 的 `ShellToolMiddleware` 提供了持久化 Shell 会话（`ShellSession`）的支持，允许 Agent 在一个持续的 Shell 会话中执行多条命令。[²](#ref2) Claude Code 的 Bash 工具默认每次独立执行，但工作目录（cwd）在调用间保持一致。

## 三、典型使用场景

CLI 命令扩展覆盖了 Agent 与外部系统交互的几乎所有场景：

### 3.1 版本控制：`git`、`gh`

```bash
# 查看代码变更
git diff HEAD~3..HEAD

# 创建 Pull Request
gh pr create --title "fix: resolve null pointer" --body "..."

# 查看 CI 状态
gh run list --limit 5
```

### 3.2 云服务管理：`aws`、`gcloud`、`kubectl`

```bash
# 查看 Kubernetes Pod 状态
kubectl get pods -n production

# 查看 AWS S3 存储桶
aws s3 ls s3://my-bucket/

# 查看 GCP 计算实例
gcloud compute instances list
```

### 3.3 容器操作：`docker`

```bash
# 构建镜像
docker build -t my-app:latest .

# 查看运行中的容器
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### 3.4 API 调用：`curl`、`httpie`

```bash
# 调用 REST API
curl -s -X GET https://api.example.com/health | jq .

# 发送 POST 请求
curl -X POST https://api.example.com/deploy \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"env": "staging"}'
```

### 3.5 包管理与构建：`npm`、`pip`

```bash
# 安装依赖
npm install

# 运行测试
npm test

# Python 依赖管理
pip install -r requirements.txt
```

### 3.6 系统管理：`ls`、`cat`、`grep`、`find`

```bash
# 搜索代码中的 TODO
grep -rn "TODO" src/ --include="*.ts"

# 查找大文件
find . -name "*.log" -size +100M

# 查看磁盘使用
du -sh */ | sort -rh | head -10
```

> 💡 **经验法则**：如果一个操作有现成的 CLI 工具，Agent 就可以通过 Bash 工具执行它，无需等待专用的 MCP Server 开发。

## 四、安全模型：白名单 + 沙盒

![CLI 执行安全模型：白名单 + 沙盒](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/cli-security-model_1784689125.png)

CLI 命令执行赋予了 Agent 极大的行动能力，但也带来了严峻的安全挑战。正如 [HITL 与不可逆操作](./hitl-and-irreversible-operations.md) 中所讨论的，Shell 命令被列为高风险操作，需要严格的权限控制。[⁶](#ref6)

### 4.1 CLI 调用的三大安全挑战

| 挑战 | 说明 | 示例 |
|---|---|---|
| **命令注入** | Agent 构造的命令包含未预期的恶意内容 | 用户输入 `"文件名; rm -rf /"` 被拼入命令 |
| **权限越界** | Agent 执行了超出任务范围的命令 | 修复 bug 时意外修改了生产数据库配置 |
| **数据泄露** | 命令输出包含敏感信息（密钥、Token） | `cat` 输出了包含 API Key 的 `.env` 文件 |

### 4.2 Claude Code 的沙箱配置

Claude Code 在其源码中提供了详细的沙箱配置示例（`examples/settings/settings-bash-sandbox.json`），展示了多层安全防护：[³](#ref3)

```jsonc
// Claude Code 沙箱配置示例
{
  "sandbox": {
    "enabled": true
  },
  "autoAllowBashIfSandboxed": true,
  "allowedDomains": [
    "api.github.com",
    "registry.npmjs.org"
  ]
}
```

**配置项解读**：

| 配置项 | 作用 | 安全意义 |
|---|---|---|
| `sandbox.enabled` | 是否启用沙箱隔离 | 在隔离环境中执行命令，防止影响宿主系统 |
| `autoAllowBashIfSandboxed` | 沙箱模式下是否自动放行 Bash 命令 | 减少人工确认频率，但仅限沙箱内有效 |
| `allowedDomains` | 允许访问的网络域名白名单 | 限制网络访问范围，防止数据外泄 |

### 4.3 命令白名单机制

参考 [HITL 与不可逆操作](./hitl-and-irreversible-operations.md) 中的分级授权原则，CLI 命令可按风险等级分级：[⁶](#ref6)

| 风险等级 | 命令类型 | 确认方式 | 示例 |
|---|---|---|---|
| **低** | 只读命令 | 自动执行 | `ls`、`cat`、`grep`、`git log`、`git diff` |
| **中** | 本地写操作 | 批量确认或事后 Review | `npm install`、`git commit`、文件编辑 |
| **高** | 系统级操作 | 逐个确认 | `docker rm`、`rm -rf`、`chmod`、Shell 脚本 |
| **极高** | 外部影响操作 | 强制确认 + 审计 | `kubectl delete`、`aws s3 rm`、`curl -X DELETE` |

Claude Code 的权限模式体现了这一分级思想：在 Default 模式下，每次文件写入和 Shell 命令都要求确认；在 AcceptEdits 模式下，文件编辑自动通过但 Shell 命令仍需确认；在 Auto 模式下，用分类器自动判断风险等级。[⁶](#ref6)

### 4.4 沙盒隔离技术

主流框架提供了多种沙盒隔离方案：

| 隔离级别 | 技术方案 | 使用者 | 安全程度 |
|---|---|---|---|
| **进程级** | 子进程隔离，限制工作目录 | Claude Code（默认） | 基础 |
| **容器级** | Docker 容器隔离 | LangChain Docker 执行策略 [²](#ref2) | 中等 |
| **VM 级** | 虚拟机隔离（gVisor / Firecracker） | OpenAI Agents SDK 托管容器 [¹](#ref1) | 高 |

LangChain 的 `ShellToolMiddleware` 支持三种执行策略：Host（宿主机直接执行）、Docker（容器内执行）和 Codex Sandbox（沙盒环境执行），开发者可根据安全需求选择。[²](#ref2)

## 五、CLI 调用 vs MCP 工具：核心差异

![CLI 命令与 MCP 工具对比](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/cli-vs-mcp-compare_1784689123.png)

CLI 命令扩展和 MCP 工具是 Agent 连接外部世界的两条主要通道。理解两者的差异，才能做出合理的选型决策。

### 5.1 对比表

| 维度 | CLI 命令扩展 | MCP 工具 |
|---|---|---|
| **结构化程度** | 低——命令输入输出均为非结构化文本 | 高——参数有 JSON Schema，返回有明确格式 |
| **工具发现** | 无——Agent 需要"知道"有哪些 CLI 工具可用 | 自动发现——MCP Server 声明可用工具列表 |
| **安全性** | 较低——命令注入风险高，需要额外沙箱 | 较高——参数校验由协议层保证 |
| **使用门槛** | 低——无需额外开发，直接执行已有工具 | 较高——需要开发或接入 MCP Server |
| **互操作性** | 低——命令格式因工具而异，无统一标准 | 高——统一协议，跨平台复用 |
| **输出解析** | 困难——需要 LLM 理解自由格式文本 | 容易——结构化 JSON 返回 |
| **状态管理** | 手动——通过持久化会话或环境变量管理 | 内置——MCP Server 可管理自身状态 |
| **错误处理** | 依赖 `exit code` 和 `stderr` | 协议级错误码和错误消息 |

### 5.2 何时用 CLI、何时用 MCP

```
决策树：
├─ 有现成的 MCP Server 可用？
│  └─ 是 → 优先使用 MCP（结构化、安全、可复用）
├─ 需要调用已有的 CLI 工具（git、docker、kubectl 等）？
│  └─ 是 → 使用 CLI 命令扩展（零配置、即用即走）
├─ 操作是一次性的、临时的？
│  └─ 是 → CLI 更经济（不值得开发 MCP Server）
├─ 操作需要高频、稳定、团队协作？
│  └─ 是 → MCP 更合适（标准化、可维护）
└─ 两者都可以？
   └─ 混合使用（现代产品通常同时支持两种方式）
```

### 5.3 现代产品的混合方案

Claude Code 是混合方案的典型代表——它**同时支持 MCP 工具和 Bash 命令执行**：[³](#ref3) [⁵](#ref5)

- **MCP 工具**：用于结构化的外部服务集成（如 GitHub Server、数据库 Server）
- **Bash 工具**：用于执行 CLI 命令（如 `npm test`、`git diff`、`cat` 等）
- **Edit/Read 工具**：用于文件操作（内置的结构化工具）

在 [Harness Engineering 概述](./harness-engineering-overview.md) 的产品对比中，Claude Code 的 Harness 特色就包括"Bash/文件工具链"与 MCP 工具的协同工作。[⁵](#ref5)

> 💡 **关键洞察**：CLI 和 MCP 不是二选一的关系，而是互补的。CLI 提供灵活性和即时性，MCP 提供标准化和安全性。成熟的 Agent 产品通常两者兼备。

## 六、优缺点总结

### 6.1 优点

| 优点 | 说明 |
|---|---|
| **复用已有工具链** | 开发者日常使用的 `git`、`docker`、`kubectl` 等工具，Agent 可以直接调用，无需额外开发 |
| **灵活性高** | 可以组合任意命令（管道、脚本），适应各种场景 |
| **无需额外协议** | 不需要实现 MCP Server，不需要定义 Schema，直接执行 |
| **零配置启动** | 只要系统上安装了 CLI 工具，Agent 就能立即使用 |

### 6.2 缺点

| 缺点 | 说明 |
|---|---|
| **非结构化输出解析困难** | CLI 工具的输出格式各异，LLM 需要理解自由格式的文本，容易遗漏关键信息 |
| **安全风险高** | 命令注入、权限越界、数据泄露等风险需要额外防护（详见第四节） |
| **不可靠** | 同一命令在不同环境下行为可能不同（路径、权限、工具版本差异） |
| **缺乏类型安全** | 命令参数是字符串拼接，没有 Schema 校验，容易构造出无效命令 |
| **不可移植** | 依赖本地安装的 CLI 工具，不同环境需要不同的工具集 |

## 七、从 CLI 到 MCP 的迁移路径

CLI 命令扩展是快速启动的好方式，但随着使用深入，某些场景值得升级到正式的 MCP 集成。

### 7.1 何时该从 CLI 升级到 MCP

| 信号 | 说明 |
|---|---|
| **高频调用** | 某个 CLI 命令被 Agent 频繁调用（如每天数百次 `curl` 调用某个 API） |
| **稳定性要求高** | CLI 命令的输出格式变化导致 Agent 解析失败 |
| **团队协作需求** | 多人使用同一套 CLI 命令，需要标准化和版本管理 |
| **安全合规要求** | 需要对工具调用进行细粒度的权限控制和审计 |
| **跨平台复用** | 同一工具需要被不同的 AI 应用（Claude、Cursor、Cline）复用 |

### 7.2 迁移路径示例

以 GitHub 集成为例：

**阶段 1：临时 CLI 调用**

Agent 直接执行 `gh` 命令：
```bash
gh pr list --state open --json number,title
gh pr view 42 --json reviews
```
- ✅ 快速可用
- ❌ 输出格式需 LLM 解析，无类型安全

**阶段 2：正式 MCP 集成**

接入 GitHub MCP Server，提供结构化工具：
```json
{
  "name": "list_pull_requests",
  "parameters": {
    "state": { "type": "string", "enum": ["open", "closed"] },
    "limit": { "type": "number" }
  }
}
```
- ✅ 结构化输入输出、类型安全、跨平台复用
- ❌ 需要部署和维护 MCP Server

### 7.3 判断标准

```
问自己三个问题：
1. 这个 CLI 调用每周会被执行多少次？
   └─ > 100 次 → 考虑迁移到 MCP
2. CLI 输出的格式变化是否导致过 Agent 出错？
   └─ 是 → 考虑迁移到 MCP（结构化输出更可靠）
3. 团队中是否有多人需要相同的工具能力？
   └─ 是 → 考虑迁移到 MCP（一次开发，多人复用）
```

> 💡 **经验法则**：CLI 是"原型验证"，MCP 是"生产就绪"。先用 CLI 快速验证需求，确认价值后再投入 MCP 开发。

## 八、常见误解

- ❌ **"CLI 调用不安全，应该禁用"**——正确的做法是通过沙箱 + 白名单 + 分级确认来管控风险，而非一刀切禁用。Claude Code 的 `autoAllowBashIfSandboxed` 配置就体现了这一思路：在沙箱环境中自动放行命令，兼顾安全与效率。[³](#ref3)

- ❌ **"有了 MCP 就不需要 CLI"**——MCP 和 CLI 是互补关系。即使接入了 MCP Server，Agent 仍需要 CLI 来执行本地操作（`npm test`、`git diff`）、调用没有 MCP Server 的工具、以及进行临时性的探索操作。[³](#ref3) [⁵](#ref5)

- ❌ **"CLI 调用就是 Agent 在写代码"**——Agent 执行 CLI 命令是在**调用工具**，不是在写代码。模型通过 Function Calling 输出"我想执行 `git log`"的指令，Harness 负责实际执行。这与调用 MCP 工具的模式一致，只是工具的底层实现不同。参见 [MCP 协议深度解析](./mcp-protocol-deep-dive.md) 中的关键认知。

## 参考资料

<a id="ref1"></a>**[1]** OpenAI. *Tools — OpenAI Agents SDK Documentation*. https://github.com/openai/openai-agents-python/blob/main/docs/tools.md

<a id="ref2"></a>**[2]** LangChain. *ShellToolMiddleware Source Code*. https://github.com/langchain-ai/langchain/blob/main/libs/langchain_v1/langchain/agents/middleware/shell_tool.py

<a id="ref3"></a>**[3]** Anthropic. *Claude Code — Sandbox Configuration Examples*. https://github.com/anthropics/claude-code

<a id="ref4"></a>**[4]** 本教程. *Agent 执行循环（Loop）*. [agent-loop.md](./agent-loop.md)

<a id="ref5"></a>**[5]** 本教程. *Harness Engineering 概述*. [harness-engineering-overview.md](./harness-engineering-overview.md)

<a id="ref6"></a>**[6]** 本教程. *Human-in-the-Loop（HITL）与不可逆操作*. [hitl-and-irreversible-operations.md](./hitl-and-irreversible-operations.md)

<a id="ref7"></a>**[7]** Model Context Protocol. *Specification*. https://github.com/modelcontextprotocol/specification
