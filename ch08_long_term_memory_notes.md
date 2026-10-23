# 第 8 章学习笔记：长期记忆 —— 跨对话记住用户

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch08-long-term-memory/
> 配套 Notebook：课程仓库 `notebooks/ch08/`（本次实操以本地脚本 `memory_preferences.py` / `memory_preferences_offline.py` 完成）
> 学习时间：2026-10-24

> ⚠️ 公开仓库安全提示：本文档不含任何明文 API Key / Token。所有密钥通过环境变量注入（如 `os.environ["DEEPSEEK_API_KEY"]`），真实密钥仅存于本地 `.env`（已 gitignore），切勿提交。

---

## 一、核心动机 / 概述

**问题**：Agent 的对话记忆默认由 **Checkpointer** 提供，而它只在**同一个 `thread_id`** 内有效。用户换一个新会话（新 `thread_id`）问同样的问题，Agent 就完全失忆——上次告诉它的"代码注释用中文"全部作废。更糟的是，每次都重新交代偏好，既费 Token 又体验差。

**解决**：引入**长期记忆**——用一个独立的持久化存储 **Store** 保存跨会话信息，并通过 **`CompositeBackend`** 把特定路径（如 `/memories/`）的文件路由到 Store。对 Agent 来说，操作方式没变（还是 `read_file` / `edit_file`），**区别只在路径前缀**：写在 `/memories/` 下的文件跨会话存活，写在别处的文件随线程消亡。启动时再由 `memory=[...]` 把这些文件注入系统提示词，Agent 一开机就"带着记忆"。

---

## 二、关键概念

| 概念 | 说明 |
|------|------|
| **短期记忆** | Checkpointer（如 `MemorySaver`），仅在同一 `thread_id` 内有效，换线程即失忆 |
| **长期记忆** | Store（`InMemoryStore` 开发 / `PostgresStore` 生产）+ 路径路由，跨线程、跨会话持久 |
| **Store** | 键值持久化存储，按 **namespace（命名空间）** 组织，是长期记忆的物理载体 |
| **CompositeBackend** | 组合后端：按路径前缀把文件分发到不同后端（`/memories/` → Store，其余 → State） |
| **StoreBackend** | 把文件读写落进 Store 的后端实现，需要指定 namespace |
| **StateBackend** | 默认后端，文件只活在当前线程状态里 |
| **命名空间（namespace）** | 形如 `(user_id, "memories")` 的元组，决定记忆的作用域与隔离边界 |
| **`memory=` 参数** | `create_deep_agent(memory=["/memories/xxx.md"])`，指定**启动时加载哪些记忆文件**（读取配置） |
| **`create_file_data()`** | 把字符串包装成 Store 能存的文件对象，用于预置初始记忆 |

---

## 三、定义方式 / 字段说明

最小可运行骨架：

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.memory import InMemoryStore
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from deepagents.backends.utils import create_file_data

NAMESPACE = ("user-123", "memories")          # 用户级隔离
MEMORY_PATH = "/memories/preferences.md"

store = InMemoryStore()
# ⚠️ CompositeBackend 会去掉路由前缀，所以 Store key 是 "/preferences.md"
store.put(NAMESPACE, "/preferences.md", create_file_data("# 用户偏好\n暂无记录。"))

backend = CompositeBackend(
    default=StateBackend(),                                    # 其它路径：线程内
    routes={"/memories/": StoreBackend(namespace=lambda _rt: NAMESPACE)},  # 记忆路径：持久化
)

agent = create_deep_agent(
    model=model,
    store=store,                       # 长期记忆的存储载体
    checkpointer=MemorySaver(),        # 短期记忆（对话状态）
    backend=backend,                   # 路径路由规则
    memory=[MEMORY_PATH],              # 启动时自动加载（只读配置）
    system_prompt="当用户要求记住偏好时，先读取该文件，再用 edit_file 更新它……",
)
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `store` | 是（用长期记忆时） | Store 实例；不传则 `/memories/` 无处可落 |
| `backend` | 是 | `CompositeBackend`，决定哪些路径进 Store |
| `memory` | 否（但要跨会话生效必须给） | 字符串列表，**启动时读取**并注入系统提示词 |
| `checkpointer` | 是 | 负责当前会话的多轮状态 |
| `system_prompt` | 建议必填 | **写入约定**在这里定：何时写、写到哪、用什么工具写 |

---

## 四、长期记忆三步流程

| 步骤 | 做什么 | 关键 API |
|------|--------|----------|
| ① 预置存储 | 建 Store，并给记忆文件一个初始内容（否则 Agent 读到空） | `store.put(ns, key, create_file_data(...))` |
| ② 加载记忆 | 启动时把记忆文件注入系统提示词 | `memory=["/memories/preferences.md"]` |
| ③ 写入记忆 | 在 `system_prompt` 中约定写入规则，Agent 用 **`edit_file`** 更新 | `edit_file(file_path, old_string, new_string)` |

