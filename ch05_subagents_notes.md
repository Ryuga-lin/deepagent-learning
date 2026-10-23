# 第 5 章学习笔记：子 Agent 与上下文隔离 — 让 Agent 学会委派

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch05-subagents/
> 配套 Notebook：https://github.com/datawhalechina/deepagents-in-action/blob/main/notebooks/ch05/01-subagent-delegation.ipynb
> 学习时间：2026-10-22（理论） / 2026-10-23（补全实操与复习）

> ⚠️ **公开仓库安全提示**：本文档面向 GitHub 公开仓库，**全文不出现任何 API Key / Token / 密码**。
> 所有密钥均通过环境变量注入（如 `os.environ["DEEPSEEK_API_KEY"]`），真实密钥只存放在本地 `.env`，
> 而 `.env` 已被 `.gitignore` 忽略，**切勿提交**。下面代码中的 `api_key` 一律引用环境变量名，而非明文。

---

## 一、核心动机：Context Quarantine（上下文隔离）

### 问题：上下文膨胀
主 Agent 完成一个研究报告的子任务（如"搜索 LangGraph 文档"）可能涉及：
- 5 次网络搜索，每次返回 3000+ tokens
- 多次 write_file / read_file

这些中间过程全部堆在主 Agent 上下文里。虽然自动卸载能缓解一部分，但**主 Agent 根本不需要知道这些细节**——它只需要最终摘要。

### 解决：子 Agent 委派
1. 主 Agent 通过 `task` 工具创建子 Agent
2. 子 Agent 在**独立上下文**中执行（中间调用记录不进入主 Agent；共用 Backend 时文件仍共享）
3. 子 Agent 完成后只把**最终结果**返回给主 Agent
4. 主 Agent 上下文保持干净

类比：主 Agent 是项目经理，子 Agent 是专项负责人。项目经理不需要参加每个技术讨论会，只看总结报告。

---

## 二、什么时候用子 Agent？

| 场景 | 是否用 | 理由 |
|------|--------|------|
| 需要多次搜索和整理的研究任务 | ✅ | 大量中间结果会膨胀主 Agent 上下文 |
| 需要特殊工具或指令的专业任务 | ✅ | 子 Agent 可以有自己的工具集和系统提示词 |
| 需要不同模型能力的任务 | ✅ | 子 Agent 可以用不同的模型 |
| 需要高层协调的复杂任务 | ✅ | 主 Agent 专注协调，子 Agent 专注执行 |
| 单步简单查询 | ❌ | 委派开销大于收益 |
| 需要主 Agent 看到子 Agent 完整调用过程 | ❌ | 主 Agent 只收到最终结果 |

---

## 三、定义子 Agent：字典方式

```python
research_subagent = {
    "name": "researcher",                # 必填：唯一标识符
    "description": "深入研究特定主题，搜索多个信息源并整理成摘要",  # 必填：主 Agent 靠它决定何时委派
    "system_prompt": """你是一位专业的研究员。...""",  # 必填：子 Agent 自己的指令
    "tools": [internet_search],          # 可选，默认继承；显式指定后完全替换（不合并）
    "skills": ["/skills/research/"],     # 可选，不继承主 Agent
}

agent = create_deep_agent(
    model=model,
    skills=["/skills/main/"],            # 主 Agent 和 general-purpose 子 Agent 继承此处
    subagents=[research_subagent],
)
```

### 字段说明

| 字段 | 必填 | 继承主 Agent？ | 说明 |
|------|------|---------------|------|
| `name` | ✅ | — | 唯一标识符，主 Agent 通过它指定委派给谁 |
| `description` | ✅ | — | 描述能力，主 Agent 靠它决策路由 |
| `system_prompt` | ✅ | ❌ 不继承 | 子 Agent 自己的指令 |
| `tools` | 可选 | ✅ 默认继承，指定后完全替换 | 不指定则继承，指定后完全替换（不合并） |
| `model` | 可选 | ✅ 默认继承 | 可指定不同模型 |
| `middleware` | 可选 | ❌ 不继承 | 子 Agent 自己的中间件 |
| `interrupt_on` | 可选 | ✅ 默认继承 | 人工审批配置，可覆盖 |
| `skills` | 可选 | ❌ 不继承 | 指定后运行独立 SkillsMiddleware |
| `response_format` | 可选 | ❌ 不继承 | 结构化输出 schema（需 deepagents>=0.5.3） |
| `permissions` | 可选 | ✅ 默认继承，指定后完全替换 | 文件系统权限规则 |

