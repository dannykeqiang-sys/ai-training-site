# HITL 与不可逆操作

## 一、核心定义

### 1.1 Human-in-the-Loop（HITL，人在回路）

**Human-in-the-Loop** 是指在 AI 系统的决策和执行链路中，保留人类作为**关键审批节点**的设计原则。AI 不能完全自主行动，必须在特定关卡处暂停、等待人类确认后才继续执行。[^1][^2]

> 一句话：**AI 提方案，人拍板；AI 做执行，人验收。**

### 1.2 Confirmation Gates（确认门禁）

**Confirmation Gates** 是 HITL 原则的工程实现——在 Agent 执行流程中设置的"检查站"，当 Agent 准备执行高风险操作时，系统暂停执行并向人类发出确认请求，只有人类明确批准后才继续。[^2][^3]

### 1.3 不可逆操作（Irreversible Operations）

**不可逆操作** 是指一旦执行就**无法撤销或回滚**的操作。这类操作是 HITL 和 Confirmation Gates 最核心的保护对象。[^2]

### 1.4 三者关系

HITL 是设计原则——在 AI 执行链路中保留人类审批节点；Confirmation Gates 是 HITL 的工程实现——在流程中设置具体检查站；不可逆操作是 HITL 最核心的保护对象——因为这些操作一旦执行就无法撤回。

## 二、为什么 AI 时代必须 HITL

### 2.1 AI 的"能力-可靠性"悖论

AI Agent 的能力越强，它能做的事情越多（读写文件、执行命令、调用 API、操作数据库）——但它的**判断力并不随能力线性增长**。一个能删除整个数据库的 Agent，并不比只能读文件的 Agent 更"懂"该不该删。[^1][^4]

### 2.2 LLM 的固有缺陷

| 缺陷 | 风险 |
|---|---|
| **幻觉** | Agent 可能基于错误理解执行操作 |
| **指令误解** | 模糊的用户指令被 Agent 解读为极端操作 |
| **Prompt Injection** | 外部恶意内容诱导 Agent 执行非预期操作 |
| **过度自信** | Agent 不会主动说"我不确定，先不做" |

### 2.3 损害的不对称性

AI 犯错的代价可能远超人类犯错——因为 AI 执行速度极快、规模极大。一个人类删错一个文件可能需要 30 秒才意识到；一个 Agent 可能在 3 秒内删完整个目录。

## 三、何时必须 HITL

核心原则：**操作的可逆性越低、影响范围越大，HITL 的必要性越高。**

### 3.1 必须 HITL 的场景

| 场景 | 原因 |
|---|---|
| **数据删除**（DROP TABLE、rm -rf） | 不可逆，数据无法恢复 |
| **资金转账/支付** | 涉及真金白银，转出后难以追回 |
| **生产环境部署** | 影响真实用户，回滚有时间窗口 |
| **权限变更**（赋予 admin 权限） | 可能被滥用，撤回有延迟 |
| **发送外部通信**（邮件、通知、短信） | 发出后无法撤回，影响外部关系 |
| **基础设施变更**（DNS、证书、网络规则） | 影响全局可用性 |

### 3.2 可以自动执行的场景

| 场景 | 原因 |
|---|---|
| 读取文件/数据 | 只读，不改变状态 |
| 代码格式化 | 可逆，且有 Git 保护 |
| 运行测试 | 只读验证，不产生副作用 |
| 搜索/检索操作 | 无状态变更 |
| 文件编辑（有 Git） | 可通过 Git 回滚 |

## 四、不可逆操作的处理原则

![不可逆操作前的人工确认闸门](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/hitl-confirmation-gate_1784689210.png)

### 4.1 原则一：默认拒绝（Deny by Default）

对于不可逆操作，系统默认不允许 Agent 自动执行。必须有人类的**显式批准（Explicit Approval）**。[^3][^5]

### 4.2 原则二：最小权限（Principle of Least Privilege）