**为什么必须用 `edit_file` 而不是 `write_file`**：`write_file` 是**整体覆盖**，会把已有偏好全部抹掉；`edit_file` 是**局部替换**（`old_string` → `new_string`），能保留其他条目。这是本章最容易踩的坑。

---

## 五、记忆的作用域（namespace 怎么定）

| 级别 | namespace 写法 | 效果 | 适用 |
|------|---------------|------|------|
| **Agent 级** | `(assistant_id,)` | 所有用户共享同一份记忆 | 公共知识、产品文档 |
| **用户级** | `(user_id, "memories")` | 每个用户独立，互不干扰 | **默认推荐**：个人偏好 |
| **组织级** | `(org_id,)` | 全组织共享，通常设为只读 | 团队规范、术语表 |

> 默认用**用户级**。若把个人偏好放在 Agent 级，会 A 用户的偏好泄露给 B 用户，且构成提示注入风险。

---

## 六、两个高频坑（重点）

1. **`memory=` 只是"读取"配置，不保证 Agent 往里写。**
   写入路径与写入方式必须在 `system_prompt` 里**明确约定**，否则 Agent 只会嘴上说"已记住"，Store 里却空空如也。
   另外 0.7.10 版本中，`memory` 里列出的文件若不存在会被**静默跳过**，不会自动创建 —— 所以要先 `store.put` 预置。

2. **`CompositeBackend` 会去掉路由前缀作为 Store key。**
   路由是 `/memories/`，但写进 Store 的 key 是 `/preferences.md`（**没有** `/memories`）。手动预置或核查时必须用去掉前缀的 key，否则读出来是 `None`。

---

## 小结

1. 短期记忆靠 Checkpointer（线程内），长期记忆靠 Store + 路径路由（跨会话）。
2. `CompositeBackend` 用**路径前缀**分流：`/memories/` 进 Store 持久化，其余留在 State。
3. 写入一律用 **`edit_file`**（局部替换），`write_file` 会整体覆盖。
4. `memory=` 是读取配置；**写入行为靠 `system_prompt` 约定**。
5. Store key 是**去掉 `/memories/` 前缀**后的路径。
6. 默认用**用户级** namespace，避免跨用户泄露与提示注入。
7. `InMemoryStore` 仅在当前进程有效，重跑即清空；生产要换 `PostgresStore`。

---

## 我的实操记录

> 目标：让 Agent 记住"代码注释用中文，变量名用英文"，并在**全新会话**中自动应用。
> 环境：Windows 原生 PowerShell + 项目 venv；模型通过 `os.environ["DEEPSEEK_API_KEY"]` 接入（`load_dotenv(override=True)` 从 `.env` 读取）。

### 改造点

**① 用户级命名空间 + 预置文件**（对应坑 2）

```python
NAMESPACE = ("user-123", "memories")
store.put(NAMESPACE, "/preferences.md", create_file_data("# 用户偏好\n暂无记录。"))
#                    ↑ 注意：没有 /memories 前缀
```

**② 写入约定写进 system_prompt**（对应坑 1）

```python
system_prompt=(
    f"当用户明确要求记住偏好时，先读取 {MEMORY_PATH}，再用 edit_file 更新它。"
    "逐字保存用户给出的偏好条目，保留其他已有偏好，不要另建偏好文件。"
    "只有工具写入成功后，才告知用户已记住。"
)
```
最后一句是为了防止 Agent "口头承诺但未落盘"。

**③ 核查 Store 实际内容，而不是相信模型的自述**

```python
saved = store.get(NAMESPACE, "/preferences.md")
content = saved.value["content"]
ok = ("代码注释用中文" in content) and ("变量名用英文" in content)
```

### 实际运行结果（已跑通 ✅）

```
对话 1（thread A）：记住偏好
Agent 回复： 已记住你的偏好：代码注释用中文，变量名用英文。之后写代码时我会按这个来。
Store 中实际保存的偏好：
# 用户偏好
- 代码注释用中文，变量名用英文
✅ 偏好已按约定写入

对话 2（thread B，全新对话）：验证跨对话自动加载
Agent 回复：
根据你的偏好（注释用中文、变量名用英文），代码示例如下：
def bubble_sort(arr):
    ...  # 中文注释、英文变量名
```

### 环境踩坑记录（Windows 特有）

