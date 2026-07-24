# 评估与评估驱动开发

评估（Evals）是 LLM 应用开发中最重要但最容易被忽视的环节。没有评估，你就无法知道系统是否在变好，也无法进行有效的迭代优化。本课介绍评估的核心概念、两类验证方法、以及评估驱动开发（Eval-Driven Development）的实践方法。

## 什么是 Evals？

**Evals（评估）** 是系统性地衡量 LLM 应用输出质量的过程。它回答一个核心问题：**你怎么知道每次改动让系统变好了还是变坏了？**

### 为什么 Evals 是"最重要的事情"

OpenAI 和 Anthropic 都强调 Evals 在 LLM 应用开发中的核心地位：

| 原因 | 说明 |
|---|---|
| **迭代的基础** | 无法度量就无法优化 |
| **信心的来源** | 知道改动是变好还是变坏 |
| **回归检测** | 改 A 不会破坏 B |
| **团队协作** | 统一对"好"的定义 |
| **成本效益** | 投入 Evals 的 ROI 通常最高 |

> "Error analysis is the most important activity in evals" — Hamel Husain

### 典型失败场景

没有 Evals 的开发流程容易出现：

- **盲目优化**：优化了场景 A 的 Prompt，结果场景 B 准确率从 90% 降到 60%，两周后才发现
- **无法复现**：昨天效果很好，今天突然变差，不知道原因
- **团队分歧**：开发说"效果不错"，产品说"还不够好"，没有客观标准
- **回归问题**：修复了一个 bug，引入了三个新 bug

## 两类验证方法

![两类验证方法：确定性验证与 LLM-as-Judge](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/evals-two-methods-detailed-matched_1784705237.png)

评估方法分为两大类：**确定性验证**和 **LLM 验证**。

### 第一类：确定性验证（Deterministic Evals）

使用规则、代码断言、格式检查等确定性方法进行评估。

**特点**：
- ✅ 成本低、速度快、100% 可重复
- ❌ 只能验证"形式"，无法评判"语义质量"

**常用方法**：

#### 1. 格式验证
```python
def check_json_format(output):
    """检查输出是否为合法 JSON"""
    try:
        data = json.loads(output)
        return True, "格式正确"
    except json.JSONDecodeError as e:
        return False, f"JSON 解析失败: {e}"

def check_required_fields(output, required_fields):
    """检查 JSON 是否包含必需字段"""
    data = json.loads(output)
    missing = [f for f in required_fields if f not in data]
    if missing:
        return False, f"缺少字段: {missing}"
    return True, "字段完整"
```

#### 2. 规则断言
```python
def check_length(output, min_len=10, max_len=1000):
    """检查输出长度是否在合理范围"""
    if len(output) < min_len:
        return False, f"输出过短: {len(output)} < {min_len}"
    if len(output) > max_len:
        return False, f"输出过长: {len(output)} > {max_len}"
    return True, "长度合理"

def check_no_hallucination(output, source_docs):
    """检查输出是否包含源文档中没有的信息"""
    # 简化的幻觉检测：检查关键实体是否在源文档中出现
    entities = extract_entities(output)
    for entity in entities:
        if entity not in source_docs:
            return False, f"可能的幻觉: {entity}"
    return True, "无明显幻觉"
```

#### 3. 关键词匹配
```python
def check_keywords(output, required_keywords):
    """检查输出是否包含必需关键词"""
    missing = [kw for kw in required_keywords if kw not in output]
    if missing:
        return False, f"缺少关键词: {missing}"
    return True, "关键词完整"

def check_forbidden_words(output, forbidden_words):
    """检查输出是否包含禁用词"""
    found = [word for word in forbidden_words if word in output]
    if found:
        return False, f"包含禁用词: {found}"
    return True, "无禁用词"
```

### 第二类：LLM 验证（LLM-as-a-Judge）

使用另一个 LLM 评判输出的语义质量。

**特点**：
- ✅ 能评判开放式、主观性内容
- ❌ 成本高、有延迟、自身也有偏见

**常用方法**：

