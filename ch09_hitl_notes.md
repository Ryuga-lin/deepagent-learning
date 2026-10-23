# 第 9 章学习笔记：Human-in-the-Loop —— 执行前等待审批

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch09-human-in-the-loop/
> 配套 Notebook：课程仓库 `notebooks/ch09/`（本次实操以本地脚本 `hitl_approval.py` 完成，离线、零 Token）
> 学习时间：2026-10-24

> ⚠️ 公开仓库安全提示：本文档不含任何明文 API Key / Token。本实验为**离线脚本化实验**，不联网、不消耗 Token、不产生真实副作用（不删真实文件、不发真实邮件）。

---

## 一、核心动机 / 概述

**问题**：Agent 越自主，风险越高。它可以删文件、发邮件、下单、部署——一旦判断失误，后果不可撤销。完全禁止这些工具则让 Agent 失去实用性；完全放任则不可控。

**解决**：引入 **Human-in-the-Loop（HITL）**——给指定工具挂上"审批闸门"。Agent 提出敏感调用时**暂停执行**，把"要做什么、带什么参数"交给人类；人批准才跑，人拒绝就不跑，人还可以**改参数**后再跑。这样既保留了 Agent 的自主规划能力，又把最终决定权留给人类。

> 关键认知：HITL 不是简单的"确认弹窗"。被拒绝/被修改的结果会**回传给模型**，模型会据此调整后续行为；框架还会明确告诉模型"别再按原样重试"。

---

## 二、关键概念

| 概念 | 说明 |
|------|------|
| **`interrupt_on`** | `create_deep_agent` 的参数，按**工具名**配置哪些工具需要审批、允许哪些决策 |
| **Checkpointer** | HITL 的**必要依赖**：没有它无法保存中断状态，也就无法恢复 |
| **`thread_id`** | 恢复必须与中断时**完全一致**，否则找不到挂起的请求 |
| **`__interrupt__`** | 中断发生时 `invoke` 返回值里的字段，含 `action_requests`（待审批调用）与 `review_configs`（可选决策） |
| **`action_requests`** | 待审批的工具调用列表（工具名 + 参数） |
| **`review_configs`** | 每个待审批调用对应的 `allowed_decisions` |
| **`Command(resume=...)`** | 恢复执行的入口，`resume={"decisions": [...]}` 携带人类决策 |
| **四种决策** | `approve`（批准）/ `edit`（改参数后执行）/ `reject`（拒绝并说明原因）/ `respond`（人亲自回答） |
| **幂等性** | 恢复时节点会**从头重放**，因此 `interrupt()` 之前的副作用必须可重复执行 |

---

## 三、定义方式 / 字段说明

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command
from deepagents import create_deep_agent

agent = create_deep_agent(
    model=model,
    tools=[delete_file, send_email],
    interrupt_on={
        "delete_file": {"allowed_decisions": ["approve", "edit", "reject"]},
        "send_email":  {"allowed_decisions": ["approve", "reject"]},
    },
    checkpointer=MemorySaver(),   # 必需
)
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `interrupt_on` | 是 | 字典：key = **工具名**（字符串），value = 审批配置 |
| `allowed_decisions` | 是 | 允许人类做出的决策类型列表，决定 UI / 脚本能给哪些选项 |
| `checkpointer` | **是（必需）** | 保存中断现场；缺失则中断后无法恢复 |
| `thread_id` | 恢复时必需 | 必须与发起中断那次调用**相同** |
| `decisions` | 恢复时必需 | 列表，需与 `action_requests` **顺序一一对应** |

### 四种决策对照

| 决策 | 语义 | 工具是否执行 | 典型用途 |
|------|------|-------------|----------|
| `approve` | 批准 | ✅ 按原参数执行 | 低风险确认 |
| `edit` | 改参数后执行 | ✅ 按**修改后的**参数执行 | 路径写错、收件人写错 |
| `reject` | 拒绝 | ❌ 不执行，原因回传模型 | **删除/发送/部署等副作用操作** |
| `respond` | 人亲自回答 | ❌ 不执行，用人的回答当工具结果 | `ask_user` 类"问人"工具 |

> ⚠️ **重要**：拒绝**有副作用**的工具（删除、发送、部署、付款）必须用 `reject`。若误用 `respond`，模型会把人的回复当成"工具执行成功的结果"，从而认为操作已完成——这是危险的语义错误。

---