### 关键点
- **system_prompt 不继承**：每个子 Agent 应有自己专属的指令
- **tools 默认继承**，显式指定后**完全替换**（不合并）
- **skills 不继承**：主 Agent 的 skills 只传给 general-purpose 子 Agent
- **permissions 省略时继承，显式提供后整体替换**
- v0.7 默认配置中，**默认 general-purpose 和声明式子 Agent 都不继承主 Agent 的 TodoListMiddleware**，需要时在子 Agent 自己的 middleware 字段中添加

---

## 四、General-purpose 子 Agent

不定义任何子 Agent 时，Deep Agents 自带一个 **general-purpose 子 Agent**：
- 继承主 Agent 显式传入的 `tools`、`model` 和 `skills`
- 使用自己的默认系统提示词（DEFAULT_SUBAGENT_PROMPT）
- **不继承主 Agent 的 system_prompt**
- 中间件只继承主 Agent 对默认中间件的**同名覆盖**

```python
agent = create_deep_agent(
    model=model,
    tools=[internet_search],
    system_prompt="你是一位研究助手。",  # 不会传给默认子 Agent
)
# 主 Agent 可以这样委派：
# task(subagent_type="general-purpose", description="...")
```

### 覆盖 general-purpose 子 Agent
用 `name="general-purpose"` 覆盖默认配置（system_prompt、model、tools 等）。

### 为 general-purpose 启用 Todo
在 subagents 中用同名 spec 显式配置 middleware：
```python
subagents=[{
    "name": "general-purpose",
    "description": "处理需要规划的委派任务",
    "system_prompt": "为委派任务制定计划，执行后返回结果。",
    "middleware": [TodoListMiddleware()],  # 子 Agent 单独启用
}]
```

### 禁用 general-purpose 子 Agent
```python
from deepagents.profiles import GeneralPurposeSubagentProfile, HarnessProfile, register_harness_profile

register_harness_profile(
    key="openai:zai-org/GLM-5.2",
    profile=HarnessProfile(
        general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False)
    ),
)
agent = create_deep_agent(model=model, subagents=[])
```
⚠️ 不要用 `excluded_middleware` 排除 SubAgentMiddleware，会直接抛 ValueError。

---

## 五、CompiledSubAgent：集成 LangGraph 工作流

用预构建的 LangGraph 图作为子 Agent，适合多步骤、有分支逻辑的工作流：

```python
from deepagents import create_deep_agent, CompiledSubAgent
from langchain.agents import create_agent

custom_graph = create_agent(
    model=model,
    tools=[statistical_analysis, generate_chart],
    system_prompt="你是数据分析专家。",
)

data_subagent = CompiledSubAgent(
    name="data-analyzer",
    description="执行复杂的数据分析任务",
    runnable=custom_graph,
)
```

要求：传入的 LangGraph 图的 State 中必须包含 `"messages"` 键。

---

## 六、多子 Agent 协作模式

```python
subagents = [
    {
        "name": "data-collector",
        "description": "从多个来源收集原始数据",
        "system_prompt": "你是数据收集专家。",
        "tools": [internet_search, api_call],
    },
    {
        "name": "data-analyzer",
        "description": "对收集到的数据进行统计分析",
        "system_prompt": "你是数据分析专家。控制在 300 字以内。",
        "tools": [statistical_analysis],
    },
    {
        "name": "report-writer",
        "description": "根据分析结果撰写专业报告",
        "system_prompt": "你是技术写作专家。",
        "tools": [format_document],
    },
]

agent = create_deep_agent(
    model=model,
    middleware=[TodoListMiddleware()],
    system_prompt="""你是一位项目协调者。面对复杂任务时：
1. 先用 write_todos 制定计划
2. 将数据收集委派给 data-collector
3. 将分析工作委派给 data-analyzer
4. 将报告撰写委派给 report-writer
5. 整合各子 Agent 的输出""",
    subagents=subagents,
)
```

执行流程：主 Agent 规划 → task() 委派给各子 Agent → 子 Agent 独立上下文工作 → 主 Agent 整合输出。

