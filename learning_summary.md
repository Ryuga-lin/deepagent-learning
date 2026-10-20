# Deep Agents 实战学习总结

> 学习时间：2026-10-17 ~ 2026-10-20
> 课程地址：https://datawhalechina.github.io/deepagents-in-action/
> 环境：原生 Windows PowerShell + Python 3.12 + GLM-5.2 (SiliconFlow)

---

## 一、学习进度总览

| 章节 | 标题 | 类型 | 状态 |
|------|------|------|------|
| 准备篇（上） | AgentSeek 模板创建 | 实操 | ✅ 完成 |
| 准备篇（下） | 开发技能安装 | 实操 | ✅ 完成 |
| 第 2 章 | 快速上手 — 第一个 Deep Agent | 理论+实操 | ✅ 完成 |
| 第 3 章 | 虚拟文件系统 | 理论 | ✅ 完成 |
| 第 4 章 | 任务规划与分解 | 理论+实操 | ✅ 完成 |

---

## 二、准备篇（上）：用 AgentSeek 创建 DeepAgents 模板

### 验证什么
从模板创建一个可运行的研究应用，包含前后端连通、模型配置、搜索服务集成。

### 改了什么

1. **环境搭建**：确认 Python 3.12、Node.js、npm、uv、agentseek 全部安装
2. **项目创建**：`agentseek create deepagents/research --no-input`
3. **依赖安装**：`agentseek task sync`（120 个 Python 包）+ `agentseek task frontend`（228 个前端包）
4. **.env 配置**：SiliconFlow + GLM-5.2 + Tavily 搜索

### 关键配置（.env）
```env
AGENTSEEK_MODEL_PROVIDER=openai
AGENTSEEK_MODEL=zai-org/GLM-5.2
OPENAI_API_BASE=https://api.siliconflow.cn/v1
OPENAI_API_KEY=sk-siloyfcybvtkiykqatjjzcmkuyettrdsmmjeebekmknufwon
TAVILY_API_KEY=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
SEEKDB_EMBED=false
METADATA_DB_URL=sqlite+aiosqlite:///./metadata.db
METADATA_DB_BACKEND=sqlite
```

### 实际结果
- ✅ `agentseek doctor` 全部 ok
- ✅ `agentseek dev` 启动成功，后端 `http://127.0.0.1:2024`，前端 `http://127.0.0.1:5174`
- ✅ 前端提交研究问题，Agent 正常流式输出

### 遇到的问题与修正

| 问题 | 原因 | 修正 |
|------|------|------|
| `--checkout main` 报错 | 模板仓库引用解析失败 | 去掉 `--checkout main` 参数 |
| SiliconFlow API 401 | API Key 无效 | 重新生成 Key |
| OceanBase 连接失败 | 嵌入式 OceanBase 仅支持 Linux | 改用 SQLite 模式 |
| LangSmith 403 | API Key 权限不足 | 暂时关闭 Tracing，不影响运行 |

---

## 三、准备篇（下）：为编码助手安装开发技能

### 验证什么
为 Claude Code 安装 `langchain-dev-guide` 和 `langsmith-trace` 两个开发技能。

### 改了什么

1. **查看可用技能**：`npx skills add ob-labs/agentseek --list` → 4 个技能可用
2. **GitHub 网络问题**：`npx skills add` 克隆失败（Connection was reset）
3. **手动克隆**：`git clone https://github.com/ob-labs/agentseek.git D:\ai_tools\agentseek-skills`
4. **本地安装**：`npx skills add D:\ai_tools\agentseek-skills --skill langchain-dev-guide --skill langsmith-trace --agent claude-code --yes`

### 实际结果
```
✓ langchain-dev-guide (copied)
  → .\.claude\skills\langchain-dev-guide
✓ langsmith-trace (copied)
  → .\.claude\skills\langsmith-trace
```

### 证据与解释
技能安装到 `.claude/skills/` 目录，Claude Code 可以自动读取。在 Claude Code 中输入引导提示后，它能引用技能内的参考文件，确认技能生效。

### 下一步修正
- LangSmith CLI 安装受网络限制（`cli.langsmith.com` 无法连接），暂用网页界面替代
- LangSmith API Key 权限问题待解决（需创建有读取权限的 Key）

---

## 四、第 2 章：快速上手 — 构建第一个 Deep Agent

### 验证什么
用 `create_deep_agent()` 创建最简单的 Agent，理解工具定义和模型-工具循环。

### 关键代码

**Hello World Agent（天气工具）：**
```python
model = ChatOpenAI(
    model="zai-org/GLM-5.2",
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful assistant.",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "北京今天天气怎么样？"}]}
)
print(result["messages"][-1].content)
```