Agent 只应该拥有完成当前任务所需的最小权限集合。不需要删除权限就不给删除权限，不需要写入权限就只给读取权限。[^4]

### 4.3 原则三：预览而非执行（Preview Before Execute）

对于不可逆操作，Agent 应先展示**操作预览**（"我将要执行以下操作：删除表 users 中 id < 100 的记录，共 87 条"），等人确认后才实际执行。[^2][^3]

### 4.4 原则四：分级授权（Tiered Permissions）

不同风险等级的操作适用不同的确认机制：

| 风险等级 | 确认方式 | 示例 |
|---|---|---|
| **低** | 自动执行，事后日志 | 读取文件、搜索 |
| **中** | 批量确认 / 事后 Review | 编辑文件、安装依赖 |
| **高** | 逐个确认 | Shell 命令、API 调用 |
| **极高** | 多因素确认 + 审计 | 生产数据删除、资金操作 |

## 五、典型不可逆操作识别

### 5.1 属于不可逆操作

第三节已列出数据删除、资金转账、生产部署、权限变更、外部通信、基础设施变更等典型场景。以下是第三节未覆盖的补充场景：

| 操作类型 | 典型示例 |
|---|---|
| **合约/法律行为** | 电子签名、合同提交、订单确认 |
| **代码发布** | 生产环境部署（尤其是移动端 App 发布，用户需手动更新才能回滚） |

### 5.2 不属于不可逆操作

| 操作类型 | 为什么可逆 |
|---|---|
| **文件编辑**（在 Git 管理下） | `git checkout` 即可恢复 |
| **创建新文件** | 删除即可 |
| **安装依赖** | 卸载即可 |
| **运行测试** | 只读行为，无副作用 |
| **读取数据** | 不改变任何状态 |
| **本地分支操作** | 可以重建或删除 |

## 六、Plan-Build-Verify 三段式工作流

![Plan-Build-Verify 三段式工作流](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/plan-build-verify_1784689198.png)

以下是完整工作流。

### 6.1 三阶段定义

```
Plan（规划）→ Build（构建）→ Verify（验证）
```

| 阶段 | 主导者 | 核心动作 | 产出物 |
|---|---|---|---|
| **Plan** | 人 + AI 协作 | 定义需求、拆解任务、制定方案 | 规格文档 / 实现计划 |
| **Build** | AI 为主 | 按计划生成代码、执行操作 | 代码 / 配置变更 |
| **Verify** | 人 + 自动化 | 运行测试、代码 Review、验收 | 通过/驳回决定 |

### 6.2 各阶段的 HITL 要求

| 阶段 | HITL 要求 | 说明 |
|---|---|---|
| **Plan → Build 过渡** | ✅ 必须人工确认 | 人审批方案后 AI 才开始实现 |
| **Build 内部** | 🟡 视操作而定 | 可逆操作可自动执行；不可逆操作需确认 |
| **Build → Verify 过渡** | 自动触发 | Build 完成自动进入验证 |
| **Verify 结果判定** | ✅ 人最终决定 | 测试全通过 + 人工 Review 后才算完成 |

### 6.3 循环特性

Plan-Build-Verify 不是单次直线流程，而是可以**循环迭代**的：

```
Plan → Build → Verify → (不通过) → Plan 调整 → Build 修正 → Verify → (通过) → Done
```

## 七、Confirmation Gates 的工程实现

### 7.1 Claude Code 的权限模式（真实案例）

Claude Code 实现了分级权限体系作为 HITL 的工程实践：[^3][^5]

| 模式 | 行为 | 适用场景 |
|---|---|---|
| **Default** | 每次文件写入和 Shell 命令都要求确认 | 日常开发 |
| **Plan** | 只读分析模式，不执行任何写操作 | 代码审查、方案设计 |
| **AcceptEdits** | 文件编辑自动通过，Shell 命令仍需确认 | 信任度较高的编码任务 |
| **Auto Mode** | 用分类器自动判断：常规操作放行，高危操作拦截 | 长时间运行的任务 |
| **BypassPermissions** | 跳过所有检查（仅限沙盒环境） | CI/CD、隔离测试环境 |

