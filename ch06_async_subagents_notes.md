# 第 6 章学习笔记：异步子 Agent —— 让主 Agent 同时驱动多个子任务

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch06-async-subagents/
> 配套 Notebook：`notebooks/ch06/01-async-subagent-lifecycle.ipynb`（课程仓库已包含 offline 运行结果）
> 学习时间：2026-10-24

> ⚠️ 公开仓库安全提示：本文档不含任何明文 API Key / Token。所有密钥通过环境变量注入（如 `os.environ["MODEL_NAME"]`），真实密钥仅存于本地 `.env`（已 gitignore），切勿提交。

> ⚠️ 特性状态：`AsyncSubAgent` 是 `deepagents>=0.5.0` 的**预览特性（preview）**，API 仍在迭代，可能变化。

---

## 一、核心动机 / 概述

**问题**：第 5 章的同步子 Agent 用 `task()` 委派，主 Agent 必须**等到子 Agent 跑完**才能继续。这在两类场景下体验极差：

1. **长程任务**：深度调研、大规模代码迁移、批量数据处理，耗时从分钟级到小时级——主 Agent 在用户面前就是"死机状态"；
2. **可交互任务**：用户中途想补一句"换个数据源"、"加上 2024 年数据"，同步模式根本插不进去。

更糟的是：**被阻塞的是主 Agent 自己**，所以在子 Agent 完成前，用户连别的话题都没法聊。

**解决**：`AsyncSubAgent`——主 Agent 调 `start_async_task` 后**立即拿到 task ID 就返回**，子 Agent 在后台继续跑。用户可随时查进度、追加指令、甚至中途取消。核心收益两条：**不阻塞主线对话** + **支持中途控制**。

> 判定法则：**子任务 5 秒内能完成 → 用同步**；**可能跑数分钟以上、且过程需要交互 → 上异步**。

---

## 二、关键概念

| 概念 | 说明 |
|------|------|
| **`AsyncSubAgent`** | 一份"指向服务器上某个已注册 Agent"的**配置**，本身不创建实现、不启动服务 |
| **Agent Protocol** | 一套调用 Agent 的 API 规范：如何建会话、启动运行、查状态、取消任务 |
| **Agent Server** | 实现上述接口的运行服务，负责加载代码、调度后台执行、管理会话状态与结果 |
| **thread（会话）** | 保存消息与状态的一段会话——**不是操作系统线程**；子 Agent 有自己的 thread，历史持续累积 |
| **run（执行）** | 某个 thread 上的**一次**执行；同一个 thread 可以先后有多个 run |
| **`async_tasks` channel** | 主 Agent graph state 中**独立**于消息历史的通道，专存任务元数据 |
| **ASGI 传输** | 进程内适配器直接调用应用，省去网络收发（同部署默认） |
| **HTTP 传输** | SDK 走网络请求远程 Agent Protocol 服务（填 `url` 时启用） |
| **`AsyncSubAgentMiddleware`** | 声明 `AsyncSubAgent` 后自动注入 5 个异步工具的中间件 |

---

## 三、定义方式 / 字段说明