---

## 七、结构化输出：让子 Agent 返回 JSON

```python
from pydantic import BaseModel, Field

class ResearchFindings(BaseModel):
    summary: str = Field(description="研究摘要")
    confidence: float = Field(description="置信度 0-1")
    sources: list[str] = Field(description="信息来源 URL 列表")

research_subagent = {
    "name": "researcher",
    "description": "研究特定主题并返回结构化发现",
    "system_prompt": "深入研究给定主题，返回你的发现。",
    "tools": [internet_search],
    "response_format": ResearchFindings,  # 需 deepagents>=0.5.3
}
```
设置后主 Agent 收到符合 schema 的 JSON，而非自由文本。

---

## 八、子 Agent 最佳实践

1. **描述要具体**：主 Agent 靠 description 决定何时委派
   - ✅ "执行深度网络研究，需要多次搜索、信息交叉验证和综合分析时使用"
   - ❌ "做研究"
2. **System Prompt 要详细**：包含输出格式要求和字数限制
3. **工具集要精简**：最小权限原则，只给需要的工具
4. **不同子 Agent 用不同模型**：轻量任务用 7B，复杂推理用 GLM-5.2
5. **返回结果要精简**：在 system_prompt 中明确要求只返回核心内容，否则失去上下文隔离的意义

---

## 九、常见问题排查

| 问题 | 原因 | 解法 |
|------|------|------|
| 子 Agent 没被调用 | 主 Agent 无法从 description 判断何时委派 | description 更具体；主 Agent system_prompt 明确指示委派 |
| 上下文依然膨胀 | 子 Agent 返回大量原始数据 | system_prompt 强制简洁返回；让子 Agent 写文件只返回摘要 |
| 调用了错误的子 Agent | 多个子 Agent description 过于相似 | description 中明确区分使用场景 |

---

## 十、小结

1. **核心动机**：Context Quarantine——子 Agent 独立上下文工作，只返回精炼结果
2. **字典方式定义**：必填 name + description + system_prompt；tools 默认继承但显式指定后完全替换
3. **General-purpose 子 Agent**：默认可用，继承显式工具/模型/skills，用自己的默认提示词
4. **CompiledSubAgent**：预构建 LangGraph 图作为子 Agent
5. **结构化输出**：response_format 让子 Agent 返回 JSON
6. **最佳实践**：描述具体、提示词详细、工具精简、模型按需、返回精炼

---

## 十一、我的实操记录（subagent_delegate.py 完整实现）

> 目标：把"整理一份 LangGraph 概念笔记"的工作**委派给子 Agent**，并在子 Agent 上自定义职责提示词（`system_prompt`）与工具映射（`tools`）。
> 环境：Windows PowerShell + DeepAgents；主 Agent 模型通过 DeepSeek 官方 API（环境变量 `DEEPSEEK_API_KEY`）接入；文件后端用 `FilesystemBackend`（虚拟模式）指向本地工作区。

### 11.1 子 Agent 职责提示词 + 工具映射（核心改造点）

```python
note_writer_subagent = {
    "name": "note-writer",
    "description": "把结构化内容写入 Markdown 笔记文件。需要整理课程笔记、技术资料、会议记录时委派给此 Agent。",
    "system_prompt": """你是一位专业的技术笔记整理专家。
职责：
1. 接收主 Agent 转交的资料整理任务
2. 按指定 Markdown 格式写入文件
3. 写完后读取文件确认内容完整
格式要求：
- 仅一个一级标题
- 每章节用二级标题
- 每个概念一句话说明，不超过 50 字
- 文件末尾必须有"学习日期"一行
约束：
- 只返回文件路径和一句话摘要给主 Agent，不要返回完整内容
- 同名文件存在时先读取旧文件再追加，不要覆盖""",
    "tools": [],  # 显式指定为空 → 完全替换主 Agent 工具集（文件工具由 FilesystemMiddleware 默认提供）
}
```

> **改造点回顾**：
> - `system_prompt` 不继承主 Agent，这里写了"笔记整理专家"专属指令 + 格式约束（对应课程第二节"每个子 Agent 要有专属提示词"）。
> - `tools=[]` 显式指定 → **完全替换**主 Agent 工具（不合并），实现最小权限（对应课程字段说明：`tools` 指定后完全替换）。

