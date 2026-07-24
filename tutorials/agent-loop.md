# Agent 执行循环（Loop）

Agent 执行循环是 Harness 的核心运行机制，描述了 Agent 如何通过"感知→推理→行动→观察"的迭代过程自主完成复杂任务。

## 核心概念

![Agent 执行循环](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/agent-loop-cycle_1784689302.png)

### 什么是 Agent Loop

Agent Loop（智能体循环）是一个闭环执行过程：

```
接收输入 → 推理/规划 → 执行动作 → 观察结果 → (循环或终止)
```

每一轮循环包含四个阶段：
1. **接收输入**：获取用户指令、环境反馈或上一步的执行结果
2. **推理/规划**：LLM 基于当前上下文决定下一步行动
3. **执行动作**：调用工具或直接输出结果
4. **观察结果**：将工具返回加入上下文，作为下一轮推理依据

### 为什么需要 Loop

单轮对话只能解决简单问题。复杂任务需要多步骤执行：
- 读取多个文件理解代码库
- 调用工具搜索相关信息
- 生成代码并验证结果
- 根据错误反馈自我修正

这些都需要循环迭代，而不是一次性完成。

## 典型实现：Claude Code

Claude Code 是 Agent Loop 的典型实现：

```
1. 接收用户指令："修复这个 bug"
2. 推理：需要先看代码，决定用 Read 工具
3. 执行：读取 src/app.js
4. 观察：看到代码内容，发现 bug 在第 42 行
5. 推理：需要修改代码，决定用 Edit 工具
6. 执行：修改第 42 行
7. 观察：修改成功
8. 推理：需要验证修复，决定用 Bash 工具
9. 执行：运行 npm test
10. 观察：测试通过
11. 推理：任务完成，生成总结
12. 输出：向用户报告修复结果
```

这个过程中，Agent 自主决策、多步骤执行、根据结果调整策略，最终完成任务。

## 工程层面的关键问题

### 1. 终止条件设计

循环必须有明确的退出机制，否则会变成无限循环：

**常见终止条件：**
- **任务完成**：Agent 自己判断任务已完成（输出最终结果）
- **最大迭代次数**：达到预设的循环上限（如 50 轮）
- **Token 预算耗尽**：上下文长度达到模型限制
- **超时**：执行时间超过预设阈值
- **需要人工确认**：触发 Confirmation Gate，暂停等待用户

**最佳实践：**
```python
max_iterations = 50
for i in range(max_iterations):
    result = agent.step()
    if result.is_final:
        break
    if result.needs_confirmation:
        user_input = get_user_confirmation()
        agent.add_to_context(user_input)
else:
    # 达到最大迭代次数
    agent.output("任务未完成，需要人工介入")
```

### 2. 上下文管理