```python
from deepagents import AsyncSubAgent, create_deep_agent

graph = create_deep_agent(
    model=model,
    system_prompt="...派出异步子 Agent 后必须立刻把控制权交还给用户...",
    subagents=[
        AsyncSubAgent(
            name="researcher",
            description="深度研究 Agent，用于多次搜索 + 信息综合的调研任务",
            graph_id="researcher",
            # 不传 url → ASGI 进程内传输（与主 Agent 同部署）
        ),
    ],
)
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | 唯一标识；主 Agent 启动任务时用它指定子 Agent |
| `description` | ✅ | 能力描述；**主 Agent 靠它决定派给谁**（要行为导向、具体） |
| `graph_id` | ✅ | 服务端已注册的 graph / assistant ID；**必须与 `langgraph.json` 中的键名完全一致**（注意大小写、下划线） |
| `url` | 可选 | 不填 → 进程内 **ASGI**；填了 → 远程 **HTTP** |
| `headers` | 可选 | 远程模式下自托管服务的鉴权头 |

> `langgraph.json` 中把主图和子图**注册在一起**，才构成"同部署"：
> ```json
> { "dependencies": ["./"],
>   "graphs": { "supervisor": "./graphs/supervisor.py:graph",
>               "researcher": "./graphs/researcher.py:graph" },
>   "env": "./.env" }
> ```

---

## 四、主 Agent 的 5 把"遥控器"

声明 `AsyncSubAgent` 后，中间件自动注入 5 个工具：

| 工具 | 作用 | 返回 | 底层做了什么 |
|------|------|------|-------------|
| **`start_async_task`** | 启动后台任务 | task ID（**立即**） | 在服务端**新建 thread** + 启动一次 run，把描述作为输入；返回 thread ID 作为 task ID。**主 Agent 不轮询等待** |
| **`check_async_task`** | 查状态与结果 | 状态（+ 结果若完成） | 读 run 当前状态；已完成再读 thread state 取最终输出；在跑就说"运行中" |
| **`update_async_task`** | 追加新指令 | 确认 + 新状态 | 在**同一 thread** 上以 `interrupt` 策略发起**新 run**：前一次被打断，子 Agent 带完整历史 + 新指令重启。**task ID 不变，run ID 变** |
| **`cancel_async_task`** | 终止任务 | 确认信息 | 调 `runs.cancel()` 终止远程 run，并把本地任务标记为 `cancelled` |
| **`list_async_tasks`** | 列出所有任务 | 任务总览 | 已结束的从缓存返回；未结束的**并发**向服务端拉实时状态 |

### 一次完整生命周期

```
用户：帮我深入调研 LangGraph 多 Agent 架构。
主 Agent → start_async_task(...) → 返回 task_id: abc-123
主 Agent ← 「已派 researcher 在后台开干，ID：abc-123，你可以继续问别的。」

用户：先帮我把这段 Python 代码格式化一下。
主 Agent ← （直接处理，researcher 仍在后台跑）        ← 非阻塞区

用户：刚才那个调研有进展吗？
主 Agent → check_async_task("abc-123") ← status: running

用户：补一下：重点关注 supervisor / network / hierarchical 三种拓扑。
主 Agent → update_async_task("abc-123", "...") ← 已注入新指令

