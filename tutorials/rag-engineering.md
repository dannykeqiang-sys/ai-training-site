# RAG 工程化：选型与流水线

RAG（检索增强生成）是当前最实用的 LLM 增强方案之一。本篇从 RAG 与 Fine-tuning 的选型对比出发，系统讲解生产级 RAG 流水线的各个环节：分块、向量化、检索、重排序与混合搜索。

## 一、RAG 的核心机制

**RAG（Retrieval-Augmented Generation）** 在推理时动态检索与用户查询相关的外部文档，将其作为上下文提供给模型生成回答。[1](#ref1)

```
[预处理] 文档 → 分块（Chunking）→ 向量化（Embedding）→ 存入向量库
[在线]   用户问题 → 向量化 → 检索 Top-K 相关片段 → 拼进 Prompt → LLM 生成答案
```

### 1.1 什么是知识库（Knowledge Base）

**知识库**是 RAG 系统中存储和组织外部知识的集合，是模型"模型外知识"的载体。

**知识库的类型**：

| 类型 | 内容形式 | 典型场景 | 存储方式 |
|---|---|---|---|
| **文档知识库** | PDF、Word、Markdown 等非结构化文档 | 产品手册、技术文档、合同协议 | 向量数据库 |
| **FAQ 知识库** | 问答对（问题 + 标准答案） | 客服系统、内部 Wiki | 向量数据库 / 关系数据库 |
| **结构化知识库** | 表格、数据库记录 | 产品目录、用户信息、订单数据 | 关系数据库 + 自然语言查询 |
| **多模态知识库** | 图片、音频、视频及其描述 | 产品图片库、培训视频 | 多模态向量数据库 |

**知识库的构建要点**：

1. **文档质量**：知识库的质量直接决定 RAG 系统的上限。"垃圾进，垃圾出"——低质量、过时、矛盾的文档会导致检索结果不可靠
2. **更新策略**：
   - **增量更新**：新文档实时或定期加入知识库
   - **全量重建**：定期清空并重建索引（适合文档量小、更新频繁的场景）
   - **版本管理**：保留文档的历史版本，支持回溯查询
3. **权限控制**：企业场景下，不同用户只能检索自己有权限访问的文档（如按部门、项目、密级划分）
4. **元数据标注**：为每个文档/片段添加元数据（来源、作者、更新时间、标签），便于过滤和溯源

**知识库设计的最佳实践**：

- **单一职责**：每个知识库聚焦特定领域（如"产品手册库""HR 政策库"），避免混杂
- **去重与冲突解决**：同一事实出现在多个文档中时，需要去重或明确优先级（如"最新版本优先"）
- **覆盖度检查**：定期评估知识库是否覆盖了用户常见问题，缺失的及时补充
- **反馈闭环**：收集用户对 RAG 回答的反馈（点赞/点踩），低质量回答关联回知识库文档，驱动文档优化

> 💡 **关键认知**：RAG 系统的效果 = 知识库质量 × 检索算法 × 生成模型。三者中，**知识库质量往往是最容易被忽视但影响最大的因素**。

关键点：**模型本身没有变**。新知识只是临时被塞到 Prompt 里。

## 二、RAG vs Fine-tuning：什么时候用哪个

RAG 和 Fine-tuning 是两种不同的知识注入方式：

- **RAG**：把知识放在"模型外面"（向量库），更新快、成本低、可追溯，适合**频繁变动的知识**
- **Fine-tuning**：把知识"刻进模型里"（更新权重），适合**稳定的风格/格式/行为模式**

**组合使用**：先用 Fine-tuning（如 LoRA）让模型学会领域风格，再用 RAG 注入最新知识——是目前最推荐的方案。

> 关于 RAG vs Fine-tuning 的详细对比（知识更新维度、成本结构、选型决策框架），参见教程《Fine-tuning、SFT、PEFT 与 LoRA》。

## 三、RAG 流水线的完整步骤

![RAG 完整流水线：离线索引 + 在线查询](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/rag-pipeline_1784689004.png)

### 3.1 离线索引流水线

```
原始文档 → 文档解析 → 文本分块(Chunking) → 向量化(Embedding) → 写入向量数据库
```

| 步骤 | 说明 |
|---|---|
| **文档解析** | 从 PDF/HTML/Word/Markdown 中提取纯文本 |
| **文本分块** | 将长文本切分为语义连贯的短片段（详见下节） |
| **向量化** | 使用 Embedding 模型将每个 chunk 映射为高维向量 |
| **写入索引** | 将向量 + 原文 + 元数据存入向量数据库 |

### 3.2 在线查询流水线

```
用户问题 → Query改写 → 检索(Retrieval) → 重排序(Reranking) → Prompt组装 → LLM生成
```

**各环节详解**：

| 环节 | 目的 | 常用技术 |
|---|---|---|
| **Query 改写** | 提升检索准确率 | HyDE（用 LLM 生成假设性答案）、Query Expansion（扩展关键词） |
| **检索** | 从向量库召回候选片段 | 向量检索、BM25、混合检索 |
| **重排序** | 对候选片段精细排序 | Cross-Encoder Reranker（如 Cohere Rerank、bge-reranker） |
| **Prompt 组装** | 将检索结果注入 Prompt | 按相关性排序、添加来源标注、控制 Token 数量 |
| **LLM 生成** | 基于上下文生成答案 | 引用来源、拒绝回答（无相关信息时） |

> 💡 **Query 改写的价值**：用户原始问题往往不够精确。例如用户问"怎么部署"，改写后可能变成"Kubernetes 部署应用的最佳实践步骤"，检索准确率显著提升。

## 四、Text Chunking（文本分块）

![文本分块与 Overlap 示意](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/chunking-strategy_1784688984.png)

### 4.1 为什么要分块

1. **适配模型限制**：Embedding 模型通常有输入 Token 上限
2. **提升检索精度**：粒度越细，语义越聚焦
3. **控制成本**：精准的小片段比整篇文档 Token 消耗低

### 4.2 主流分块策略

| 策略 | 原理 | 适用场景 |
|---|---|---|
| **Fixed-size** | 按固定 Token 数切分，带重叠 | 最通用的基线方法 |
| **Recursive** | 按层级分隔符递归切分 | LangChain 默认策略 |
| **Semantic** | 在语义相似度骤降处切分 | 高精度语义边界场景 |
| **Document-based** | 利用标题/章节等结构信息 | Markdown、HTML、PDF |
| **LLM-based** | 让 LLM 判断切分点 | 高质量要求、成本不敏感 |

### 4.3 关键参数

| 参数 | 说明 | 经验值 |
|---|---|---|
| **chunk_size** | 每个块的最大 Token 数 | 256~1024 |
| **chunk_overlap** | 相邻块的重叠部分 | chunk_size 的 10%~20% |

> Overlap 的作用是避免关键信息恰好被切断在两个 chunk 边界上。

## 五、检索策略

### 5.1 向量检索（Dense Retrieval）

利用 Embedding 模型通过余弦相似度找到语义最接近的文档片段。
- **优势**：能理解语义（"苹果手机"可以匹配到"iPhone"）
- **劣势**：对精确关键词（产品编号、人名）不敏感

### 5.2 关键词检索（BM25）

基于词频统计的传统全文检索。
- **优势**：精确匹配关键词命中率高
- **劣势**：无法处理同义词

### 5.3 Hybrid Search（混合搜索）

**Hybrid Search = 向量检索 + 关键词检索**，取两者并集后加权融合。

| 维度 | 纯向量 | 纯 BM25 | Hybrid |
|---|---|---|---|
| 语义理解 | 强 | 弱 | 强 |
| 精确匹配 | 弱 | 强 | 强 |
| 综合召回率 | 中 | 中 | **高** |

> 为什么 Hybrid Search 成为主流？因为用户的 query 可能既包含精确关键词（如 "JIRA-12345"），又包含模糊语义描述（如 "登录报错"）。

## 六、Reranking（重排序）

### 为什么需要重排序

初次检索使用 Bi-Encoder（双编码器），速度快但精度有上限。Reranking 使用 Cross-Encoder 做联合打分，精度显著更高。

### 两阶段检索架构

```
Stage 1（粗排）: Bi-Encoder / BM25 → 快速召回 Top-100
Stage 2（精排）: Cross-Encoder Reranker → 精细打分 → 输出 Top-5~10
```

| 对比 | Stage 1 | Stage 2 |
|---|---|---|
| 速度 | 毫秒级（百万文档） | 较慢（几十~几百条） |
| 精度 | 中等 | **高** |
| 作用 | 快速缩小候选集 | **把最相关的排到最前** |

> **一句话**：Reranking 确保送入 LLM 的内容是最相关、最高质量的片段。

## 常见误解

- ❌ "RAG 比 FT 更准确"——准确性取决于检索质量

## 参考资料

<a id="ref1"></a>**[1]** 钱红凯(趋势). 《AI实践之路》第三节 RAG. ATA, 2025-12-07. https://ata.atatech.org/articles/11020520887 *（内网）*

<a id="ref2"></a>**[2]** Monte Carlo Data. *RAG vs. Fine Tuning: Which One Should You Choose?*. https://montecarlo.ai/blog-rag-vs-fine-tuning/

<a id="ref3"></a>**[3]** Weaviate. *Chunking Strategies to Improve LLM RAG Pipeline Performance*. https://weaviate.io/blog/chunking-strategies-for-rag

<a id="ref4"></a>**[4]** Meilisearch. *Understanding hybrid search RAG for better AI answers*. https://www.meilisearch.com/blog/hybrid-search-rag

<a id="ref5"></a>**[5]** Pinecone. *Rerankers and Two-Stage Retrieval*. https://www.pinecone.io/learn/series/rag/rerankers/
