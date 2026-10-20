# 第 4 章学习心得：任务规划与分解 — 让 Agent 学会拆解复杂任务

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch04-task-planning/
> 学习时间：2026-10-20
> 环境：原生 Windows PowerShell + Python 3.12 + GLM-5.2 (SiliconFlow)

---

## 一、验证什么

显式启用 `TodoListMiddleware`，让 Agent 自主规划并执行多步骤文件管理任务，验证：
- Agent 是否会主动调用 `write_todos` 制定计划
- 任务状态是否能完整流转（pending → in_progress → completed）
- `FilesystemBackend` 是否真实写入本地磁盘
- `edit_file` 是否能精确修改文件内容
- 多步骤任务是否能链式执行完毕

---

## 二、改了什么

### 2.1 环境配置

```powershell
cd D:\ai_tools\research_deepagent
$env:SILICONFLOW_API_KEY="<已脱敏>"
```

### 2.2 关键代码

```python
import os
os.environ["LANGSMITH_TRACING"] = "false"
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langchain.agents.middleware import TodoListMiddleware

# 1. 配置模型（GLM-5.2，适合多步骤规划任务）
model = ChatOpenAI(
    model=os.environ.get("MODEL_NAME", "zai-org/GLM-5.2"),
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

# 2. 配置本地文件系统后端
backend = FilesystemBackend(
    root_dir="D:/ai_tools/agent_workspace",
    virtual_mode=True,
)

# 3. 创建 Agent，显式启用 TodoListMiddleware
agent = create_deep_agent(
    model=model,
    backend=backend,
    middleware=[TodoListMiddleware()],
    system_prompt="""你是一个资料管理助手。
面对复杂任务时，你会先用 write_todos 制定计划，然后逐步执行并更新进度。
你可以使用 write_file、read_file、edit_file、ls、grep 等文件工具来完成任务。""",
)

# 4. 多步骤任务
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": """请完成以下资料整理任务：
1. 创建文件 /reports/langchain_notes.md，写入 LangChain 的三个核心概念
2. 创建文件 /reports/langgraph_notes.md，写入 LangGraph 的三个核心概念
3. 读取两个文件，确认内容已写入
4. 在 langchain_notes.md 末尾追加一个"总结"部分
5. 列出 /reports/ 目录下所有文件
"""
    }]
})

print(result["messages"][-1].content)
for todo in result.get("todos", []):
    print(f"  [{todo.get('status')}] {todo.get('content')}")
```

### 2.3 关键改动说明

| 改动 | 原因 |
|------|------|
| `os.environ["LANGSMITH_TRACING"] = "false"` | LangSmith API Key 权限不足导致 403，关闭 Tracing 消除噪音 |
| `virtual_mode=True` | 开启路径沙箱，阻止 `..` 和越界路径访问 |
| 使用 GLM-5.2 而非 Qwen 7B | 7B 模型无法处理多步骤链式任务（见第 2 章验证） |

---

## 三、实际结果

### 3.1 Agent 最终回复

```
全部任务已完成。以下是执行情况汇总：

1. 创建 /reports/langchain_notes.md ✅
   写入了 LangChain 三个核心概念：Chains、Agents、Memory
2. 创建 /reports/langgraph_notes.md ✅
   写入了 LangGraph 三个核心概念：State、Nodes、Edges
3. 读取确认 ✅
4. 追加"总结"部分 ✅
5. /reports/ 目录文件列表：langchain_notes.md, langgraph_notes.md（共 2 个文件）
```

### 3.2 任务清单状态（todos）

```
● [completed] 创建文件 /reports/langchain_notes.md，写入 LangChain 三个核心概念
● [completed] 创建文件 /reports/langgraph_notes.md，写入 LangGraph 三个核心概念
● [completed] 读取两个文件，确认内容已写入
● [completed] 在 langchain_notes.md 末尾追加「总结」部分
● [completed] 列出 /reports/ 目录下所有文件
```

5 个任务全部 `pending` → `in_progress` → `completed`，状态流转完整。

### 3.3 输出文件

**`/reports/langchain_notes.md`（1195 字节）：**

```markdown
# LangChain 核心概念

## Chains（链式调用）
Chains 将多个组件按顺序串联起来，使前一步的输出能够作为下一步的输入。

## Agents（智能代理）
Agents 让大语言模型根据用户意图动态决定调用哪些工具或采取何种行动。

## Memory（记忆机制）
Memory 使链或代理在多轮交互中保留上下文信息。

## 总结
LangChain 与 LangGraph 同属 LangChain 生态体系。LangChain 侧重"能力拼装"，
LangGraph 侧重"复杂流程编排"，二者配合可构建从简单链到高级多智能体系统的完整应用。
```

**`/reports/langgraph_notes.md`（536 字节）：**