#### 1. 单点评分
```python
JUDGE_PROMPT = """
你是一个严格的评估专家。请评估以下回答的质量。

**问题**: {question}
**参考回答**: {reference}
**待评估回答**: {answer}

请从以下维度评分（1-5分）：
1. **准确性**: 回答是否正确、无幻觉
2. **完整性**: 是否覆盖了问题的所有方面
3. **相关性**: 是否紧扣问题，没有跑题
4. **清晰度**: 表达是否清晰、易于理解

输出 JSON 格式：
{
  "accuracy": <1-5>,
  "completeness": <1-5>,
  "relevance": <1-5>,
  "clarity": <1-5>,
  "overall": <1-5>,
  "reasoning": "<评分理由>"
}
"""

def llm_judge(question, reference, answer, judge_model="gpt-4"):
    """使用 LLM 评判回答质量"""
    prompt = JUDGE_PROMPT.format(
        question=question,
        reference=reference,
        answer=answer
    )
    response = call_llm(prompt, model=judge_model)
    scores = json.loads(response)
    return scores
```

#### 2. 对比评估
```python
COMPARISON_PROMPT = """
你是一个评估专家。请比较两个回答，判断哪个更好。

**问题**: {question}
**回答 A**: {answer_a}
**回答 B**: {answer_b}

请判断：
- A 更好
- B 更好
- 两者相当
- 无法判断

输出 JSON：
{
  "winner": "A" | "B" | "tie" | "unknown",
  "reasoning": "<判断理由>"
}
"""

def compare_answers(question, answer_a, answer_b, judge_model="gpt-4"):
    """对比两个回答的质量"""
    prompt = COMPARISON_PROMPT.format(
        question=question,
        answer_a=answer_a,
        answer_b=answer_b
    )
    response = call_llm(prompt, model=judge_model)
    result = json.loads(response)
    return result
```

#### 3. 分类评估
```python
CLASSIFICATION_PROMPT = """
请判断以下回答是否满足要求。

**问题**: {question}
**要求**: {criteria}
**回答**: {answer}

判断：
- PASS: 满足要求
- FAIL: 不满足要求

输出 JSON：
{
  "result": "PASS" | "FAIL",
  "reasoning": "<判断理由>"
}
"""

def classify_answer(question, criteria, answer, judge_model="gpt-4"):
    """判断回答是否满足特定要求"""
    prompt = CLASSIFICATION_PROMPT.format(
        question=question,
        criteria=criteria,
        answer=answer
    )
    response = call_llm(prompt, model=judge_model)
    result = json.loads(response)
    return result
```

### 最佳实践：分层组合

```
第 1 层：确定性验证（成本最低，先跑）
  └─ 格式检查、规则断言、关键词匹配

第 2 层：规则断言（中等成本）
  └─ 业务逻辑验证、数据一致性检查

第 3 层：LLM-as-a-Judge（成本最高，最后跑）
  └─ 语义质量评估、开放式问题评判
```

**为什么分层？**
- 确定性验证可以快速过滤明显错误，减少 LLM Judge 的调用次数
- LLM Judge 成本高、有延迟，应该只用于需要语义判断的场景
- 分层可以在保证质量的同时控制成本

## 评估数据集

评估数据集是 Evals 的基础。一个好的评估数据集应该：

### 数据集设计原则

| 原则 | 说明 |
|---|---|
| **代表性** | 覆盖真实场景的各种情况 |
| **多样性** | 包含简单、中等、困难不同难度的样本 |
| **边界情况** | 包含异常输入、极端情况 |
| **可验证性** | 每个样本都有明确的评估标准 |

### 数据集来源

1. **真实用户数据**：从生产日志中提取真实查询（需脱敏）
2. **人工构造**：专家根据业务场景设计测试用例
3. **边界测试**：针对已知弱点设计的对抗性样本
4. **回归测试**：从历史 bug 中提取的测试用例

### 数据集规模

| 场景 | 建议规模 | 说明 |
|---|---|---|
| **冒烟测试** | 10-50 条 | 快速验证基本功能 |
| **日常评估** | 100-500 条 | 日常开发迭代使用 |
| **全面评估** | 1000+ 条 | 版本发布前的完整测试 |

## 评估驱动开发（Eval-Driven Development）

![评估驱动开发闭环](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/eval-driven-loop-matched_1784705266.png)

评估驱动开发是一种以评估为中心的开发方法论：**先写评估，再优化系统**。

### 核心流程

```
1. 定义评估标准
   └─ 什么算"好"？如何度量？

2. 构建评估数据集
   └─ 收集代表性样本，标注预期结果

3. 实现评估流水线
   └─ 自动化评估脚本，一键运行

4. 基线评估
   └─ 跑一遍评估，记录当前性能

5. 迭代优化
   └─ 改 Prompt / 改 RAG / 改模型
   └─ 跑评估，对比基线
   └─ 如果变好，提交；如果变差，回滚

6. 持续监控
   └─ 定期跑评估，检测性能退化
```

