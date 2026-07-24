# 可观测性与评估

LLM Observability 是"看清系统在做什么"的能力，Evals 是"知道系统做得好不好"的度量手段。两者共同构成 LLM 应用运维和迭代的核心支撑——没有可观测性就无法发现问题，没有评估就无法改进。

## 一、LLM Observability（可观测性）

![LLM 可观测性三支柱](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/llm-observability-matched-v2_1784705449.png)

### 1.1 定义

**LLM Observability** 是对 LLM 应用在生产环境中的运行状态进行**追踪（Tracing）、监控（Monitoring）和评估（Evaluation）**的系统化能力。[1](#ref1) [2](#ref2)

> 一句话：让你能看清 LLM 应用内部"发生了什么"以及"为什么会这样"。

### 1.2 为什么 LLM 应用特别需要可观测性

| 传统软件 | LLM 应用 |
|---|---|
| 输入/输出确定性 | **非确定性**（相同输入 → 不同输出） |
| 错误有明确堆栈 | 错误可能表现为"质量下降"而非崩溃 |
| 单次调用可独立分析 | 多步 Agent 需要**链路追踪** |

### 1.3 核心能力

| 能力 | 说明 | 价值 |
|---|---|---|
| **Tracing（追踪）** | 记录每次调用的完整链路 | 定位"哪一步出了问题" |
| **Monitoring（监控）** | 实时指标：延迟、Token 消耗、错误率 | 发现性能退化和异常 |
| **Evaluation（评估）** | 对输出质量的自动/人工评分 | 衡量"回答好不好" |
| **Cost Tracking** | 按请求/用户/功能统计费用 | 控制开支 |
| **Prompt Versioning** | 管理 Prompt 变更历史 | 支持 A/B 测试和回滚 |

### 1.4 关键指标

在生产环境中，以下指标帮助你量化 LLM 应用的健康度：

| 指标类别 | 具体指标 | 目标值 | 说明 |
|---|---|---|---|
| **性能** | 首 Token 延迟（TTFT） | < 1s | 用户感知的响应速度 |
| **性能** | 完整响应延迟 | < 5s | 端到端耗时 |
| **成本** | 单次调用平均 Token | < 2000 | 输入 + 输出总和 |
| **成本** | 单用户日均成本 | < $0.5 | 按用户维度的开支 |
| **质量** | 用户满意度（👍/👎） | > 80% | 直接反馈 |
| **质量** | Eval 通过率 | > 90% | 自动评估 |
| **可靠性** | 错误率 | < 1% | 调用失败或异常 |
| **可靠性** | 重试率 | < 5% | 触发重试的请求占比 |

> 💡 **指标陷阱**：单纯追求低延迟可能导致输出质量下降（如过度截断）。建议同时监控质量指标，避免"快但差"的优化方向。

### 1.5 主流平台

| 平台 | 开源 | 核心定位 |
|---|---|---|
| **LangSmith** | ❌ | 深度集成 LangChain 生态 |
| **LangFuse** | ✅ | 最流行的开源 LLMOps 平台 |
| **Braintrust** | ❌ | 强调评估和实验对比 |
| **Arize Phoenix** | ✅ | 向量嵌入可视化 |

> **选型建议**：LangChain 生态用 LangSmith；框架无关的自托管方案用 LangFuse（开源免费，几行代码即可接入）。

## 二、LLMOps（LLM Operations）

### 2.1 定义

**LLMOps** 是管理 LLM 应用全生命周期的工程实践和工具集，涵盖从开发、评估、部署到持续监控和优化的完整链路。[3](#ref3)

> 类比：DevOps 让传统软件持续交付，MLOps 让 ML 模型持续交付，LLMOps 是专门针对 LLM 应用特点适配的"持续交付"方法论。

### 2.2 LLMOps vs MLOps vs DevOps

| 维度 | DevOps | MLOps | LLMOps |
|---|---|---|---|
| **核心产物** | 应用代码 | 训练好的模型 | Prompt + Harness + 模型配置 |
| **版本管理对象** | 源代码 | 数据 + 模型 + 代码 | **Prompt + 工具定义 + 评估集** |
| **测试** | 单测/集测 | 模型指标 | **Evals（自动评估 + LLM-as-Judge）** |
| **监控重点** | 延迟/错误率 | 模型漂移 | **输出质量/成本/安全** |

### 2.3 生命周期

```
Develop → Eval → Deploy → Monitor → Optimize → (循环)
```

**各阶段关键活动**：

| 阶段 | 关键活动 | 产出 |
|---|---|---|
| **Develop** | 设计 Prompt、工具定义、护栏规则 | Harness 配置 |
| **Eval** | 构建评估集、运行 Evals、分析失败案例 | Eval 报告 |
| **Deploy** | 灰度发布、A/B 测试、监控指标 | 生产环境 |
| **Monitor** | 追踪关键指标、检测异常、收集用户反馈 | 告警 + 反馈数据 |
| **Optimize** | 分析失败模式、迭代改进、重新评估 | 改进版本 |

> 💡 **LLMOps 的核心理念**：LLM 应用不是一次性交付，而是持续迭代。每次迭代都需要 Evals 验证效果，Monitor 发现新问题。

## 三、Evals（评估）概述

![两类评估方法：确定性验证与 LLM-as-Judge](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/evals-two-methods_1784689381.png)

Evals（评估）是"知道系统做得好不好"的度量手段，与可观测性共同构成 LLM 应用运维和迭代的核心支撑——没有可观测性就无法发现问题，没有评估就无法改进。

### 3.1 Evals 在可观测性中的位置

在可观测性平台（如 LangFuse、LangSmith）中，Evals 是关键的度量能力：

| 可观测性能力 | Evals 的角色 |
|---|---|
| **Tracing** | 记录调用链路，发现"哪一步出了问题" |
| **Monitoring** | 监控延迟、Token、错误率等运行指标 |
| **Evaluation** | 对输出质量进行自动/人工评分，回答"回答好不好" |
| **Cost Tracking** | 按请求/用户/功能统计费用 |

> 💡 **可观测性提供数据（日志、指标、链路），Evals 提供判断（质量评分、通过率）。两者配合才能形成完整的反馈闭环。**

### 3.2 评估方法简介

评估方法分为两大类：

1. **确定性验证（Deterministic Evals）**：使用规则、代码断言、格式检查验证输出（成本低、速度快，只能验证"形式"）
2. **LLM 验证（LLM-as-a-Judge）**：使用另一个 LLM 评判输出的语义质量（成本高、有延迟，但能评判"语义"）

最佳实践是分层组合：先跑确定性验证过滤明显错误，再用 LLM Judge 评判语义质量，在质量和成本之间取得平衡。

> 📖 **详细讲解**：两类验证方法的具体实现、评估数据集设计、LLM-as-a-Judge 的已知局限与缓解方法，详见 [评估与评估驱动开发](./evals-llm-as-judge-eval-driven-development.md)。

## 四、Eval-Driven Development 概述

Eval-Driven Development（评估驱动开发）是一种以评估为中心的开发方法论：**先写评估，再优化系统**。其核心流程是：观察真实数据 → 识别失败模式 → 编写针对性 Eval → 迭代改进 → 验证改进效果 → 部署并继续观察。

> 📖 **详细讲解**：Eval-Driven Development 的完整流程、评估数据集设计原则、CI/CD 集成、生产环境监控，详见 [评估与评估驱动开发](./evals-llm-as-judge-eval-driven-development.md)。

## 常见误解

- ❌ "只做模型训练"——LLMOps 聚焦应用层

## 参考资料

<a id="ref1"></a>**[1]** Agenta.ai. *Top LLM Observability Platforms 2025*. https://agenta.ai/blog/top-llm-observability-platforms

<a id="ref2"></a>**[2]** LangFuse. *Langfuse is the #1 most used Open Source LLMOps Product*. 2024. https://langfuse.com/blog/2024-11-most-used-oss-llmops

<a id="ref3"></a>**[3]** Kanerika. *LLMOps Observability*. https://kanerika.com/blogs/llmops-observability/

<a id="ref4"></a>**[4]** OpenAI. *Evaluation Best Practices*. https://developers.openai.com/api/docs/guides/evaluation-best-practices

<a id="ref5"></a>**[5]** Hamel Husain. *LLM Evals: Everything You Need to Know*. https://hamel.dev/blog/posts/evals-faq/

<a id="ref6"></a>**[6]** Zheng, L., et al. *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*. NeurIPS 2023. https://arxiv.org/abs/2306.05685
