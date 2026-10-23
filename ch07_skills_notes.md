# 第 7 章学习笔记：Skills — 可复用的 Agent 能力包

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch07-skills/
> 配套 Notebook：https://github.com/datawhalechina/deepagents-in-action/blob/main/notebooks/ch07/01-skills-progressive-disclosure.ipynb
> 学习时间：2026-10-23

> ⚠️ 公开仓库安全提示：本文档不含任何明文 API Key / Token。所有密钥通过环境变量注入（如 `os.environ["DEEPSEEK_API_KEY"]`），真实密钥仅存于本地 `.env`（已 gitignore），切勿提交。

---

## 一、核心动机 / 概述

- **问题**：工具（Tools）只是原子操作；但像"代码审查、查文档、写报告"这类能力需要"多步骤工作流 + 领域知识 + 模板资源"的组合，单靠工具表达不了。
- **解决**：**Skill** 把这种组合打包成标准化、可复用的能力块。
- **类比**：Skills 之于 Agent，就像 npm 包之于 Node.js——一套标准化的能力分发与复用机制。遵循开放的 [Agent Skills 规范](https://agentskills.io/specification)，已被 30+ 主流 AI 工具采用（Claude Code、Cursor、Deep Agents、GitHub、Databricks 等）。

---

## 二、关键概念

| 概念 | 说明 |
|------|------|
| Skill | 一个目录 + 核心 `SKILL.md` + 可选 `scripts/`、`references/`、`assets/` |
| SKILL.md | 必需文件：YAML frontmatter（元数据）+ Markdown 正文（指令） |
| Progressive Disclosure | 渐进式披露：元数据→指令→资源，分三级按需加载 |
| SkillsMiddleware | 启动时扫描 skills 目录，提取 name/description 注入系统提示词 |
| skills 参数 | `create_deep_agent(skills=[...])` 指定发现路径列表 |
| 状态隔离 | 每个 Agent 拥有独立 Skill 状态空间，互不影响 |

---

## 三、定义方式 / 字段说明

### 3.1 目录结构
```text
skills/<skill-name>/
├── SKILL.md          # 必需：元数据 + 核心指令
├── scripts/          # 可选：可执行脚本（沙箱运行）
├── references/       # 可选：详细参考文档（按需 read_file）
└── assets/           # 可选：模板、schema 等静态资源
```

### 3.2 SKILL.md frontmatter 字段
| 字段 | 必填 | 继承主 Agent？ | 说明 |
|------|------|---------------|------|
| `name` | ✅ | — | 小写字母/数字/连字符，1–64 字符，须与父目录名一致 |
| `description` | ✅ | — | 功能与适用场景，≤1024 字符；**Agent 选 Skill 的唯一依据** |
| `license` | ❌ | — | 许可证名称 |
| `compatibility` | ❌ | — | 环境要求（网络/CLI），≤500 字符 |
| `metadata` | ❌ | — | 任意键值对（author/version/entrypoint 等） |
| `allowed-tools` | ❌ | — | 空格分隔的预授权工具列表 |

> 关键点：`description` 最重要——Agent **只靠它路由**，不会提前读正文；写具体行为导向的描述（❌"做研究" → ✅"深度网络研究、多次搜索交叉验证时使用"）。

---

## 四、渐进式披露（Progressive Disclosure）三级加载

| 层级 | 加载内容 | 加载时机 | 处理者 |
|------|----------|----------|--------|
| L1 元数据 | name + description + 文件路径 | Agent 启动时 | SkillsMiddleware |
| L2 指令 | SKILL.md 正文 | Agent 判断相关后 `read_file` | LLM 自行决策 |
| L3 资源 | scripts/references/assets | 正文引用到某文件时 | LLM 自行决策 |

工作流：启动仅注入元数据 → 用户请求后 Agent 按 description 匹配 → 调 `read_file` 读 SKILL.md → 按指令工作，引用资源时再读。

---

## 五、配置：skills 参数与三种后端

```python
agent = create_deep_agent(
    model=model,
    skills=[
        "/skills/shared/",    # 团队共享
        "/skills/project/",   # 项目专属（同名时后覆盖前）
    ],
)
```
- **同名覆盖规则**：列表中**后面的覆盖前面的（last wins）**。
- **FilesystemBackend**：`skills=["/skills/"]` 指向含 Skill 子目录的父目录；本地建议 `virtual_mode=True`。
- **StateBackend**：用 `create_file_data()` 格式化内容，经 `invoke` 的 `files` 注入。
- **StoreBackend**：跨线程持久化，用 `store.put(namespace=("filesystem",), key="/skills/x/SKILL.md", value=create_file_data(...))`。
- **运行时动态构造**：可按用户角色返回不同 skills 列表（如 engineering/data/support）。

---

## 六、子 Agent 继承与状态隔离

```python
research_subagent = {
    "name": "researcher",
    "description": "Research assistant with specialized skills",
    "system_prompt": "You are a researcher.",
    "tools": [web_search],
    "skills": ["/skills/research/", "/skills/web-search/"],  # 子 Agent 专属 Skills
}
agent = create_deep_agent(
    model=model,
    skills=["/skills/main/"],          # 主 Agent + general-purpose 子 Agent 使用
    subagents=[research_subagent],     # researcher 只有自己的 Skills
)
```
- **general-purpose 子 Agent**：自动继承主 Agent 的所有 skills，无需额外配置。
- **自定义子 Agent**：**不继承**主 Agent 的 skills，必须显式写 `skills` 字段。
- **状态完全隔离**：各 Agent 独立 Skill 状态空间，一个 Agent 改 Skill 文件不影响其他 Agent。

---

## 七、权限与代码执行

- **写权限控制**：`permissions=[FilesystemPermission(operations=["write"], paths=["/skills/**"], mode="deny" | "interrupt")]`。`deny` 直接拒绝；`interrupt` 需人工审批（需配 `checkpointer`）。
- **沙箱脚本**：`scripts/` 下脚本由沙箱后端（如 DaytonaSandbox）在隔离容器执行；SKILL.md 指令里写 `Run scripts/search.py "<query>"`。
- **解释器技能**：`metadata.entrypoint` 指定 JS/TS 入口，配 `CodeInterpreterMiddleware`，Agent 通过 `await import("@/skills/<name>")` 导入，实现确定性代码复用。

---

## 八、小结

1. **结构标准**：Skill = 目录 + `SKILL.md`（YAML+MD）+ 可选 `scripts/references/assets/`。
2. **发现机制**：启动仅注入 name+description；运行时 LLM 按需 `read_file` 正文与资源。
3. **配置**：`skills=[...]` 路径列表；last-wins 覆盖；支持 Filesystem/State/Store 后端与运行时动态构造。
4. **继承与隔离**：general-purpose 继承主 Agent skills；自定义子 Agent 不继承、须显式指定；状态空间隔离。
5. **生产可用**：`deny`/`interrupt` 控写权限，沙箱/解释器技能跑确定性代码。
6. **跨平台**：遵循 agentskills.io 标准，非 Deep Agents 私有，可跨 30+ 工具复用。

---

## 九、我的实操记录

> 目标：把前几章的学习经验做成可复用的"笔记内容规范" skill，在主项目 skills 目录新增一种内容规范。
> 环境：Windows PowerShell + DeepAgents；模型经 `os.environ["DEEPSEEK_API_KEY"]` 接入（OpenAI 兼容接口）。

### 改造点
在主项目 skills 目录 `D:/ai_tools/research_deepagent/.claude/skills/` 下**新增** skill `deepagents-course-notes`，结构对齐现有 skill 规范：

```
.claude/skills/deepagents-course-notes/
├── SKILL.md                    # 元数据 + 规范正文（含黄金安全规则）
├── template/note-template.md   # 填空式笔记模板（内容规范的载体）
└── reference/content-spec.md   # 各节细则 + Windows 原生环境经验
```

- `SKILL.md` 的 `description` 写明"课程笔记内容规范/模板/公开仓库安全"，让 Agent 在写笔记时路由到它。
- `template/note-template.md` 把 ch03/04/05 的统一写法固化为 10 节结构。
- `reference/content-spec.md` 沉淀字段表三列规则、提交前密钥扫描清单、Windows/SQLite/模型接入等环境经验。

---

## 十、如何验证

1. **文件产物**：`SKILL.md` / `template/note-template.md` / `reference/content-spec.md` 三个文件存在于 `.claude/skills/deepagents-course-notes/`，目录结构符合 Agent Skills 规范。
2. **安全扫描**：对 skill 目录 grep `sk-` / `tvly-` / `lsv2_` / `ghp_` → 仅命中"安全提示里的扫描说明"文字，无真实密钥。
3. **可发现性**：AgentSeek 自动发现 `.claude/skills/` 下的 skill，无需改 `skills-lock.json`（该 lock 仅记录来自 `../agentseek-skills` 的来源哈希，非功能必需）。
4. **（进阶验证）**：写 `skills_demo.py` 用 `FilesystemBackend` + `skills=["/skills/"]` 调 Agent "按规范写一份 XX 笔记"，观察其是否 `read_file` 读取本 skill 的 SKILL.md 正文，即验证渐进式披露生效。

---

## 十一、回忆复习 · 自测题（含答案）

**Q1. 什么是 Skill？它解决什么问题？**
A. Skill 是把"多步骤工作流 + 领域知识 + 模板资源"打包成的可复用能力块；解决单工具表达不了复杂能力的问题，类比 npm 包。

**Q2. SKILL.md 哪两个字段是必填的？**
A. `name`（与父目录同名）和 `description`（Agent 路由依据）。

**Q3. 为什么 description 最重要？**
A. Agent 只靠 description 决定调用哪个 Skill，不会提前读正文；描述越具体路由越准。

**Q4. Progressive Disclosure 三级分别加载什么？**
A. L1 元数据（name/description/路径，启动注入）；L2 指令（SKILL.md 正文，按需 read_file）；L3 资源（scripts/references/assets，引用时读）。

**Q5. skills 参数里出现同名 Skill 怎么处理？**
A. 列表中后面的覆盖前面的（last wins）。

**Q6. DeepAgents 支持哪三种 Skill 后端？**
A. FilesystemBackend（本地目录）、StateBackend（state 注入）、StoreBackend（跨线程持久化）。

**Q7. 子 Agent 如何继承主 Agent 的 skills？**
A. general-purpose 自动继承；自定义子 Agent 不继承，必须显式写 `skills` 字段。各 Agent 状态空间隔离。

**Q8. 如何防止 Skill 被随意改写？**
A. `permissions` 用 `FilesystemPermission(operations=["write"], paths=["/skills/**"], mode="deny")` 拒绝，或 `mode="interrupt"` 需人工审批（配 checkpointer）。

**Q9. 沙箱脚本和解释器技能有什么区别？**
A. 沙箱脚本在隔离容器执行 `scripts/` 下的 .py/.sh；解释器技能用 `metadata.entrypoint` + `CodeInterpreterMiddleware` 跑 JS/TS 确定性代码。

**Q10. 本次实操我新增了哪个 skill？它包含什么？**
A. 新增 `deepagents-course-notes`：内含 SKILL.md（笔记内容规范+安全规则）、template/note-template.md（填空模板）、reference/content-spec.md（细则与环境经验）。

---

## 学习日期
2026-10-23（理论 + 实操与复习）