### 实践示例

#### 场景：优化客服机器人的回答质量

**第 1 步：定义评估标准**
```python
EVAL_CRITERIA = {
    "accuracy": "回答是否正确，无幻觉",
    "completeness": "是否覆盖问题的所有方面",
    "politeness": "语气是否礼貌、专业",
    "actionability": "是否提供可操作的建议",
}

# 定义及格线
PASS_THRESHOLD = {
    "accuracy": 4.0,      # 准确性必须 >= 4 分
    "completeness": 3.5,  # 完整性 >= 3.5 分
    "politeness": 4.0,    # 礼貌度 >= 4 分
    "actionability": 3.0, # 可操作性 >= 3 分
}
```

**第 2 步：构建评估数据集**
```python
EVAL_DATASET = [
    {
        "id": 1,
        "question": "如何重置密码？",
        "reference": "请访问 xxx.com/reset，输入邮箱，点击重置链接",
        "category": "账户管理",
        "difficulty": "简单"
    },
    {
        "id": 2,
        "question": "为什么我的订单还没发货？",
        "reference": "订单处理通常需要 1-2 个工作日，您可以在订单页面查看状态",
        "category": "订单查询",
        "difficulty": "中等"
    },
    # ... 更多样本
]
```

**第 3 步：实现评估流水线**
```python
def run_evaluation(system, dataset, judge_model="gpt-4"):
    """运行完整评估"""
    results = []
    
    for sample in dataset:
        # 1. 调用系统获取回答
        answer = system.answer(sample["question"])
        
        # 2. 确定性验证
        deterministic_scores = {
            "length_ok": check_length(answer),
            "no_forbidden": check_forbidden_words(answer, FORBIDDEN_WORDS),
        }
        
        # 3. LLM Judge
        llm_scores = llm_judge(
            question=sample["question"],
            reference=sample["reference"],
            answer=answer,
            judge_model=judge_model
        )
        
        # 4. 记录结果
        results.append({
            "sample_id": sample["id"],
            "question": sample["question"],
            "answer": answer,
            "deterministic": deterministic_scores,
            "llm_scores": llm_scores,
        })
    
    # 5. 计算总体指标
    metrics = compute_metrics(results)
    return metrics, results

def compute_metrics(results):
    """计算总体评估指标"""
    accuracy_scores = [r["llm_scores"]["accuracy"] for r in results]
    completeness_scores = [r["llm_scores"]["completeness"] for r in results]
    
    return {
        "total_samples": len(results),
        "avg_accuracy": sum(accuracy_scores) / len(accuracy_scores),
        "avg_completeness": sum(completeness_scores) / len(completeness_scores),
        "pass_rate": sum(1 for r in results if is_pass(r)) / len(results),
    }
```

**第 4 步：基线评估**
```python
# 记录当前性能作为基线
baseline_metrics = run_evaluation(current_system, EVAL_DATASET)
print(f"基线性能: {baseline_metrics}")
# 输出: 基线性能: {avg_accuracy: 3.8, avg_completeness: 3.5, pass_rate: 0.72}
```

**第 5 步：迭代优化**
```python
# 尝试优化 Prompt
new_prompt = "你是一个专业的客服助手，请用礼貌、清晰的语气回答用户问题..."
new_system = CustomerServiceSystem(prompt=new_prompt)

# 跑评估
new_metrics = run_evaluation(new_system, EVAL_DATASET)
print(f"优化后性能: {new_metrics}")
# 输出: 优化后性能: {avg_accuracy: 4.1, avg_completeness: 3.8, pass_rate: 0.85}

# 对比基线
if new_metrics["pass_rate"] > baseline_metrics["pass_rate"]:
    print("✅ 优化有效，提交更改")
    save_system(new_system)
else:
    print("❌ 优化无效，回滚")
```

### 持续评估与监控

#### 自动化评估流水线

```python
# CI/CD 集成：每次提交自动跑评估
def ci_evaluation():
    """CI/CD 中的自动化评估"""
    # 1. 拉取最新代码
    system = load_latest_system()
    
    # 2. 跑冒烟测试（快速验证）
    smoke_metrics = run_evaluation(system, SMOKE_DATASET)
    if smoke_metrics["pass_rate"] < 0.9:
        raise Exception("冒烟测试失败，阻止部署")
    
    # 3. 跑完整评估
    full_metrics = run_evaluation(system, FULL_DATASET)
    
    # 4. 对比基线
    baseline = load_baseline_metrics()
    if full_metrics["pass_rate"] < baseline["pass_rate"] - 0.05:
        raise Exception("性能退化超过 5%，阻止部署")
    
    # 5. 更新基线
    save_baseline_metrics(full_metrics)
    print("✅ 评估通过，允许部署")
```