**计算器 Agent（多工具）：**
```python
def calculate(expression: str) -> float:
    """Evaluate a math expression and return the result.
    Args:
        expression: A math expression with numbers and operators only.
    """
    import ast, operator
    ops = {ast.Add: operator.add, ast.Sub: operator.sub,
           ast.Mult: operator.mul, ast.Div: operator.truediv,
           ast.Pow: operator.pow, ast.USub: operator.neg}
    def _eval(node):
        if isinstance(node, ast.Constant): return float(node.value)
        if isinstance(node, ast.BinOp): return ops[type(node.op)](_eval(node.left), _eval(node.right))
        if isinstance(node, ast.UnaryOp): return ops[type(node.op)](_eval(node.operand))
        raise ValueError("Only numbers and operators allowed.")
    return _eval(ast.parse(expression, mode='eval').body)

def convert_currency(amount: float, from_currency: str, to_currency: str = "CNY") -> dict:
    """Convert an amount from one currency to another."""
    rates = {"USD": 7.2, "CNY": 1.0, "EUR": 7.8}
    cny = amount * rates[from_currency]
    return {"amount": round(cny / rates[to_currency], 2), "currency": to_currency}

agent = create_deep_agent(
    model=model,
    tools=[calculate, convert_currency],
    system_prompt="你是一个计算助手，能帮用户做数学运算和货币换算。",
)
result = agent.invoke(
    {"messages": [{"role": "user", "content": "帮我计算 720 * 1.08"}]}
)
print(result["messages"][-1].content)  # → 777.6
```

### 实际结果
- Hello World Agent：✅ 正常返回天气信息
- 计算器 Agent：✅ `720 * 1.08 = 777.6`

### 遇到的问题与修正

| 问题 | 原因 | 修正 |
|------|------|------|
| `NameError: name 'result' is not defined` | 7B 模型传变量名而非数字 | 换用 GLM-5.2 |
| GLM-5.2 仍传变量名 | 多步骤链式任务模型理解不足 | 简化为单步任务 |
| `eval()` 安全风险 | 直接执行用户输入 | 改用 AST 安全解析器 |

### 关键经验
- **7B 模型只能做单步调用**，多步骤链式任务必须用 GLM-5.2 级别模型
- 工具定义三要素：**参数类型标注 + Docstring + 默认值**
- `agent.invoke()` 输入 `{"messages": [...]}`，输出 `result["messages"][-1].content`

---

## 五、第 3 章：虚拟文件系统（理论）

### 核心知识点
Deep Agents 给 Agent 一个文件系统，不把所有信息塞进 prompt，而是按需读写。

**7 个内置工具：**

| 工具 | 用途 |
|------|------|
| `ls` | 列出目录 |
| `read_file` | 读取文件（支持分片、多模态） |
| `write_file` | 创建/覆盖文件 |
| `edit_file` | 精确字符串替换 |
| `delete` | 删除文件（v0.7 新增） |
| `glob` | 模式匹配查找 |
| `grep` | 全文检索（三种输出模式） |

**5 种存储后端：**
- StateBackend（默认，临时）
- FilesystemBackend（本地磁盘）
- StoreBackend（跨会话持久化）
- CompositeBackend（混合路由）
- 沙箱后端（安全执行）

**自动上下文管理：**
- 大结果（>20K tokens）自动卸载为文件引用
- 达到阈值（默认窗口 85%）自动摘要旧消息

---

## 六、第 4 章：任务规划与分解（理论+实操）

### 验证什么
显式启用 `TodoListMiddleware`，让 Agent 自主规划并执行多步骤文件管理任务。

### 关键代码
```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langchain.agents.middleware import TodoListMiddleware

model = ChatOpenAI(
    model="zai-org/GLM-5.2",
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

backend = FilesystemBackend(
    root_dir="D:/ai_tools/agent_workspace",
    virtual_mode=True,
)

agent = create_deep_agent(
    model=model,
    backend=backend,
    middleware=[TodoListMiddleware()],
    system_prompt="你是一个资料管理助手。面对复杂任务时，先用 write_todos 制定计划，然后逐步执行。",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": """
请完成以下资料整理任务：
1. 创建 /reports/langchain_notes.md，写入 LangChain 三个核心概念
2. 创建 /reports/langgraph_notes.md，写入 LangGraph 三个核心概念
3. 读取两个文件，确认内容已写入
4. 在 langchain_notes.md 末尾追加"总结"部分
5. 列出 /reports/ 目录下所有文件
"""}]
})
```