| 现象 | 根因 | 修法 |
|------|------|------|
| `KeyError: 'DEEPSEEK_API_KEY'` | 环境变量未设置（**还没联网**） | 写进 `.env`（持久）或 `$env:DEEPSEEK_API_KEY="..."`（窗口级） |
| PowerShell 报"无法识别为 cmdlet" | 用了 bash 语法 `VAR=value` | PowerShell 必须写 `$env:VAR="value"` |
| 换完 Key 仍 401 | `load_dotenv()` 默认 `override=False`，旧会话变量压过新值 | 改成 `load_dotenv(override=True)` |
| 每轮刷 `403 Forbidden`（LangSmith） | 关闭遥测的语句写在 `load_dotenv(override=True)` **之前**被覆盖 | 把 `LANGSMITH_TRACING="false"` 移到 `load_dotenv()` 之后 |
| `401 Authentication Fails` | 密钥本身被平台判定无效 | 去平台重新生成 Key 替换 |
| `402 Insufficient Balance` | 密钥有效但余额不足 | 充值（不是换 Key） |

> **通用教训**：使用 `load_dotenv(override=True)` 时，任何想强制生效的 `os.environ[...]` 赋值都必须放在它**之后**。

---

## 如何验证

三条判定证据，全部满足才算跨对话长期记忆成立：

| # | 检查项 | 看到什么算成功 |
|---|--------|---------------|
| 1 | **Store 实际内容** | 打印出偏好文本，并出现 `✅ 偏好已按约定写入`（不是只听模型说"已记住"） |
| 2 | **写入方式** | 原有标题 `# 用户偏好` 仍在 → 说明用的是 `edit_file` 局部追加，非 `write_file` 覆盖 |
| 3 | **全新 thread_id** | 对话 2 用 `uuid.uuid4()` 生成全新线程，回复仍引用偏好且代码是**中文注释 + 英文变量名** |

想看系统提示词里记忆长什么样（离线版支持）：
```powershell
$env:DEBUG_MEMORY="1"
.\.venv\Scripts\python.exe memory_preferences_offline.py
```
框架会把它包在 `<agent_memory>` 标签里注入：
```
<agent_memory>
/memories/preferences.md
# 用户偏好
- 代码注释用中文，变量名用英文
</agent_memory>
```

> 另有一个**离线版** `memory_preferences_offline.py`（脚本化模型，无需 API Key、零 Token），可在没有可用模型时复现同一链路。

---

## 回忆复习 · 自测题（含答案）

**Q1.** 短期记忆和长期记忆分别由什么组件承载？
A. 短期 = Checkpointer（线程内）；长期 = Store + `CompositeBackend` 路径路由（跨会话）。

**Q2.** Agent 操作 `/memories/preferences.md` 和普通文件，代码上有区别吗？
A. 没有。对 Agent 完全一致，区别只在文件系统**路径前缀**决定它落到哪个后端。

**Q3.** 为什么写入记忆要用 `edit_file` 而不是 `write_file`？
A. `write_file` 整体覆盖会抹掉已有偏好；`edit_file` 是 `old_string → new_string` 局部替换，能保留其他条目。

**Q4.** `memory=["/memories/preferences.md"]` 这句话做了什么？它保证 Agent 会写入吗？
A. 只在**启动时读取**该文件并注入系统提示词。不保证写入——写入规则必须在 `system_prompt` 里约定。

**Q5.** 预置 Store 时，key 应该写 `/memories/preferences.md` 还是 `/preferences.md`？为什么？
A. `/preferences.md`。`CompositeBackend` 会**去掉路由前缀**再作为 Store key。

**Q6.** 三种 namespace 作用域分别是什么？默认该用哪个？
A. Agent 级 `(assistant_id,)` 共享；用户级 `(user_id, "memories")` 隔离；组织级 `(org_id,)` 通常只读。**默认用户级**。

**Q7.** 记忆文件不存在时会怎样？
A. 0.7.10 中 `memory=` 列表里不存在的文件会被**静默跳过**，不会自动创建。所以要先用 `store.put` 预置。

**Q8.** `InMemoryStore` 有什么限制？生产环境怎么办？
A. 只在当前进程内存有效，脚本重跑即清空；生产应换成 `PostgresStore` 等持久化实现。

**Q9.** 怎么证明"跨对话"生效，而不是同一轮对话的上下文残留？
A. 对话 2 必须使用**全新 `thread_id`**（如 `uuid.uuid4()`），若仍带上偏好才算成立。

**Q10.** 系统消息里为什么用字符串匹配判断记忆注入会失败？
A. Deep Agents 的系统消息 `content` 是**内容块列表** `[{"type":"text","text":...}]` 而非纯字符串，直接 `in` 判恒为 False，需先展平。

---

## 学习日期
2026-10-24（理论） / 2026-10-24（实操与复习）