> Auto Mode 的核心设计：路由每个工具调用通过自动化分类器，常规操作即时执行，但"批量文件删除"或"数据外泄"等高危行为会被拦截并触发人工确认。[^5]

### 7.2 应设置 Confirmation Gates 的操作

| 操作类别 | 具体示例 |
|---|---|
| **数据变更** | 数据库写入/删除、文件删除 |
| **外部调用** | 发送 HTTP 请求到生产 API、触发 Webhook |
| **系统命令** | `rm`、`chmod`、`kill`、部署脚本 |
| **通信** | 发送邮件/消息、发布内容 |
| **配置变更** | 修改环境变量、权限设置、网络规则 |
| **成本操作** | 创建云资源（会产生费用）、调用付费 API |

### 7.3 设计原则

来自 ATA 文章的实践——Cline Agent 在使用 MCP 工具时的授权设计：[^6]

1. **首次使用某工具时必须授权**：Agent 第一次要调用某个 MCP 工具时暂停，显示工具名称和参数，等用户确认
2. **可配置"始终允许"**：对于低风险工具，用户可以勾选"自动授权"
3. **参数透明**：确认时展示完整的工具名称和调用参数，让人类知道 Agent 要做什么
4. **日志审计**：所有操作（无论是否经过人工确认）都记录在日志中

## 八、HITL 的演进方向

| 阶段 | 模式 | 说明 |
|---|---|---|
| **当前** | Human-in-the-Loop | 人在每个关键节点审批 |
| **近期** | Human-on-the-Loop | 人监督全局，只在异常时介入 |
| **远期** | Human-above-the-Loop | 人设定策略和规则，Agent 按策略自治 |

无论演进到哪个阶段，**不可逆操作始终需要显式人类确认**——这是 AI 安全的底线原则。[^2]

## 常见误解


- ❌ "HITL 意味着人类必须参与每一步" —— 只是关键决策点，不是所有步骤

- ❌ "所有 Shell 命令都必须 HITL" —— `ls`、`cat` 等只读命令不需要

- ❌ "只要加了日志就可以自动执行" —— 日志不能替代事前确认

- ❌ "修改代码是不可逆的" —— 在 Git 管理下，代码修改是可逆的

- ❌ "Plan-Build-Verify 是线性的一次性流程" —— 它是可循环迭代的

- ❌ "所有 MCP 工具调用都需要 Confirmation Gate" —— 读取类工具不需要


## 参考资料

<a id="ref1"></a>**[1]** ByteBridge. *From Human-in-the-Loop to Human-on-the-Loop: Evolving AI Agent Autonomy*. Medium, 2025. https://bytebridge.medium.com/from-human-in-the-loop-to-human-on-the-loop-evolving-ai-agent-autonomy-c0ae62c3bf91

<a id="ref2"></a>**[2]** Agno. *How to add human-in-the-loop controls to AI agents that actually run in production*. 2025. https://www.agno.com/blog/how-to-add-human-in-the-loop-controls-to-ai-agents-that-actually-run-in-production

<a id="ref3"></a>**[3]** Sentrely. *Human-in-the-Loop AI: When and How to Gate Claude Agent Actions*. 2025. https://sentrely.com/blog/human-in-the-loop-ai/

<a id="ref4"></a>**[4]** OWASP. *LLM06:2025 Excessive Agency*. https://genai.owasp.org/llmrisk/llm06-excessive-agency/

<a id="ref5"></a>**[5]** Anthropic. *Auto mode for Claude Code*. 2025. https://claude.com/blog/auto-mode

<a id="ref6"></a>**[6]** 钱红凯(趋势). 《AI实践之路：LLM、RAG、MCP、Agent、A2A，掌握AI相关基础原理》第五节 Cline 使用与 MCP 工具授权. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*