用户：算了，先停一下。
主 Agent → cancel_async_task("abc-123") ← cancelled
```

---

## 五、四个 ID 必须分清（Notebook 的核心观察点）

| 名称 | 含义 | 更新任务时 |
|------|------|-----------|
| `graph_id` | 工作流程的名称，如 `researcher` | 使用同一个图 |
| `thread_id` | 保存消息与状态的一段会话 | **保留**子会话 |
| `task_id` | 主 Agent 跟踪子任务的编号（本例等于子 thread_id） | **保持不变** |
| `run_id` | 这段会话上的某一次执行编号 | **换成新的 run** |

> 记忆口诀：**任务 T1 先有执行 R1，追加要求后仍是 T1，但执行变成 R2。**

实际运行时必须从返回状态里**取出生成的 ID**，不能硬编码上次输出的编号。

---

## 六、两层状态：主 Agent 记录 vs 服务端执行

同一个"状态"两个字，来源不同、含义不同：

| 字段来源 | 常见值 | 怎么理解 |
|---------|--------|----------|
| 主图 `state["async_tasks"][task_id]["status"]` | `running`、`success`、`cancelled` | **主 Agent 对这项任务的跟踪记录** |
| 服务端 `runs.get(...)["status"]` | `pending`、`running`、`success`、`interrupted` | **具体某次 run 的执行状态** |

主 Agent 标 `running` 时服务端可能还在 `pending`（等待调度）；取消后主记录可为 `cancelled`，服务端终态可为 `interrupted`。**它们描述不同层次，不要求字符串完全相同**——所以取消操作要查两边确认。

---

## 七、任务元数据为何要单开一个 channel？

`async_tasks` 通道独立于消息历史，存放：task ID、子 Agent 名、thread ID、run ID、状态、`created_at`、`last_checked_at`、`last_updated_at`。

**原因**：Deep Agents 在上下文逼近上限时会**自动压缩对话历史**（见第 3 章）。如果 task ID 只活在某条 ToolMessage 里，压缩后就丢了——主 Agent 会瞬间"忘了"自己派出去的所有任务，再也无法 check 或 cancel。

> 设计哲学（与虚拟文件系统、TodoList 一脉相承）：**会被截断的放消息历史，必须长存的进 state channel**。

---

## 八、两种传输 & 三种部署拓扑

```
HTTP 传输：SDK → 网络连接 → Uvicorn → Agent Server 应用
ASGI 传输：SDK → 进程内 ASGI 适配器 → Agent Server 应用
```

> **调用方式提醒**：ASGI 传输需要主 Agent 通过 `ainvoke()` 等**异步入口**执行，不支持同步 `invoke()`。

| 拓扑 | 形态 | 推荐场景 |
|------|------|---------|
| **单部署（Single）** | 所有 Agent 同部署，全部走 ASGI | **绝大多数项目的起点**：好运维、零网络延迟 |
| **拆分部署（Split）** | 主一台、子一台，全走 HTTP | 子 Agent 资源画像/扩缩容策略与主 Agent 显著不同 |
| **混合（Hybrid）** | 部分 ASGI（同部署）+ 部分 HTTP（远程） | 多数同部署省事，少数特殊子 Agent 单独扩 |

**起手式**：先用单部署 + ASGI，遇到具体的扩缩容或团队边界问题再拆。

---

## 小结

1. 异步子 Agent 解决同步委派的两个瓶颈：**主 Agent 不被阻塞** + **任务可中途追加/取消**。
2. 5 把遥控器：`start` / `check` / `update` / `cancel` / `list`，由 `AsyncSubAgentMiddleware` 自动注入。
3. 四个 ID 分清：**task_id 不变、run_id 会变**（update 时）、thread 是会话、graph_id 是流程名。
4. 状态分两层：主 Agent 跟踪记录 vs 服务端 run 执行状态，取值不必相同；取消要查两边。
5. 元数据存**独立的 `async_tasks` channel**，避免上下文压缩后丢失任务 ID。
6. 传输默认 ASGI（同部署、零延迟、需 `ainvoke()`）；填 `url` 切 HTTP（远程、可独立扩缩容）。
7. `graph_id` 必须与 `langgraph.json` 注册名**完全一致**，否则"找不到 Agent"。
8. Worker 槽位要够：1 主 + N 子，至少 N+1；不够会排队，表现为 `start` 迟迟不返回。

---

## 我的实操记录

> **状态：本章尚未在本机跑通（理论已学习完）。**
> 原因：本章不像前几章可以在 Python 进程内 `agent.invoke()`，它**必须运行在支持 Agent Protocol 的 LangGraph 服务环境中**（`langgraph dev` 启动本地 Agent Server）。课程 Notebook 自带 `LocalAgentServer` 辅助类来自行启动与清理服务。
>
> 环境：Windows 原生 PowerShell + 项目 venv（`.venv`）；模型通过环境变量注入（`os.environ["MODEL_NAME"]` 等），真实密钥存于本地 `.env`。

### 课程 Notebook 给出的 offline 运行结果（作为预期判据）

课程让 `researcher` **故意 `await asyncio.sleep(8)`**，从而留出观察窗口。以下为课程 offline 模式实测输出（**注：这是课程仓库的运行结果，不是本机产出**）：

```
阶段一：启动 → 列举 → 完成
  start_async_task → Launched async subagent. task_id: 01a12f55-...-777dd9a01d8c
  主 Agent 已返回，后台 run 仍为： pending        ← 关键证据：主 Agent 返回时子任务未完成
  list_async_tasks → 1 tracked task(s): ... status: running
  check_async_task → status: running → status: success
                     result: Research completed: 总结异步任务的状态变化

