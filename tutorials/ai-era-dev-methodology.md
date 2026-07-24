# AI 时代的开发方法论：SDD 与 TDD

在 AI 编码大行其道的今天，"先写规格再写代码"的 Spec-Driven Development（SDD）和"先写测试再写实现"的 Test-Driven Development（TDD）正在成为保障 AI 生成代码质量的两条核心方法论。SDD 解决"AI 应该做什么"，TDD 解决"怎么验证做对了"。

## 一、Spec-Driven Development（规格驱动开发）

![Spec-Driven Development 工作流](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/sdd-workflow-matched_1784705206.png)

### 1.1 一句话定义

**SDD（Spec-Driven Development，规格驱动开发）** 是一种"先写规范再写代码"的研发范式：在让 AI 动手写任何一行代码之前，先用自然语言 + 结构化文档把需求、约束、验收标准定义清楚。[^1][^2]

> SDD 的核心信念：**Intent is the source of truth**——规格文档是整个项目的单一事实来源。[^3]

### 1.2 为什么需要 SDD：Vibe Coding 的困境

Andrej Karpathy 提出的 **Vibe Coding**（通过自然语言与 AI 对话来写代码）暴露了三个问题：[^1]

| 问题 | 表现 |
|---|---|
| **可验证性缺失** | 不知道"对不对"，只能靠肉眼看 diff |
| **上下文衰减** | 本次对话中提过的规则，换个 session 就忘了 |
| **工程化缺失** | 千人千面的 Prompt 无法沉淀为团队通用规则 |

SDD 的回应：把"一次性口头要求"升级为**持久化、可版本化、可共享的规格文档**。

### 1.3 SDD 核心做法

```
Specify（定义规格）→ Plan（生成计划）→ Task（拆解任务）→ Implement（逐项实现）
```

**Specify**：用自然语言写出用户故事、验收标准、技术约束，产出 `spec.md`。

**Plan**：将 spec 交给 AI 生成技术方案，人工 Review 确认后才进入下一步。

**Task**：把大计划分解为多个独立、可测试的小任务，每个有明确的 Done Definition。

**Implement**：AI 按任务逐个实现代码，每完成一个任务，人工检查是否符合 spec。

### 1.4 Spec 文档结构

```markdown
# Feature: [功能名称]

## 1. 概述
- 一句话说清楚功能做什么

## 2. 用户故事
- As a [角色], I want to [动作], so that [目的]

## 3. 验收标准
- Given [前置条件], When [操作], Then [期望结果]

## 4. 技术约束
- 性能要求、安全要求、兼容性要求

## 5. 接口定义
- 输入格式、输出格式、错误码

## 6. 范围边界（Out of Scope）
- 明确"本次不做"的内容，防止 AI 越界发挥
```

### 1.5 SDD vs Vibe Coding

| 维度 | Vibe Coding | SDD |
|---|---|---|
| **起点** | 一句模糊 Prompt | 结构化规格文档 |
| **验证方式** | 人肉看输出 | 规格 → 自动化测试 → CI |
| **上下文** | 仅当前 session | 文件级持久化 |
| **适用场景** | 临时脚本、Demo | 功能开发、团队协作 |

> **选择信号**：需求一句话能说清楚、临时脚本、明确的 BugFix → Vibe Coding；其他场景 → SDD。

## 二、TDD 在 AI 时代的复兴

### 2.1 一句话定义

**TDD（Test-Driven Development，测试驱动开发）** 是一种"先写测试、再写实现"的开发方法论。在 AI 编码时代，TDD 被重新重视，因为预先编写的测试用例为 AI 生成的代码提供了**客观的、可自动化的验证基准**。[^4][^5]

### 2.2 TDD 经典流程：Red-Green-Refactor

```
Red（写一个失败的测试）→ Green（写最少的代码让测试通过）→ Refactor（重构优化）
```

| 阶段 | 动作 | 目的 |
|---|---|---|
| **Red** | 编写描述期望行为的测试，运行→失败 | 明确"什么是正确的" |
| **Green** | 编写最小化实现，让测试通过 | 让代码满足需求 |
| **Refactor** | 在测试保护下优化代码结构 | 改善设计 |

AI 时代的新意义：**Red 阶段由人类完成（定义期望），Green 阶段可以交给 AI（生成实现）**。

