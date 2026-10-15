# AgentSeek 调试复盘笔记（两次经历）

> 结构：验证什么 → 改了什么 → 实际结果 → 证据与解释 → 下一步修正

---

## 经历一：跑 content_builder_deepagent 全栈项目（晚）

### 项目定位
- 路径：`D:\ai_tools\deepagents-in-action\content_builder_deepagent`
- 架构：AgentSeek API（后端 FastAPI）+ Vite/React（前端）+ OceanBase（向量存储）
- 启动方式：lifecycle runner 同时拉起两个进程

### 问题 1：后端启动崩溃 — Can't connect to MySQL server on 'localhost'

**验证什么**：确定 OceanBase 嵌入模式为何在 Windows 失效。

**改了什么**：在 `.env` 加：
```env
SEEKDB_EMBED=false
METADATA_DB_URL=sqlite+aiosqlite:///./agentseek.db
METADATA_DB_BACKEND=sqlite
```

**实际结果**：后端启动成功，`/health` 返回 `{"status":"healthy"}`。

**证据与解释**：
- `lifecycle.toml` 默认 `SEEKDB_EMBED=true`，走内嵌 SeekDB 分支
- 内嵌分支调用 `pyseekdb` → 依赖 `pylibseekdb`
- 源码 `pyseekdb/client/client_seekdb_embedded.py` 注释明确写：
  ```python
  """Note: Only available when pylibseekdb is installed (Linux only)"""
  ```
- Windows AMD64 上 `import pylibseekdb` → `ModuleNotFoundError`
- 内嵌失败后 fallback 到连独立 OceanBase（`localhost:2881`），但本机没起 → 报错

```python
# agentseek_api/core/database.py initialize()
if embed_mode:
    await asyncio.to_thread(_ensure_embed_database, embed_dir, db_name)  # Windows 崩在这
else:
    metadata_db_url = resolve_metadata_db_url()  # SQLite 走这里
```

**下一步修正**：装 Docker Desktop 起一个真的 OceanBase 容器，把这三行删掉即切回正式架构（带向量检索）。

---

### 问题 2：前端 5174 拒绝连接

**验证什么**：前端 Vite 是否在监听 5174。

```bash
netstat -ano | grep -E ":(5174|2024)" | grep LISTENING
# 5174 无进程  2024 在跑
```

**改了什么**：在 `frontend/` 下手动 `npm run dev`。

**实际结果**：5174 返回 200。

**证据与解释**：用户重启时只起了后端，前端进程没拉起。lifecycle runner 一次会拉起两个，单独跑 `uv run` 不会起前端。

---

### 问题 3：发消息报 OpenAIConnectionError → 401 Token is invalid

**验证什么**：确定模型 API 地址与 key 归属平台是否匹配。

**改了什么**：
- 第一阶段：加 `OPENAI_API_BASE=https://api.siliconflow.cn/v1`
- 第二阶段：换 key 后发现是 DeepSeek 官方的 key，改成：
```env
OPENAI_API_KEY=sk-0823...
OPENAI_API_BASE=https://api.deepseek.com/v1
AGENTSEEK_MODEL=deepseek-chat
```

**实际结果**：AI 回复完整内容。

**证据与解释**：
- 用户第一次换的 key 在硅基流动返回 `{"code":30014,"message":"Token is invalid."}`
- DeepSeek 官方 key 格式为 `sk-` + 32 位 hex（长度 35）
- 硅基流动 key 通常 51 字符以上
- 走错平台必然返回 401，key 本身可能没抄错

**下一步修正**：如果要与教程一致，回到硅基流动注册账号拿正确的 key（今天做了）。

---

### 问题 4：AI 回复正常，偶发 Unable to connect to LangGraph server

**验证什么**：为何 AI 已成功回复后还会弹 Connection error。

**改了什么**：改 `.agentseek/lifecycle.toml`：
```toml
[processes.langgraph]
command = ["uv", "run", "agentseek-api", "dev", "--port", "2024", "--no-reload"]
```

**实际结果**：重启后不再复现该报错。

**证据与解释**：
- `agentseek-api dev` 默认开启 uvicorn `--reload`
- agent 执行时会向项目目录写工作文件（`planning.md` 等）
- uvicorn 监测到文件变化自动重启服务，掐断进行中的连接

---

## 经历二：跟教程走 Chapter 2 纯脚本（次日晚上）

### 目标定位
教程 Chapter 2 只装 `deepagents + langchain-openai` 包，直接 `python xxx.py` 运行。
不涉及 AgentSeek API、不涉及数据库、不涉及前端界面。

### 关键代码（hello_world.py）

```python
import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent

# 通过硅基流动接入模型（兼容 OpenAI 接口）
model = ChatOpenAI(
    model=os.environ.get("MODEL_NAME", "Qwen/Qwen2.5-7B-Instruct"),
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

### 报错：PowerShell 不认识 `set -a; source .env`

**验证什么**：用户用的什么 shell。

**证据**：报错截图显示 `Set-Variable: A parameter cannot be found that matches parameter name 'a'` —— 这是 PowerShell 错误。教程里的 `set -a; source .env; set +a` 是 bash/WSL 语法。

**修正方案**：
- 方案一：PowerShell 用 `Get-Content .env | ForEach-Object { if ($_ -match '^\s*([A-Za-z_][A-Za-z0-9_]*)=(.*)$') { $value = $matches[2] -replace '^"(.*)"$', '$1'; Set-Item -Path "Env:$($matches[1])" -Value $value } }`
- 方案二：在脚本里用 `python-dotenv` 自动加载，运行直接 `python hello_world.py`

---

## 输出文件清单

| 文件 | 位置 | 说明 |
|---|---|---|
| `agentseek-debug-notes.md` | GitHub repo 根 | 第一次复盘（按问题时间线） |
| `agentseek-debug-notes-v2.md` | GitHub repo 根 | 本文件，按五段式结构重写 |

## 工具调用状态 Trace

| 操作 | 工具 | 结果 |
|---|---|---|
| 验证 OceanBase 内嵌支持 | `Read pyseekdb/client/client_seekdb_embedded.py` | 确认 Linux only |
| 验证配置生效 | 启动后端 + `curl /health` | 返回 healthy |
| 验证 key 归属 | `curl` 调硅基流动 + DeepSeek | 确认是 DeepSeek 官方 |
| 验证模型可调 | `curl chat/completions` | 返回正常回复 |
| GitHub push | REST API PUT contents | 200 + 返回 html_url |
| 验证 token | `curl /user` | Bad credentials（已 revoke） |

## 经验提炼

1. **报错先定位根因再改** — OceanBase 连不上不是 OB 的问题，是 Windows 不支持内嵌
2. **环境差异是头号坑** — `pylibseekdb` Linux only，PowerShell 不认 bash 语法
3. **教程分层** — Chapter 2 是纯 SDK 脚本，`content_builder_deepagent` 是 AgentSeek API 全栈项目，必须区分
4. **Token 平台要对齐** — 不同 LLM 平台 key 格式不同，接口地址必须匹配
5. **dev 模式热重载的副作用** — agent 写文件到项目目录会触发服务重启