阶段二：更新 → 新 run → 读取新结果
  update_async_task → Updated async subagent. task_id: 01a12f55-...-0b44071f0391
  已验证：task ID 不变、run ID 更新、追加指令传递成功。
  result: Research completed: 追加要求：使用三条要点    ← 追加指令真的进了子图

阶段三：取消 → 服务端确认
  cancel_async_task → Cancelled async subagent task: 01a12f56-...-c7863648d6bc
  list_async_tasks → 3 tracked task(s): success / success / cancelled

生命周期验证结果：
  完成 → success
  更新 → success
  取消 → interrupted
```

### 若在本机（Windows）跑通，需要的关键改造

**① `graphs/supervisor.py` 的 system_prompt 要"命令协议化"**（课程做法，便于离线脚本模型稳定触发）

```python
system_prompt=(
    "START|description -> start_async_task(subagent_type='researcher', description=description).\n"
    "CHECK|task_id -> check_async_task(task_id=task_id).\n"
    "LIST -> list_async_tasks(status_filter='all').\n"
    "UPDATE|task_id|message -> update_async_task(task_id=task_id, message=message).\n"
    "CANCEL|task_id -> cancel_async_task(task_id=task_id).\n"
    "Never invent a task ID or report a cached status as live."
)
```
最后一句很关键：防止模型**编造 task ID**或**把旧状态当实时状态**汇报。

**② `graphs/researcher.py` 故意放慢**，让异步现象可观察
```python
async def research(state):
    await asyncio.sleep(8)     # 给 check / update / cancel 留出观察窗口
    return {"messages": [{"role": "ai", "content": f"Research completed: {request}"}]}
```

**③ 用 `async with LocalAgentServer(...)` 管生命周期**
进入时启动服务并创建主会话；正常结束或抛异常都会清理（先取消未完成任务并删除会话，再等服务退出）。**三个阶段必须在 `async with` 内部执行**，退出后服务已关闭。

> Windows 注意：命令行启动器会再起 Python 子进程，必须等子进程也退出才能释放日志与工作目录；正常结束会删除本次日志，异常退出**保留**日志供排查。

### 环境差异提醒（Windows vs 课程默认）

- 课程用 `uv sync --project notebooks --extra server` 装依赖、**用 uv 管理环境**；本机是 `.venv` + pip，需自行补装 `langgraph-cli[inmem]`、`langgraph-sdk`。
- 课程环境打印 `Python 3.12.3 / Linux x86_64 / deepagents==0.7.22`；本机 `deepagents` 版本可能不同，异步 API 属 preview，**需先核对版本**。
- 服务只绑定本机地址（`http://127.0.0.1:2024`），临时服务状态不写入仓库。

---

## 如何验证

| # | 检查项 | 看到什么算成功 |
|---|--------|---------------|
| 1 | **启动即返回** | `start_async_task` 立刻返回 task ID；用 SDK 直查服务端，run 仍是 `pending` 或 `running` → 证明"主 Agent 已返回、子任务未完成" |
| 2 | **能列举** | `list_async_tasks` 的输出里包含刚拿到的 task ID |
| 3 | **轮询到成功** | `check_async_task` 状态 `running → success`，且结果含 `Research completed: ...`；服务端 run 终态 `success` |
| 4 | **更新保留 task、换 run** | `update_async_task` 后 **task ID 不变、run ID 改变**；且追加的 `message` 出现在子图返回文本里（不只是一句"已更新"） |
| 5 | **取消两边确认** | 主 Agent 记为 `cancelled`；轮询服务端直到 run 进入 `interrupted` / `cancelled` |
| 6 | **清理** | 结束时打印"任务、thread、服务进程与临时状态已清理" |