### 11.2 主 Agent：协调者，强制委派

```python
agent = create_deep_agent(
    model=model,
    backend=backend,
    middleware=[TodoListMiddleware()],
    system_prompt="""你是一位项目协调者。
1. 面对任务先用 write_todos 制定计划
2. 把"写入笔记文件"类工作委派给 note-writer 子 Agent
3. 子 Agent 完成后，向用户汇报结果
重要：涉及创建和写入 Markdown 笔记文件的工作，一律使用 task() 委派给 note-writer，不要自己动手写。""",
    subagents=[note_writer_subagent],
)
```

### 11.3 触发委派

```python
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "请把以下 LangGraph 核心概念整理成笔记，保存到 /notes/langgraph_basics.md：\n"
                   "1. State ... 2. Nodes ... 3. Edges ... 4. Checkpointer ...\n学习日期：2026-10-22"
    }]
})
```

执行链路：主 Agent 规划 → `task(subagent_type="note-writer")` → 子 Agent 独立上下文写文件 → 主 Agent 收到精炼摘要。

---

## 十二、如何验证"委派确实发生"

1. **看文件产物**：子 Agent 生成的 `D:/ai_tools/agent_workspace/notes/langgraph_basics.md`
   严格符合 11.1 的格式约束（一个 H1、四个 H2、每概念一句话、末尾"学习日期"）——说明是子 Agent 自己写的，而非主 Agent 代写。
2. **看消息流**：脚本末尾打印「主 Agent 消息流」，出现 `调用工具: task` 且 `委派给: note-writer` 即证明委派成功。
3. **验证结果（本次实操）**：文件于 2026-10-23 14:37 生成，内容含 State / Nodes / Edges / Checkpointer 四个概念，格式完全合规 → **第五章实操目标达成 ✅**。

---

## 十三、回忆复习 · 自测题（含答案）

**Q1. 子 Agent 机制的核心动机是什么？**
A. Context Quarantine（上下文隔离）：子 Agent 在独立上下文执行，中间过程不进主 Agent，只返回精炼结果，避免主 Agent 上下文膨胀。

**Q2. 主 Agent 通过什么工具委派任务？参数有哪些？**
A. `task()` 工具；关键参数 `subagent_type`（委派给谁）、`description`（任务描述）。

**Q3. 字典方式定义子 Agent 哪些字段是必填的？**
A. `name`、`description`、`system_prompt` 三个必填。

**Q4. 子 Agent 的 `tools` 字段不写和写了有什么区别？**
A. 不写 → 默认继承主 Agent 工具；显式写了（如 `tools=[]`）→ **完全替换**（不合并）。

**Q5. `system_prompt` 和 `skills` 是否继承主 Agent？**
A. **都不继承**。`system_prompt` 每个子 Agent 必须自己写；`skills` 指定后子 Agent 独立运行 SkillsMiddleware。

**Q6. 不定义任何子 Agent 时，默认存在哪个？它继承什么？**
A. `general-purpose` 子 Agent；继承主 Agent 显式传入的 `tools`、`model`、`skills`，但**不继承主 Agent 的 `system_prompt`**，可用 `name="general-purpose"` 的同名 spec 覆盖。

**Q7. 怎样彻底禁用 general-purpose？为什么不能用 excluded_middleware？**
A. 用 `register_harness_profile` 注册 `GeneralPurposeSubagentProfile(enabled=False)`；`excluded_middleware` 排除 `SubAgentMiddleware` 会直接抛 `ValueError`。

**Q8. CompiledSubAgent 适合什么场景？有什么硬性要求？**
A. 适合把已有 LangGraph 图作为子 Agent（多步骤/分支工作流）；要求图的 State 必须包含 `"messages"` 键。

**Q9. 让子 Agent 返回结构化 JSON 用什么字段？版本要求？**
A. `response_format`（Pydantic schema）；需 `deepagents>=0.5.3`。

**Q10. 本次实操中，我是怎么"修改子 Agent 职责提示词和工具映射"的？**
A. 定义 `note-writer` 子 Agent：`system_prompt` 写成"技术笔记整理专家 + 格式约束"；`tools=[]` 显式置空以完全替换主 Agent 工具集，实现最小权限。

---

## 学习日期
2026-10-22（理论整理） / 2026-10-23（补全实操与复习自测）