![长任务中的上下文压缩](https://yk-ai-mentor.youku.com/yk_ai_mentor/tutorials/ch1/images/context-compaction_1784689260.png)

每次循环都会累积新的信息，上下文会快速增长：

**问题：**
- 工具调用结果可能很长（如读取大文件）
- 多轮对话历史不断叠加
- Token 消耗线性增长，成本飙升
- 上下文过长导致"Lost in the Middle"问题

**解决方案：**

**摘要压缩**：定期将历史对话压缩为摘要
```python
if len(context.messages) > 20:
    summary = llm.summarize(context.messages[:15])
    context.messages = [summary] + context.messages[15:]
```

**滑动窗口**：只保留最近 N 轮对话
```python
context.messages = context.messages[-10:]  # 只保留最近 10 轮
```

**选择性保留**：根据重要性过滤信息
```python
# 保留关键信息：用户指令、错误信息、最终结果
context.messages = [
    msg for msg in context.messages 
    if msg.is_user_input or msg.is_error or msg.is_final
]
```

### 3. 错误恢复

工具调用可能失败，Agent 需要有恢复机制：

**重试策略**：
```python
for attempt in range(3):  # 最多重试 3 次
    try:
        result = tool.execute(params)
        break
    except ToolError as e:
        if attempt == 2:
            raise  # 最后一次失败，抛出异常
        wait(2 ** attempt)  # 指数退避：1s, 2s, 4s
```

**降级策略**：
```python
try:
    result = primary_tool.execute(params)
except ToolError:
    # 主工具失败，尝试备用工具
    result = fallback_tool.execute(params)
```

**放弃并报告**：
```python
try:
    result = tool.execute(params)
except ToolError as e:
    agent.output(f"工具调用失败：{e}，无法继续执行任务")
    return
```

### 4. 成本控制

一次 Loop 可能消耗数十万 Token，需要控制成本：

**单次循环限制**：
```python
MAX_TOKENS_PER_STEP = 50000
if context.token_count > MAX_TOKENS_PER_STEP:
    context.compress()  # 压缩上下文
```

**总预算限制**：
```python
total_tokens = 0
BUDGET = 500000  # 总预算 50 万 Token

for i in range(max_iterations):
    step_tokens = agent.step()
    total_tokens += step_tokens
    if total_tokens > BUDGET:
        agent.output("预算耗尽，任务未完成")
        break
```

**成本估算**：
```python
# 预估剩余 Token
remaining_budget = BUDGET - total_tokens
estimated_tokens_per_step = 10000
remaining_steps = remaining_budget // estimated_tokens_per_step

if remaining_steps < 5:
    agent.output("预算不足，建议简化任务或增加预算")
```

## 与其他组件的关系

### 与前馈/反馈控制（Lesson 30）

- **前馈控制**：在 Loop 开始前注入 System Prompt、工具定义
- **反馈控制**：在每轮循环后检查结果、注入新信息

### 与护栏（Lesson 31）

- 护栏在每轮循环中检查：
  - 工具调用是否安全
  - 输出是否包含敏感信息
  - 是否需要人工确认

### 与可观测性（Lesson 33）

- 可观测性记录每轮循环的：
  - 工具调用详情
  - Token 消耗
  - 执行时间
  - 错误信息

### 与评估（Lesson 34）

- 评估在 Loop 结束后：
  - 检查最终输出是否符合预期
  - 统计循环次数、Token 消耗等指标
  - 评估 Agent 的决策质量

## 常见误区

### 误区1：Loop 越多越好

**错误认知**：让 Agent 循环更多次，任务完成得更好。

**现实**：循环次数过多会导致：
- 上下文过长，信息丢失
- Token 成本飙升
- 执行时间过长，用户体验差

**正确做法**：
- 设置合理的最大迭代次数
- 在每轮循环后检查是否可以提前终止
- 优化 Agent 的推理能力，减少不必要的循环

### 误区2：Agent 应该自主完成所有任务

**错误认知**：Agent 应该完全自主，不需要人工介入。

**现实**：某些任务需要人工确认：
- 高风险操作（删除文件、修改配置）
- 不确定的决策（选择哪个方案）
- 超出 Agent 能力范围的任务

**正确做法**：
- 设计 Confirmation Gate，在关键节点暂停
- 让 Agent 在不确定时主动请求人工帮助
- 明确 Agent 的能力边界

### 误区3：Loop 中的错误可以忽略

**错误认知**：工具调用失败就跳过，继续执行下一步。

**现实**：错误会累积，导致：
- 最终结果不正确
- 浪费 Token 和时间
- 难以调试和定位问题

**正确做法**：
- 实现错误恢复机制（重试、降级）
- 记录所有错误信息
- 在关键错误时终止任务

## 最佳实践

### 1. 设计清晰的终止条件

```python
def should_terminate(agent_state):
    # 任务完成
    if agent_state.has_final_answer:
        return True
    
    # 达到最大迭代次数
    if agent_state.iteration >= MAX_ITERATIONS:
        return True
    
    # Token 预算耗尽
    if agent_state.total_tokens >= BUDGET:
        return True
    
    # 需要人工确认
    if agent_state.needs_confirmation:
        return True
    
    return False
```

### 2. 实现上下文压缩

```python
def compress_context(context):
    # 保留用户指令
    user_messages = [msg for msg in context if msg.role == 'user']
    
    # 保留最近的工具调用结果
    recent_tool_results = context[-5:]
    
    # 压缩早期对话
    early_messages = context[:-5]
    summary = llm.summarize(early_messages)
    
    return [summary] + user_messages + recent_tool_results
```

### 3. 记录循环详情

```python
for i in range(max_iterations):
    # 记录每轮循环的详情
    step_log = {
        'iteration': i,
        'timestamp': datetime.now(),
        'action': None,
        'result': None,
        'tokens_used': 0,
    }
    
    # 执行一步
    action, result = agent.step()
    step_log['action'] = action
    step_log['result'] = result
    step_log['tokens_used'] = count_tokens(result)
    
    # 保存日志
    logs.append(step_log)
```

### 4. 提供进度反馈

```python
for i in range(max_iterations):
    # 显示进度
    progress = f"步骤 {i+1}/{max_iterations}"
    agent.output(progress, to_user=True)
    
    # 执行一步
    result = agent.step()
    
    # 显示当前行动
    if result.action:
        agent.output(f"正在执行：{result.action.description}", to_user=True)
```

## 总结

Agent Loop 是 Harness 的核心运行机制，通过"感知→推理→行动→观察"的迭代过程自主完成复杂任务。

**关键要点：**
1. Loop 是一个闭环过程，每轮循环包含四个阶段
2. 必须有明确的终止条件，防止无限循环
3. 需要管理上下文，避免 Token 成本飙升
4. 实现错误恢复机制，提高可靠性
5. 控制成本，设置 Token 预算限制

**与其他组件的关系：**
- 前馈/反馈控制：在循环前后注入信息
- 护栏：在循环中检查安全性
- 可观测性：记录循环详情
- 评估：评价循环结果

**常见误区：**
- Loop 越多越好（应该优化质量而非数量）
- Agent 应该完全自主（需要人工确认关键节点）
- 错误可以忽略（错误会累积，必须处理）