### 常见故障速查

| 现象 | 优先检查 | 处理 |
|------|---------|------|
| 定义阶段没有任何输出 | `async def` 只是定义，未执行 | 正常，继续跑到最后执行节 |
| `start_async_task` 报找不到 Agent | `graph_id` 与 `langgraph.json` 注册名是否一致（大小写/下划线） | 改成完全一致 |
| 启动卡住、`start` 迟迟不返回 | worker 槽位被打满 | `langgraph dev --n-jobs-per-worker 10` |
| 任务一直 `running` | worker 数 / 外部工具无超时 | 提高 worker；给外部 API 设超时 |
| `cancel` 后状态不立刻变 | 服务端取消是异步生效 | cancel 后再 `check` / `list` 确认终态 |
| 主 Agent 查不到旧任务 | 是否配了 checkpointer | 配 checkpointer；元数据依赖 `async_tasks` channel |
| 主 Agent 刚启动就疯狂轮询 | 模型把异步退化成伪同步 | 在 `system_prompt` 强化"派完立刻交还控制权，用户不问就别 check" |
| 模型报告过时状态 | 直接引用历史里的旧状态 | `system_prompt` 加"回答进度前必须先 `check_async_task` / `list_async_tasks`" |
| task ID 被截断/改写 | 模型缩写 UUID | `system_prompt` 加"始终使用完整 task_id"；或换更听话的模型 |

---

## 回忆复习 · 自测题（含答案）

**Q1.** 同步子 Agent 的两个瓶颈是什么？异步如何解决？
A. ① 主 Agent 被阻塞（长程任务"死机"）；② 无法中途交互。异步：立即返回 task ID 不阻塞 + 支持 update/cancel。

**Q2.** 什么时候该用同步，什么时候该上异步？
A. 5 秒内完成 → 同步；数分钟以上且过程需交互 → 异步。

**Q3.** `AsyncSubAgent` 的必填字段有哪些？`url` 不填意味着什么？
A. 必填 `name` / `description` / `graph_id`；`url` 不填 → 走**进程内 ASGI**（同部署）。

**Q4.** `update_async_task` 之后，task ID 和 run ID 分别怎么变？
A. **task ID 不变**（同一 thread），**run ID 变**（新的一次执行）。

**Q5.** 主 Agent 的 `async_tasks[...]["status"]` 与服务端 `runs.get(...)["status"]` 取值必须相同吗？
A. 不必。一个是主 Agent 的跟踪记录（`running`/`success`/`cancelled`），一个是某次 run 的执行状态（`pending`/`running`/`success`/`interrupted`），描述不同层次。

**Q6.** 为什么任务元数据要放进独立的 `async_tasks` channel？
A. 上下文逼近上限时消息历史会被自动压缩；若 task ID 只存在 ToolMessage 里会被压掉，导致主 Agent 忘记派过的任务。

**Q7.** ASGI 传输对调用方式有什么限制？
A. 必须用 `ainvoke()` 等**异步入口**，不支持同步 `invoke()`。

**Q8.** `graph_id` 写错的典型症状是什么？
A. `start_async_task` 报"找不到 Agent"——需与 `langgraph.json` 的 `graphs` 键名完全一致（含大小写、下划线）。

**Q9.** worker 槽位不够会出现什么现象？
A. `start_async_task` 长时间不返回，或拿到 ID 后 `check` 长时间无实质进展（任务在排队）。1 主 + N 子至少需要 N+1 个槽位。

**Q10.** 怎么防止模型"编造 task ID"或"把缓存状态当实时状态"？
A. 在 `system_prompt` 明确要求：`Never invent a task ID or report a cached status as live.`；并要求回答进度前必须先调 `check_async_task` / `list_async_tasks`。

---

## 学习日期
2026-10-24（理论） / 实操待进行（需启动本地 Agent Server）
