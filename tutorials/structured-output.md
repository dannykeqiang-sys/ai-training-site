# 结构化输出

Structured Output（结构化输出）是让 LLM 按照预定义的数据结构（如 JSON Schema）生成输出的技术，确保下游程序可以确定性地解析和使用。在生产系统中，LLM 的输出往往不是给人看的，而是给程序消费的——Structured Output 解决的就是"如何让 LLM 的输出像 API 返回值一样可靠"这一问题。

## 一、一句话定义

**Structured Output（结构化输出）** 是指让 LLM 按照预定义的数据结构（如 JSON Schema）生成输出的技术，确保输出格式**可预测、可解析、可验证**，而不是自由格式的自然语言文本。[¹](#ref1) [²](#ref2)

> 比喻：如果自由文本是"手写便条"，Structured Output 就是"填好的表单"——格式固定，字段明确，程序可以直接读取。

## 二、为什么 Structured Output 在生产中很重要

![自由文本 vs 结构化输出](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/structured-output-flow_1784689360.png)

### 2.1 核心问题

LLM 天生是"自由文本生成器"，它输出的是一串自然语言。但在生产系统中，LLM 的输出往往需要被**下游程序**消费：[¹](#ref1) [²](#ref2)

```
用户提问 → LLM 生成 → ??? → 下游系统处理
                         ↑
                   这里需要确定格式
```

如果 LLM 的输出格式不确定，下游系统就无法可靠地解析它。

### 2.2 没有 Structured Output 的痛点

| 问题 | 表现 | 后果 |
|---|---|---|
| **解析失败** | LLM 有时返回合法 JSON，有时夹杂解释文字 | `json.loads()` 随机报错 |
| **字段缺失** | 有时缺少必要字段 | 下游空指针异常 |
| **类型错误** | 数字有时返回为字符串 | 类型不匹配 |
| **格式不一致** | 日期有时 "2024-01-01" 有时 "Jan 1, 2024" | 无法统一处理 |
| **额外字段** | 模型自行添加未预期的字段 | 数据污染 |
| **重试成本** | 格式错误 → 重试 → 多花 Token | 成本增加 30-50% |

### 2.3 Structured Output 解决了什么

| 能力 | 说明 |
|---|---|
| **100% 格式合规** | 输出严格遵循预定义 Schema，不会出现格式错误 |
| **零解析失败** | 下游代码可以确定性地解析每一次输出 |
| **类型安全** | 字段类型有保证（string/number/boolean/array） |
| **可组合性** | 结构化的输出可以直接喂给下一个系统/工具/Agent |
| **降低成本** | 消除格式错误导致的重试 |

## 三、技术实现方式

![结构化输出三种实现方式的可靠性对比](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/structured-output-methods-matched_1784705087.png)

主流 LLM API 都提供了原生的结构化输出支持，确保模型输出 100% 符合预定义的 Schema。实现方式包括：

- **约束解码**（如 OpenAI Structured Outputs 的 `strict: true` 模式）：在模型推理层强制约束，每个 token 生成步骤只允许合法 token
- **Function Calling / Tool Use**：通过定义 Tool 的参数 Schema 间接实现结构化输出，这是 Agent 系统中最常见的形式
- **Prompt + Post-processing**：传统方式，通过 Prompt 要求 JSON 格式 + 后处理校验，但不能 100% 保证格式正确

### 3.1 各方案对比

| 方案 | 格式保证 | 实现层 | 额外成本 | 适用场景 |
|---|---|---|---|---|
| **OpenAI Structured Outputs** | ✅ 100% | 模型推理层（约束解码） | 极低 | 需要严格 Schema 遵循 |
| **Function Calling** | ✅ 接近 100% | 模型推理层 | 无 | Agent 工具调用 |
| **Prompt 指令 + 后处理** | ❌ 约 85-95% | Prompt + 应用层 | 重试成本 | 无原生 API 支持时 |
| **Guardrails AI Validators** | ✅ (通过修复) | 应用层 | 修复调用 | 需要灵活自定义规则 |

**如何选择？** 如果你使用 OpenAI 模型且需要严格 Schema 遵循，直接用 Structured Outputs——它提供 100% 的格式保证，几乎没有额外成本。如果你在构建 Agent 系统，Function Calling 是自然的选择——工具调用参数天然就是结构化的，且与 Agent 编排无缝衔接。只有当你使用的模型不提供原生结构化输出 API 时，才需要考虑"Prompt 指令 + 后处理"的方式，并务必实现重试逻辑以应对格式错误。

## 四、Structured Output 在 Harness 中的角色

```
┌───────────────────────────────────────────────────┐
│  Agent Harness                                    │
│                                                   │
│  Feedforward:                                     │
│    System Prompt + JSON Schema 定义               │
│         ↓                                         │
│  Model:                                           │
│    约束解码 → 结构化输出                           │
│         ↓                                         │
│  Feedback:                                        │
│    Schema 验证（确定性验证的一种）                  │
│         ↓                                         │
│  下游消费:                                        │
│    工具调用 / API 响应 / UI 渲染 / 数据库写入      │
└───────────────────────────────────────────────────┘
```

Structured Output 同时涉及 Harness 的多个层面：
- **Feedforward Control**：通过 Schema 定义预先约束输出格式
- **Deterministic Middleware**：Schema 校验是确定性验证
- **Tool Integration**：Function Calling 的基础

## 五、典型应用场景

| 场景 | 为什么需要 Structured Output |
|---|---|
| **Agent Tool Calling** | 工具参数必须是确定格式，才能被程序调用 |
| **数据提取 / NER** | 从非结构化文本中提取结构化实体（人名、地址、金额） |
| **分类任务** | 输出必须是预定义类别之一，不能随意编造 |
| **API 响应** | 前端期望固定格式的 JSON 响应 |
| **多步编排** | Agent A 的输出是 Agent B 的输入，格式必须对齐 |
| **数据库写入** | 提取的数据要写入数据库，字段/类型必须匹配 |

### 5.1 场景深度分析：数据提取

数据提取是 Structured Output 最常见的非 Agent 场景。典型流程：原始文本 → LLM + Schema → 结构化输出 → 写入下游系统。

**Schema 设计要点**（以从客户邮件提取信息为例）：
- 使用 `enum` 约束分类字段（如 issue_category: billing/technical/shipping/other），避免模型自由发挥
- 每个字段加 `description` 帮助模型理解语义
- `required` 只标记下游必需字段
- 数组类型（如 action_items）便于下游逐项处理

### 5.2 场景深度分析：多 Agent 编排

在多 Agent 系统中，Structured Output 是 Agent 间通信的基础：Agent A 的结构化输出直接作为 Agent B 的输入。

```
Agent A（需求分析师）→ { requirements, priority, effort }
    ↓ 格式必须对齐
Agent B（任务拆解器）→ { tasks, dependencies, assignments }
    ↓ 格式必须对齐
Agent C（排期规划器）
```

如果 Agent A 的输出格式不稳定（有时返回数组，有时返回字符串），整个编排链就会断裂。Structured Output 确保每一环的输出格式可预测。


## 六、Schema 设计常见陷阱

| 陷阱 | 问题 | 建议 |
|---|---|---|
| **过度约束** | 所有字段都标记 required，模型被迫编造内容填充 | 只标记下游必需字段 |
| **枚举值设计不当** | 选项过少无法覆盖场景，过多则模型难以选择 | 控制 3-8 个，始终提供 "other" 兜底 |
| **嵌套过深** | 超过 3 层嵌套降低模型理解准确度 | 扁平化设计，控制在 2-3 层 |
| **缺少 description** | 字段名模糊，模型无法理解语义 | 每个字段加 description，尤其是含义不直观的字段 |

**关键原则**：Schema 是模型理解字段语义的关键线索，设计时应站在模型角度思考"这些信息够不够我正确填充？"


## 七、跨平台结构化输出对比

主流 LLM 平台对结构化输出的支持程度不同：

| 平台 | 原生支持 | 实现方式 | 格式保证 |
|---|---|---|---|
| **OpenAI** | ✅ Structured Outputs | 约束解码（`strict: true`） | 100% Schema 遵循 |
| **Anthropic (Claude)** | ✅ Tool Use | 模型层工具调用 | 接近 100% |
| **Google (Gemini)** | ✅ Response Schema | 模型层约束 | 接近 100% |
| **开源模型（Llama/Qwen）** | ⚠️ 部分支持 | 依赖框架（vLLM、Outlines） | 取决于实现 |

**平台选型考量**：
- 如果需要 100% 格式保证且使用 OpenAI，直接用 Structured Outputs
- 如果构建 Agent 系统，Function Calling / Tool Use 是跨平台最通用的方式
- 如果使用开源模型，需要额外引入约束解码框架（如 Outlines、Guidance）


## 八、与 Evals 的关系

Structured Output 为 Evals 提供了基础：[²](#ref2)

- **格式可验证**：Schema 验证本身就是最基本的确定性 Eval
- **字段可断言**：可以对特定字段进行精确匹配或范围检查
- **可批量处理**：结构化数据易于批量分析和统计

```python
# 有了 Structured Output，确定性 Eval 变得很简单
assert output["temperature"] > -50  # 合理范围检查
assert output["unit"] in ["celsius", "fahrenheit"]  # 枚举验证
assert len(output["description"]) < 200  # 长度约束
```

## 常见误解

- ❌ "Structured Output 只是在 Prompt 里要求返回 JSON"——真正的 Structured Output 是在模型推理层通过约束解码保证的
- ❌ "Structured Output 会降低回答质量"——合理设计的 Schema 不影响内容质量，只约束格式
- ❌ "只有 JSON 才是 Structured Output"——XML、YAML、甚至严格的 Markdown 表格也算

## 参考资料

<a id="ref1"></a>**[1]** OpenAI. *Introducing Structured Outputs in the API*. 2024-08. https://openai.com/index/introducing-structured-outputs-in-the-api/

<a id="ref2"></a>**[2]** OpenAI. *Structured Model Outputs — API Documentation*. https://developers.openai.com/api/docs/guides/structured-outputs

<a id="ref3"></a>**[3]** Anthropic. *Tool Use (Function Calling) — Documentation*. https://docs.anthropic.com/en/docs/build-with-claude/tool-use

<a id="ref4"></a>**[4]** Humanloop. *Structured Outputs: Everything You Should Know*. https://humanloop.com/blog/structured-outputs

<a id="ref5"></a>**[5]** 蹇志坤(义坤). 《AI 核心概念大串联：从 LLM 到 Agent 的完整认知地图》. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

