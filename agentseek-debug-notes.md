# AgentSeek content_builder 调试学习心得

> 记录从启动报错到全链路跑通的过程，以及在 Windows 环境下踩过的坑。

## 项目背景

- **项目**: `content_builder_deepagent`（来自 deepagents-in-action）
- **技术栈**: LangGraph + AgentSeek API（后端）+ Vite/React（前端）+ OceanBase（向量存储）
- **启动方式**: lifecycle runner 同时拉起前后端

```
Langgraph: uv run agentseek-api dev --port 2024
Frontend:  npm run dev
前端: http://127.0.0.1:5174
后端: http://127.0.0.1:2024
```

## 报错全景

从启动到跑通，先后遇到 4 个问题，逐层往下排查。

### 问题 1：后端启动直接崩溃 — Can't connect to MySQL server on 'localhost'

**错误**:
```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server on 'localhost'")
sqlalchemy.exc.OperationalError: ... Application startup failed.
```

**排查**:
- `lifecycle.toml` 默认 `SEEKDB_EMBED=true`，想用内嵌 SeekDB（免装服务器）
- 但内嵌模式依赖 `pylibseekdb`，该库**仅支持 Linux**
- Windows AMD64 上 `import pylibseekdb` 直接 `ModuleNotFoundError`
- 内嵌失败后 fallback 到连远程 OceanBase（`localhost:2881`），但本机没有 OB 服务在跑 → 连接失败

**关键代码** (`agentseek_api/core/database.py`):
```python
if embed_mode:
    # 内嵌分支: 调 pyseekdb → pylibseekdb (Linux only)
    await asyncio.to_thread(_ensure_embed_database, embed_dir, db_name)
    ...
else:
    # 非内嵌: 连独立 OceanBase / SQLite
    metadata_db_url = resolve_metadata_db_url()
    ...
```

**解决方案（方案 B — SQLite 降级）**:

在 `.env` 中加三行：
```env
SEEKDB_EMBED=false
METADATA_DB_URL=sqlite+aiosqlite:///./agentseek.db
METADATA_DB_BACKEND=sqlite
```

这样走非内嵌分支的 SQLite 路径：`SqliteCheckpointSaver` + `SqliteStore` + `InMemorySaver`。

**代价**:
- LangGraph 运行检查点用 `InMemorySaver`（**重启即丢**）
- Store 退化为 SQLite，**失去 OceanBase 向量检索**
- content_builder 深度依赖长期记忆/语义检索，B 只适合临时调试

**正经方案（方案 A — Docker OceanBase）**:
```bash
docker run --name agentseek-ob -p 2881:2881 -e MODE=mysql -d oceanbase/oceanbase-ce
docker exec -it agentseek-ob obclient -h127.0.0.1 -P2881 -uroot@test -e "CREATE DATABASE IF NOT EXISTS seekdb"
```
镜像默认 `root@test` 空 密码，和 app 默认 `OCEANBASE_*` 对齐。方案 B 跑通后切 A，
只需删掉 `.env` 里那三行重启即可，两套存储完全独立，互不影响。

---

### 问题 2：前端页面打不开 — 127.0.0.1 拒绝了连接请求

**错误**: 浏览器输入地址后提示 "127.0.0.1 拒绝了连接请求"

**排查**:
```
netstat -ano | grep -E ":(5174|2024)" | grep LISTENING
```
- 后端 2024 在跑 ✅
- 前端 5174 没有进程监听 ❌
- 用户重启时只起了后端，前端进程没拉起来
- 另外浏览器地址没带端口

**解决**: 起前端 Vite 服务，并提醒用户地址必须带 `:5174`。

---

### 问题 3：发消息后报 OpenAIConnectionError → 401 Token is invalid

**错误演变**:
1. 第一阶段: `Connection error`（连不上模型 API）
2. 补 `OPENAI_API_BASE` 后: `401 Token is invalid`（连上了但 key 无效）

**排查**:
- 项目默认模型是 `deepseek-ai/DeepSeek-V3.2`（硅基流动的模型名）
- README 指定 `OPENAI_API_BASE=https://api.siliconflow.cn/v1`
- 用户 `.env` 缺 `OPENAI_API_BASE`，SDK 默认连 `api.openai.com`，国内不通 → Connection error
- 补上硅基流动地址后，用户的 key 在硅基流动返回 `Token is invalid`
- 用户换了一个新 key `sk-0823...`，**仍然是 `Token is invalid`**
- 实测发现这个 key 走 **DeepSeek 官方接口** `api.deepseek.com` 是有效的

**根因**: 用户的 key 是 DeepSeek 官方建的，不是硅基流动的。

**最终配置** (`.env`):
```env
OPENAI_API_KEY=sk-0823...
OPENAI_API_BASE=https://api.deepseek.com/v1
AGENTSEEK_MODEL=deepseek-chat
```

**关键代码** (`src/content_builder_deepagent/agent.py`):
```python
if MODEL_PROVIDER == "openai":
    if _nonempty_env("OPENAI_API_BASE"):
        PROVIDER_KWARGS["base_url"] = _nonempty_env("OPENAI_API_BASE")
```

**教训**: 看到 `Token is invalid` 不要先入为主认为是 key 抄错了，
**先确认 key 属于哪个平台**（DeepSeek key 是 `sk-` + 32位 hex，长度 35；
硅基流动 key 通常 51+ 字符）。

---

### 问题 4：AI 回复正常，但偶发 "Unable to connect to LangGraph server"

**现象**: 前端收到完整回复后，底部弹 `ConnectionError: Unable to connect to LangGraph server`。

**排查**:
- 后端 `/health` 仍返回 healthy
- `agentseek-api dev` 默认开启 uvicorn `--reload`（热重载）
- content_builder 执行时会往项目目录写工作文件（`planning.md`、`files.md` 等）
- uvicorn 监测到文件变化 → 自动重启服务 → 掐断进行中的连接 → 前端报错

**修复**: 在 `.agentseek/lifecycle.toml` 里给 langgraph 启动命令加 `--no-reload`：
```toml
[processes.langgraph]
command = ["uv", "run", "agentseek-api", "dev", "--port", "2024", "--no-reload"]
```

---

## 最终 .env 配置

```env
OPENAI_API_KEY=sk-0823...
OPENAI_API_BASE=https://api.deepseek.com/v1
AGENTSEEK_MODEL=deepseek-chat
SEEKDB_EMBED=false
METADATA_DB_URL=sqlite+aiosqlite:///./agentseek.db
METADATA_DB_BACKEND=sqlite
```

## 关键收获

1. **先定位再改，不要试错** — 每个错误都先看错误栈定位根因，修就一次性修对
2. **环境差异坑** — `pylibseekdb` 仅 Linux，Windows 跑必须降级或上 Docker
3. **key 平台要对齐** — DeepSeek 官方 key 和硅基流动 key 不能混用，接口地址和模型名都要匹配
4. **dev 模式的副作用** — 热重载对 agent 输出文件到项目目录的场景是灾难性的

## 技术栈速览

| 组件 | 作用 |
|---|---|
| LangGraph | Agent 工作流编排 |
| AgentSeek API | LangGraph 的 API 服务封装 |
| OceanBase | 向量数据库 + checkpointer + store |
| pyseekdb/pylibseekdb | OceanBase 内嵌模式（Linux only）|
| DeepSeek API | LLM 模型（deepseek-chat）|
| Vite + React | 前端界面 |