#### 生产环境监控

```python
def production_monitoring():
    """生产环境的持续监控"""
    # 1. 定期采样真实查询
    daily_samples = sample_production_queries(date=today(), n=100)
    
    # 2. 跑评估
    metrics = run_evaluation(production_system, daily_samples)
    
    # 3. 检测异常
    if metrics["pass_rate"] < ALERT_THRESHOLD:
        send_alert(f"性能下降: {metrics}")
    
    # 4. 记录趋势
    log_metrics(metrics, date=today())
```

## 常见误区

### 误区 1：只看平均分数

**错误做法**：只看 `avg_accuracy = 4.2`，认为"效果不错"

**问题**：
- 平均数掩盖了分布问题
- 可能 80% 的样本是 5 分，20% 的样本是 1 分
- 那 20% 的失败案例可能是关键场景

**正确做法**：
```python
# 查看分数分布
score_distribution = analyze_distribution(results)
# 输出: {5分: 60%, 4分: 20%, 3分: 10%, 2分: 5%, 1分: 5%}

# 重点关注低分样本
low_score_samples = [r for r in results if r["llm_scores"]["overall"] < 3]
# 分析低分样本的共同特征，针对性优化
```

### 误区 2：评估数据集太小或不够代表性

**错误做法**：只用 10 个样本评估，而且都是简单场景

**问题**：
- 无法发现边界情况和长尾问题
- 评估结果不可靠，方差大
- 上线后遇到真实场景就崩了

**正确做法**：
- 至少 100+ 样本，覆盖各种难度和场景
- 包含真实用户数据（脱敏后）
- 包含历史 bug 和边界情况

### 误区 3：只在开发时评估，上线后不监控

**错误做法**：开发时跑评估，上线后就不管了

**问题**：
- 数据分布会漂移（用户行为变化、新场景出现）
- 模型可能退化（API 更新、服务不稳定）
- 无法及时发现性能下降

**正确做法**：
- 生产环境定期采样评估（如每天 100 条）
- 设置告警阈值（如 pass_rate < 0.8 触发告警）
- 记录趋势，及时发现异常

### 误区 4：LLM Judge 自身也有偏见

**错误做法**：完全信任 LLM Judge 的评分，不做校准

**问题**：
- LLM Judge 可能有位置偏见（倾向于给第一个回答高分）
- 可能有长度偏见（倾向于给长回答高分）
- 可能有自我偏见（倾向于给自己生成的回答高分）

**正确做法**：
```python
# 1. 随机化顺序，消除位置偏见
if random.random() > 0.5:
    answer_a, answer_b = answer_b, answer_a
    swap_flag = True
else:
    swap_flag = False

# 2. 使用多个 Judge 模型，交叉验证
scores_gpt4 = llm_judge(question, reference, answer, judge_model="gpt-4")
scores_claude = llm_judge(question, reference, answer, judge_model="claude-3-opus")

# 3. 人工抽检，校准 Judge
sample_10_percent = random.sample(results, len(results) // 10)
human_scores = human_evaluation(sample_10_percent)
judge_agreement = compute_agreement(human_scores, llm_scores)
print(f"Judge 与人工的一致性: {judge_agreement}")
```

## 总结

### 核心要点

1. **Evals 是 LLM 应用开发的基础**：无法度量就无法优化
2. **两类验证方法**：确定性验证（快、便宜）+ LLM Judge（慢、贵但能评判语义）
3. **分层组合**：先跑确定性验证，再跑 LLM Judge，控制成本
4. **评估驱动开发**：先写评估，再优化系统，持续迭代
5. **持续监控**：生产环境也要定期评估，检测性能退化

### 实践建议

1. **从小开始**：先用 50 条样本做冒烟测试，建立评估习惯
2. **逐步扩展**：随着项目发展，增加到 100-500 条日常评估集
3. **自动化**：将评估集成到 CI/CD，每次提交自动跑评估
4. **持续迭代**：根据评估结果优化系统，形成正向循环

### 与其他课程的关系

- **可观测性（Lesson 33）**：提供评估所需的数据（日志、指标）
- **护栏（Lesson 31）**：确定性验证是护栏的一种形式
- **结构化输出（Lesson 35）**：便于进行确定性验证