### 2.3 为什么 TDD 在 AI 时代复兴

| 原因 | 说明 |
|---|---|
| **AI 生成代码需要验证器** | LLM 生成的代码"看起来对，不一定真的对"，测试是唯一自动化、客观的验证机制 |
| **测试是"需求的可执行规格"** | 给 AI 一个失败的测试，目标极其明确："让这个测试通过" |
| **形成自动修复闭环** | 人类写测试 → AI 生成实现 → 测试失败 → AI 修正 → 通过 |
| **DORA 报告实证** | 采用 TDD 的团队在使用 AI 辅助编码时，代码质量显著更高 |

### 2.4 AI 时代 TDD 的两种模式

**模式 A：人写测试，AI 写实现（推荐）**

```
人类：编写测试用例（定义"什么是对的"）
  ↓
AI：生成通过测试的代码
  ↓
CI：运行测试，验证正确性
  ↓
人类：Code Review + 重构指导
```

**模式 B：规格 → 测试 → 实现（SDD + TDD 组合）**

```
人类：编写 Spec 文档（需求规格）
  ↓
AI：从 Spec 生成测试套件
  ↓
人类：审核测试
  ↓
AI：生成实现代码
  ↓
CI：运行测试 → 全部通过 → 完成
```

这是当前最成熟的 AI 辅助开发范式：**SDD 定义"做什么"，TDD 定义"怎么验证对了"，AI 负责"怎么做"**。

### 2.5 需要警惕的反模式

| 反模式 | 问题 | 正确做法 |
|---|---|---|
| **AI 同时写测试和实现** | "自己出题自己答"，测试与实现逻辑耦合 | 至少有一方由人类主导 |
| **测试粒度过细** | 限制 AI 选择更优方案的自由度 | 验证**行为和结果**，而非实现细节 |
| **只有 Happy Path** | 异常场景完全没有保护 | 边界条件和异常路径的测试优先级应高于正常路径 |

### 2.6 测试边界条件 Checklist

AI 最容易遗漏边界场景：

- [ ] 空值 / null / undefined 输入
- [ ] 空集合 / 空字符串
- [ ] 极大值和极小值
- [ ] 并发场景
- [ ] 异常路径：网络超时、数据库连接失败
- [ ] 类型边界：整数溢出、浮点精度
- [ ] 权限边界：未授权用户、Token 过期

## 三、SDD + TDD 的组合价值

![SDD 与 TDD 的组合](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/sdd-tdd-combo_1784689207.png)

| 传统价值 | AI 时代新增价值 |
|---|---|
| SDD：消除歧义 | **为 AI 提供明确的目标边界，减少自由发挥** |
| TDD：保证代码正确性 | **为 AI 生成的代码提供自动化验证** |
| SDD：持久化上下文 | **解决跨 session 上下文衰减问题** |
| TDD：支持重构 | **让 AI 大胆重构而不破坏功能** |

## 常见误解

- ❌ "SDD 就是写更好的 Prompt"——SDD 是完整的研发流程

- ❌ "规格先行是因为 AI 不够聪明"——即使模型再强，明确规格也能减少试错

- ❌ "因为 AI 不会写测试"——AI 可以写测试，但不应该同时写测试和实现

## 参考资料

<a id="ref1"></a>**[1]** 蹇志坤(义坤). 《AI 核心概念大串联》第 12 节 Vibe Coding / Spec Coding. ATA, 2026-05-11. https://ata.atatech.org/articles/11020625318 *（内网）*

<a id="ref2"></a>**[2]** GitHub Blog. *Spec-driven development with AI*. 2025. https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/

<a id="ref3"></a>**[3]** Towards Data Science. *From Vibe Coding to Spec-Driven Development*. 2025. https://towardsdatascience.com/from-vibe-coding-to-spec-driven-development/

<a id="ref4"></a>**[4]** Google Cloud / DORA. *TDD and AI: Quality in the DORA Report*. 2025. https://cloud.google.com/discover/how-test-driven-development-amplifies-ai-success

<a id="ref5"></a>**[5]** ACM Digital Library. *Test-Driven Development and LLM-based Code Generation*. FSE 2024. https://dl.acm.org/doi/10.1145/3691620.3695527