## 四、中断 → 恢复的完整时序

```
① agent.invoke(用户输入, config={thread_id: T})
        ↓
② Agent 生成工具调用 → 命中 interrupt_on → 【暂停，工具尚未执行】
        ↓
③ invoke 返回，result["__interrupt__"] 里拿到 action_requests + review_configs
        ↓
④ 人类决策（可以是几秒后，也可以是几小时后）
        ↓
⑤ agent.invoke(Command(resume={"decisions": [...]}), config={thread_id: T})  ← 必须同一个 T
        ↓
⑥ 按决策执行/跳过 → 结果写回消息流 → Agent 继续推理
```

**第 ② 步是本章的核心**：中断期间工具**一次都没跑**，这才是"执行前等待审批"。

---

## 五、按风险分层配置

| 风险等级 | 工具示例 | 建议配置 |
|---------|---------|---------|
| **高**（不可逆、有外部副作用） | 删除文件、发送邮件、付款、部署 | `["approve", "edit", "reject"]` |
| **中**（可修改内容） | 写入/编辑文件、创建记录 | `["approve", "reject"]` |
| **低**（只读） | 读取文件、搜索、列举 | `False`（不拦截） |
| **问人型** | 向用户提问 | `["respond"]` |

原则：**只读放行，写入留痕，破坏性操作必须人工点头。**

---

## 六、底层原理：`interrupt()` 是如何暂停的

1. `interrupt()` 通过**抛出异常**来暂停图执行；
2. 暂停时，已执行到哪一步由 **Checkpointer** 落盘保存；
3. 恢复时，LangGraph 会**从节点开头重新执行（replay）**，直到再次遇到 `interrupt()`，这次直接返回人类决策。

由此产生两条硬约束：

- **`interrupt()` 之前的副作用必须幂等**（重复执行不出问题），否则重放时会重复扣款/重复发送。
- **不要用裸 `try/except` 包裹 `interrupt()`**，否则会把中断异常吞掉，审批机制彻底失效。

---

## 小结

1. `interrupt_on` 按工具名配置审批；**必须配 Checkpointer**，恢复必须用**同一个 `thread_id`**。
2. 中断期间工具**尚未执行**；批准后才真正跑。
3. 四种决策：`approve` / `edit` / `reject` / `respond`；**拒绝副作用工具用 `reject`，不用 `respond`**。
4. `decisions` 列表要与 `action_requests` **顺序一一对应**。
5. 按风险分层：高风险给全决策、中风险 approve/reject、只读不拦截。
6. `interrupt()` 靠抛异常暂停 + Checkpointer 保存；恢复时**节点从头重放** → 副作用要幂等、别裸 try/except。
7. HITL 的价值不只是"拦一下"：拒绝原因会回传模型，模型据此调整策略。

---

## 我的实操记录

> 目标：亲手扮演"审批人"，验证同一个请求在不同决策下产生不同结果。
> 环境：Windows 原生 PowerShell + 项目 venv；**离线**（脚本化模型 `ScriptedModel`，不联网、不消耗 Token）；工具为示例实现，不产生真实副作用。

### 改造点

**① 离线化：用脚本化模型固定提出一次工具调用**

```python
class ScriptedModel(BaseChatModel):
    def _generate(self, messages, stop=None, run_manager=None, **kwargs):
        already_ran_tool = any(isinstance(m, ToolMessage) for m in messages)
        if already_ran_tool:
            message = AIMessage(content="已按人工审批结果处理完毕。")
        else:
            message = AIMessage(content="", tool_calls=[self.tool_call])
        return ChatResult(generations=[ChatGeneration(message=message)])
```
好处：实验**可重复**、零成本，且结果不依赖模型心情。

**② 真正的交互审批（`ask_human`）**

```python
def ask_human(action_requests, review_configs) -> dict:
    print("\n⏸  Agent 已暂停 —— 此刻工具【尚未执行】，等你决策")
    ...
    choice = input("   >>> 你的决策 > ").strip().lower() or "approve"
```
第一版是脚本里写死 `decisions=[{"type":"approve"}]`，看起来像"自动审批"；改成 `input()` 后才是**真正由人决策**。

**③ 打印"执行记录"作为硬证据**

```python
print("\n执行记录  :")
for msg in result["messages"]:
    if getattr(msg, "type", "") == "tool":
        print(f"  - {msg.content}")
```
只看"最终回复"无法区分 approve / reject（两者结尾都像"处理完毕"），必须看**工具消息**里有没有真正的执行结果。

