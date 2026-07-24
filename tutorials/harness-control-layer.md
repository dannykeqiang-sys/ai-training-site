# Harness 控制层：前馈与护栏

Harness Engineering 借鉴了控制论中的两种控制机制来管理 Agent 行为：前馈控制（行动前引导）和反馈控制（行动后纠正）。Guardrails（护栏）是这两类控制在安全层面的具体实现——在 LLM 的输入端和输出端设置确定性防线。

## 一、前馈控制与反馈控制

![前馈控制与反馈控制](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/feedforward-feedback_1784689268.png)

### 1.1 控制论视角下的 Agent Harness

| 控制类型 | 别名 | 时机 | 目的 |
|---|---|---|---|
| **Feedforward Control** | Guides（引导） | 行动**之前** | **预防错误**，提升首次输出准确率 |
| **Feedback Control** | Sensors（传感器） | 行动**之后** | **纠正错误**，提供修正信号 |

两者协同形成闭环控制系统：Agent 既能"看准方向出发"，又能"走错了及时修正"。[1](#ref1) [2](#ref2)

### 1.2 Feedforward Controls

在 Agent 执行动作**之前**施加的引导性指令和约束：[2](#ref2) [3](#ref3)

| 手段 | 说明 | 示例 |
|---|---|---|
| **System Prompt** | 角色定义 + 行为规则 | "你是一个代码审查 Agent" |
| **Planning Files** | 预先制定的任务计划 | `plan.md` 定义实现步骤 |
| **Coding Standards** | 注入的编码规范 | `CLAUDE.md` 中的项目规则 |
| **Few-shot Examples** | 期望行为的示例 | Prompt 中提供正确的输入-输出对 |
| **Schema Definitions** | 输出格式约束 | JSON Schema 强制结构化输出 |

设计原则：具体优于模糊、结构化优于散文、分层加载、可版本化。

### 1.3 Feedback Controls

在 Agent 执行动作**之后**提供的检测和修正机制：[2](#ref2) [3](#ref3)

| 手段 | 说明 | 示例 |
|---|---|---|
| **Test Results** | 自动化测试的通过/失败 | `pytest` 结果喂回 Agent |
| **Linter / Type Checker** | 静态分析报错 | ESLint、TypeScript 类型错误 |
| **LLM-as-a-Judge** | 用另一个 LLM 评审输出 | AI Code Reviewer 打分 |
| **User Feedback** | 人工确认/拒绝 | Confirmation Gate |
| **Runtime Errors** | 执行时错误信息 | Python traceback |

### 好的 Feedback 信号 vs 差的 Feedback 信号

**差的**（原始 dump）：`FAILED tests/test_auth.py - AssertionError... (500行完整日志)`

**好的**（结构化、可操作）：
```
测试结果：2 失败 / 45 通过
1. test_login_success (tests/test_auth.py:42)
   错误：expected status 200, got 401
   可能原因：login() 返回值未正确设置 token
建议：检查 src/auth/service.py 中的 login() 方法
```

### 1.4 协同模式

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│ Feedforward │     │   Agent     │     │  Feedback    │
│  (Guides)   │────▶│  Action     │────▶│  (Sensors)   │
└─────────────┘     └─────────────┘     └──────┬───────┘
      ▲                                        │
      │              修正信号                    │
      └─────────────────────────────────────────┘
```

### 1.5 权衡与反模式

前馈和反馈的平衡需要根据场景权衡：

| 维度 | 偏重 Feedforward | 偏重 Feedback |
|---|---|---|
| **优点** | 首次输出质量高，减少重试 | 容错性强，能处理意外情况 |
| **缺点** | 过多规则导致信息过载 | 重试循环消耗 Token，延迟增加 |
| **适用** | 任务模式明确、可预测 | 任务开放性强、探索性强 |

**最佳实践**：先用 Feedforward 覆盖 80% 已知场景，再用 Feedback 兜底剩余 20%。

**常见反模式**：

| 反模式 | 问题 | 正确做法 |
|---|---|---|
| 全靠 Prompt 不设 Feedback | 无法检测和纠正错误 | 即使 Prompt 很好，也要设基本验证（如格式检查） |
| 反馈信号不可操作 | Agent 不知道如何修复 | 提取关键错误，格式化为"问题+位置+建议" |
| 无限重试循环 | Token 消耗失控 | 设最大重试次数（如 3 次）+ 注入额外指导 |
| Feedforward 信息过载 | 模型注意力分散 | 分层加载，只注入当前任务相关的指导 |

**实际案例**：

假设你构建一个代码审查 Agent：

**Feedforward 层**：
- System Prompt：定义审查标准和输出格式
- Few-shot Examples：提供 2-3 个审查示例
- Planning File：预定义的审查清单（安全性 → 性能 → 可维护性）

**Feedback 层**：
- Linter 检查：自动运行 ESLint/TypeScript 检查
- 测试结果：运行单元测试，将失败信息反馈给 Agent
- LLM-as-Judge：用另一个 LLM 评审输出的完整性和准确性

**协同效果**：Feedforward 确保 Agent 知道"如何审查"，Feedback 确保审查结果"确实正确"。

## 二、护栏（Guardrails）——前馈与反馈在安全层面的实现

![Harness 控制层在架构中的位置](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/control-layer-position_1784689270.png)

护栏是 Harness 控制机制在安全领域的具体应用。从控制论视角看：

| 护栏类型 | 对应的控制机制 | 作用时机 | 典型功能 |
|---|---|---|---|
| **输入侧护栏** | Feedforward Control | Agent 行动**之前** | 话题控制、越狱检测、PII 脱敏、有害内容过滤 |
| **输出侧护栏** | Feedback Control | Agent 行动**之后** | 幻觉检测、内容安全、格式验证、相关性检查 |

护栏与 System Prompt 的关键区别在于**确定性**：Prompt 是"建议"（模型可能不遵守），护栏是"强制执行"（代码级拦截）。

**生产系统的多层防御**：模型对齐打底 → Prompt 约束引导 → 护栏强制拦截 → 人工兜底。

### 2.1 主流护栏框架

| 框架 | 开发者 | 特点 |
|---|---|---|
| **NeMo Guardrails** | NVIDIA（开源） | 基于 Colang DSL，支持对话流程控制和安全规则 |
| **Guardrails AI** | 社区（开源） | 基于 Pydantic 验证器，轻量灵活 |
| **Amazon Bedrock Guardrails** | AWS | 托管服务，与 Bedrock 深度集成 |

### 2.2 护栏的核心实现模式

护栏本质上是一个**确定性中间件**，拦截在 LLM 的输入/输出路径上：

```
用户输入 → [输入侧护栏] → LLM → [输出侧护栏] → 返回用户
                  ↑                    ↑
           话题分类/PII检测        幻觉检测/内容安全
           越狱检测/敏感词过滤      格式验证/相关性检查
```

> 关于护栏的完整定义、主流框架的详细对比、实现模式和测试策略，详见教程《护栏（Guardrails）》。

## 三、设计高效的 Harness：实践建议

### 3.1 Harness 的渐进式设计

不要试图一次性构建完美的 Harness。推荐分阶段迭代：

| 阶段 | 重点 | 目标 |
|---|---|---|
| **MVP** | System Prompt + 基本格式验证 | 跑通核心流程 |
| **V1** | 添加 Few-shot + 关键 Feedback（如测试） | 提升首次输出质量 |
| **V2** | 引入护栏 + 结构化错误处理 | 增强安全性和容错性 |
| **V3** | 优化 Token 效率 + 可观测性 | 生产级稳定运行 |

### 3.2 调试 Harness 的常见问题

| 现象 | 可能原因 | 排查方法 |
|---|---|---|
| Agent 频繁进入重试循环 | Feedback 信号不可操作 | 检查错误信息是否包含"问题+位置+建议" |
| 输出质量不稳定 | Feedforward 指令模糊 | 审查 System Prompt 是否足够具体 |
| 护栏误杀正常请求 | 规则过于严格 | 查看护栏日志，调整阈值 |
| Token 消耗过高 | 上下文冗余或重试过多 | 分析每次调用的 Token 分布 |

## 常见误解

- ❌ "好的 Prompt 就够了，不需要 Harness"——Prompt 是非确定性的，无法保证行为边界
- ❌ "护栏就是 System Prompt"——Prompt 可被绕过，护栏是确定性中间件
- ❌ "Feedback 越多越好"——过多 Feedback 导致重试循环，消耗 Token 且增加延迟
- ❌ "前馈控制可以完全消除错误"——前馈只能降低错误概率，无法消除，仍需 Feedback 兜底

## 参考资料

<a id="ref1"></a>**[1]** Anthropic. *Building Effective Agents*. 2024. https://www.anthropic.com/research/building-effective-agents

<a id="ref2"></a>**[2]** LangChain. *The Anatomy of an Agent Harness*. 2025. https://www.langchain.com/blog/the-anatomy-of-an-agent-harness

<a id="ref3"></a>**[3]** Truong Pham. *Harness Engineering*. Dev.to, 2025. https://dev.to/truongpx396/harness-engineering-the-emerging-discipline-of-making-ai-agents-reliable-42gf

<a id="ref4"></a>**[4]** NVIDIA. *NeMo Guardrails*. https://github.com/NVIDIA/NeMo-Guardrails

<a id="ref5"></a>**[5]** AWS. *Amazon Bedrock Guardrails*. https://aws.amazon.com/bedrock/guardrails/