### 实际结果

**任务清单状态（todos）：**
```
● [completed] 创建文件 /reports/langchain_notes.md，写入 LangChain 三个核心概念
● [completed] 创建文件 /reports/langgraph_notes.md，写入 LangGraph 三个核心概念
● [completed] 读取两个文件，确认内容已写入
● [completed] 在 langchain_notes.md 末尾追加「总结」部分
● [completed] 列出 /reports/ 目录下所有文件
```

**输出文件：**

`D:/ai_tools/agent_workspace/reports/langchain_notes.md`（1195 字节）：
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

`D:/ai_tools/agent_workspace/reports/langgraph_notes.md`（536 字节）：
```markdown
# LangGraph 核心概念

## State（状态图）
State 是在图的不同节点之间流转和共享的结构化数据。

## Nodes（节点）
Nodes 是图中的执行单元，每个节点接收当前状态并执行特定逻辑。

## Edges（边）
Edges 定义了节点之间的连接与流转规则，包括无条件流转和条件分支。
```

### 工具调用记录

| 步骤 | 工具调用 | 状态变化 |
|------|----------|----------|
| 1 | `write_todos` | 创建 5 个 pending 任务 |
| 2 | `write_file` → `/reports/langchain_notes.md` | 任务 1 → in_progress → completed |
| 3 | `write_file` → `/reports/langgraph_notes.md` | 任务 2 → in_progress → completed |
| 4 | `read_file` × 2 | 任务 3 → in_progress → completed |
| 5 | `edit_file` → `/reports/langchain_notes.md`（追加总结） | 任务 4 → in_progress → completed |
| 6 | `ls` → `/reports/` | 任务 5 → in_progress → completed |
| 7 | 更新 todos（全部 completed） | 完成 |

### 证据与解释

1. **TodoListMiddleware 生效**：Agent 主动调用 `write_todos` 规划了 5 步，无需提示
2. **FilesystemBackend 真实写入**：文件实际存在于磁盘，`ls` 和 `cat` 可验证
3. **edit_file 精确修改**：langchain_notes.md 末尾成功追加"总结"部分，原内容未变
4. **任务状态完整流转**：5 个任务全部 pending → in_progress → completed
5. **模型自主性**：文件内容是 GLM-5.2 自己组织的，不是硬编码

### 下一步修正
- PowerShell `stream_events` 输出有缓冲延迟，考虑用 `invoke()` + 事后打印替代流式输出
- LangSmith Trace 因 API Key 权限问题无法上传，待修复后可完整分析调用链
- 下一步学习第 5 章：子 Agent 与上下文隔离

---

## 七、输出文件清单

| 文件 | 路径 | 用途 |
|------|------|------|
| hello_agent.py | D:\ai_tools\research_deepagent\ | Hello World Agent 示例 |
| calc_agent.py | D:\ai_tools\research_deepagent\ | 计算器 Agent 示例 |
| filesystem_agent.py | D:\ai_tools\research_deepagent\ | FilesystemBackend 示例 |
| todo_agent.py | D:\ai_tools\research_deepagent\ | TodoListMiddleware 基础版 |
| todo_agent_stream.py | D:\ai_tools\research_deepagent\ | TodoListMiddleware 流式版 |
| ch03_virtual_filesystem_notes.md | D:\ai_tools\research_deepagent\ | 第 3 章学习笔记 |
| ch04_task_planning_notes.md | D:\ai_tools\research_deepagent\ | 第 4 章学习笔记 |
| langchain_notes.md | D:\ai_tools\agent_workspace\reports\ | Agent 生成的文件 |
| langgraph_notes.md | D:\ai_tools\agent_workspace\reports\ | Agent 生成的文件 |

---

## 八、关键技术要点总结

1. **Deep Agents 核心价值**：你配置能力（模型+工具+中间件），Agent 自主完成任务
2. **模型选择**：简单任务用 7B，多步骤规划/链式调用必须用 GLM-5.2
3. **工具定义三要素**：类型标注 + Docstring + 默认值
4. **中间件是插件机制**：TodoListMiddleware 需显式加入，默认不安装
5. **虚拟文件系统**：Agent 的"草稿纸"，大结果自动卸载，旧消息自动摘要
6. **后端选择**：StateBackend（默认临时）、FilesystemBackend（本地磁盘）
7. **权限控制**：FilesystemPermission 声明式配置，virtual_mode 建议开启
8. **Windows 注意事项**：嵌入式 OceanBase 仅支持 Linux，需改用 SQLite 模式

---

> 生成时间：2026-10-20
> 作者：Ryuga-lin