```markdown
# LangGraph 核心概念

## State（状态图）
State 是在图的不同节点之间流转和共享的结构化数据。

## Nodes（节点）
Nodes 是图中的执行单元，每个节点接收当前状态并执行特定逻辑。

## Edges（边）
Edges 定义了节点之间的连接与流转规则，包括无条件流转和条件分支。
```

---

## 四、工具调用记录

| 步骤 | 工具调用 | 状态变化 |
|------|----------|----------|
| 1 | `write_todos` | 创建 5 个 pending 任务 |
| 2 | `write_file` → `/reports/langchain_notes.md` | 任务 1 → in_progress → completed |
| 3 | `write_file` → `/reports/langgraph_notes.md` | 任务 2 → in_progress → completed |
| 4 | `read_file` × 2 | 任务 3 → in_progress → completed |
| 5 | `edit_file` → `/reports/langchain_notes.md`（追加总结） | 任务 4 → in_progress → completed |
| 6 | `ls` → `/reports/` | 任务 5 → in_progress → completed |
| 7 | 更新 todos（全部 completed） | 全部完成 |

---

## 五、证据与解释

1. **TodoListMiddleware 生效**：Agent 主动调用 `write_todos` 规划了 5 步，无需在提示词中逐条指示。这证明 `middleware=[TodoListMiddleware()]` 成功注入了 `write_todos` 工具和规划提示词。

2. **FilesystemBackend 真实写入**：通过 `ls -la D:/ai_tools/agent_workspace/reports/` 验证，两个文件实际存在于磁盘（1195 字节和 536 字节），不是虚拟存储。

3. **edit_file 精确修改**：`langchain_notes.md` 末尾成功追加了"总结"部分，原始的三个概念内容未被覆盖或修改。这验证了 `edit_file` 的精确字符串替换能力（不同于 `write_file` 的完整覆盖）。

4. **任务状态完整流转**：5 个任务全部经历了 `pending` → `in_progress` → `completed` 三种状态，证明 Agent 在每步执行前更新状态为 `in_progress`，执行后更新为 `completed`。

5. **模型自主性**：文件内容（概念解释、总结段落）是 GLM-5.2 自己组织语言生成的，不是硬编码。用户只提供了"写什么主题"，Agent 自己决定"怎么写"。

6. **规划与上下文管理协同**：教程指出对话总结不会删除 `todos` 字段。虽然本任务未触发总结阈值，但任务清单作为"锚点"帮助 Agent 在整个执行过程中保持对进度的把控。

---

## 六、下一步修正

| 待修正项 | 原因 | 计划 |
|----------|------|------|
| PowerShell `stream_events` 输出缓冲延迟 | 终端缓冲机制导致实时输出不刷新 | 用 `invoke()` + 事后打印替代，或加 `flush=True` |
| LangSmith Trace 无法上传 | API Key 权限不足（403 Forbidden） | 创建有完整权限的 Key，或使用网页界面查看 |
| LangSmith CLI 无法安装 | `cli.langsmith.com` 网络不通 | 尝试 GitHub Releases 下载二进制，或使用网页替代 |
| 未验证跨次 invoke 的 todos 接续 | 需配置 Checkpointer + thread_id | 下一阶段配置 `InMemorySaver` 测试 |

---

## 七、关键代码文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| todo_agent.py | `D:\ai_tools\research_deepagent\` | TodoListMiddleware 基础版（invoke） |
| todo_agent_stream.py | `D:\ai_tools\research_deepagent\` | TodoListMiddleware 流式版（stream_events） |
| ch04_task_planning_notes.md | `D:\ai_tools\research_deepagent\` | 第 4 章理论笔记 |
| langchain_notes.md | `D:\ai_tools\agent_workspace\reports\` | Agent 生成的文件 |
| langgraph_notes.md | `D:\ai_tools\agent_workspace\reports\` | Agent 生成的文件 |

---

## 八、本章核心知识点

1. **TodoListMiddleware 需显式启用**：v0.7 默认不安装，通过 `middleware=[TodoListMiddleware()]` 加入
2. **write_todos 工具**：让 Agent 创建任务清单，包含 `content` 和 `status` 两个字段
3. **三种任务状态**：`pending`（待办）→ `in_progress`（进行中）→ `completed`（已完成）
4. **中间件是插件机制**：`create_deep_agent()` 的本质是把一组中间件自动组装到 Agent 上
5. **两类 Hook**：Node-style（校验/审计/中断）vs Wrap-style（重试/缓存/转换）
6. **默认中间件**：FilesystemMiddleware（文件工具）、SummarizationMiddleware（上下文压缩）、PatchToolCallsMiddleware（补齐缺失响应）
7. **completed ≠ 正确**：清单全部完成不代表结果正确，需检查实际产物
8. **规划与上下文管理协同**：对话总结不会删除 `todos` 字段，清单帮助 Agent 在总结后继续追踪进度

---

> 生成时间：2026-10-20
> 作者：Ryuga-lin