### 实际运行结果（三种决策全部验证 ✅）

**① `reject`（拒绝，原因"不清楚"）**
```
⏸  Agent 已暂停 —— 此刻工具【尚未执行】，等你决策
待审批工具: delete_file      工具参数: {'path': 'temp.txt'}
>>> 你的决策 > reject
执行记录：
- User rejected the tool call for `delete_file` with reason: 不清楚
```
→ 工具**没有执行**，拒绝原因回传给了 Agent。

**② `approve`（批准）**
```
>>> 你的决策 > approve
执行记录：
- [示例] 已删除 temp.txt
```
→ 工具**在你同意之后**才真正执行。

**③ `edit`（改参数后执行）**
```
>>> 你的决策 > edit
>>> 改成新路径 >            （回车用默认 archive/temp.txt）
执行记录：
- Note: a human reviewer replaced this tool call before it ran. The call recorded
  in your message is the one you produced, not the one that executed. This was
  intentional and authorized. Do not re-issue your original call.
  Executed instead: delete_file with arguments {"path": "archive/temp.txt"}.
  Tool response: [示例] 已删除 archive/temp.txt
```
→ 删除的是**你改后的路径**；框架还主动警告模型"别再按原样重试"。

### 结论

**同一个请求"删除 temp.txt"，三种决策产生三种结果，唯一变量是人的决策 → HITL 成立。**

---

## 如何验证

| # | 检查项 | 看到什么算成功 |
|---|--------|---------------|
| 1 | **确实发生了中断** | 首次 `invoke` 返回里有 `__interrupt__`；否则会打印"⚠️ 未发生中断，请检查 interrupt_on 配置" |
| 2 | **中断期间工具未执行** | 中断时的执行记录为空，"已删除"只出现在 resume **之后** |
| 3 | **approve 真的执行** | 执行记录出现 `[示例] 已删除 temp.txt` |
| 4 | **reject 真的没执行** | 执行记录出现 `User rejected ... with reason`，且**没有**"已删除"字样 |
| 5 | **edit 改参数生效** | 出现 `Executed instead: ...{"path":"archive/temp.txt"}` 与对应的"已删除 archive/temp.txt" |

运行方式：
```powershell
cd D:\ai_tools\research_deepagent
.\.venv\Scripts\python.exe hitl_approval.py          # 交互模式：由你来审批
.\.venv\Scripts\python.exe hitl_approval.py --auto   # 自动模式：快速对比 approve / reject
```

---

## 回忆复习 · 自测题（含答案）

**Q1.** HITL 缺少 Checkpointer 会怎样？
A. 无法保存中断现场，也就无法恢复；且中断本身会失败。

**Q2.** 恢复执行时 `thread_id` 能换吗？
A. 不能。必须与中断那次**完全一致**，否则找不到挂起的请求。

**Q3.** 中断发生的那一刻，工具执行了吗？
A. **没有**。工具在人类决策返回之后才执行（本实验中"已删除"只在 resume 后出现）。

**Q4.** 四种决策分别是什么？
A. `approve`（批准）、`edit`（改参数后执行）、`reject`（拒绝并回传原因）、`respond`（人亲自回答）。

**Q5.** 拒绝"删除文件"应该用 `reject` 还是 `respond`？为什么？
A. 用 `reject`。`respond` 会把人的回复当成工具的成功结果，模型会误以为删除已完成。

**Q6.** 多个工具同时待审批时，`decisions` 怎么给？
A. 列表形式，与 `action_requests` **顺序一一对应**。

**Q7.** 按风险分层，只读工具应该配成什么？
A. 配 `False`（不拦截），避免审批疲劳。

**Q8.** 为什么 `interrupt()` 前的副作用必须幂等？
A. 恢复时节点会**从头重放**，非幂等操作（如付款、发信）会被重复执行。

**Q9.** 为什么不能用裸 `try/except` 包裹 `interrupt()`？
A. 中断靠抛异常实现，裸 except 会把异常吞掉，审批机制失效。

**Q10.** 本实验中，判断"审批真生效"的最硬证据是什么？
A. 看**工具消息（执行记录）**：approve 有"已删除"、reject 只有"User rejected"、edit 显示 `Executed instead` 改后的参数——而不只看模型的最终回复。

---

## 学习日期
2026-10-24（理论） / 2026-10-24（实操与复习）
