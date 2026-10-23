# 第 4 章学习笔记：任务规划与分解 — 让 Agent 学会拆解复杂任务

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch04-task-planning/
> 章节定位：核心篇 EP.04

---

## 一、核心概念

### 为什么 Agent 需要规划能力？
复杂任务（多步骤、多信息源）没有规划时容易：遗漏步骤、重复劳动、半途而废、质量不稳定。

### v0.7 中显式启用
`TodoListMiddleware` 需要显式传入 `middleware=[TodoListMiddleware()]`，v0.7 默认不安装。

### write_todos 工具
启用后获得：
1. `write_todos` 工具 — 创建和管理任务清单
2. `todos` 状态 — 保存任务及其状态
3. 规划指导提示词 — 引导 Agent 先规划再执行

### 三种任务状态
| 状态 | 含义 |
|------|------|
| `pending` | 待办 |
| `in_progress` | 进行中 |
| `completed` | 已完成 |

---

## 二、启用任务规划的代码

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_deep_agent(
    model=model,
    middleware=[TodoListMiddleware()],
)
```

### 自定义配置
```python
TodoListMiddleware(
    system_prompt="...",      # 自定义规划指导提示词
    tool_description="...",   # 自定义 write_todos 工具描述
)
```

---

## 三、LangChain 中间件机制

### 两类 Hook
| 风格 | Hook | 适合场景 |
|------|------|----------|
| Node-style | before_agent、before_model、after_model、after_agent | 校验、状态更新、审计、人工中断 |
| Wrap-style | wrap_model_call、wrap_tool_call | 重试、缓存、降级、请求/响应转换 |

### 默认中间件
- FilesystemMiddleware — 7 个文件工具 + 权限控制
- SummarizationMiddleware — 上下文自动压缩
- PatchToolCallsMiddleware — 补齐缺失的工具响应

### 通过 middleware 显式加入
- TodoListMiddleware — 任务规划
- ToolRetryMiddleware — 工具重试
- ModelFallbackMiddleware — 模型降级
- ToolCallLimitMiddleware — 限制工具调用次数

---

## 四、SummarizationMiddleware 两版本对比

| 对比项 | LangChain 版本 | Deep Agents 版本 |
|--------|----------------|------------------|
| 导入 | langchain.agents.middleware | deepagents.middleware |
| 执行位置 | before_model（独立节点） | wrap_model_call（model 节点内部） |
| 摘要后消息 | 替换旧消息为摘要 | 保留原始消息，模型输入用摘要+近期 |

手动配置示例：
```python
SummarizationMiddleware(
    model=model,
    backend=backend,
    trigger=("tokens", 4000),  # 何时开始摘要
    keep=("messages", 20),      # 保留多少近期消息
)
```

---

## 五、任务清单的持久化

- 保存在 Agent State 的 `todos` 字段
- 同一次运行内可访问
- 跨次 invoke 需要配置 Checkpointer + 复用 thread_id
- InMemorySaver 只在当前进程保存
- 子 Agent 默认不继承主 Agent 的 TodoListMiddleware

---

## 六、关键要点

1. `write_todos` 需要显式启用 TodoListMiddleware
2. 任务三种状态：pending → in_progress → completed
3. 中间件是 Agent 能力的插件机制
4. completed 标记不等于结果正确，需检查产物
5. 规划 + 上下文管理协同工作：摘要不会删除 todos 字段

---

## 学习日期
2026-10-20