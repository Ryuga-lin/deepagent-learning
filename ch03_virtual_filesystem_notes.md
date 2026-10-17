# 第 3 章学习笔记：虚拟文件系统 — Deep Agents 的 Context Engineering 核心

> 课程地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch03-virtual-filesystem/
> 章节定位：核心篇 EP.03

---

## 一、核心问题与解决方案

### 传统 Agent 的致命问题
所有信息都直接塞进 prompt——文件内容、搜索结果、中间计算全部挤在对话历史里，导致上下文膨胀、成本飙升、模型注意力分散。

### Deep Agents 的方案：给 Agent 一个文件系统
设计思路类比人类工作方式：
- 不把所有资料同时铺在桌上，而是分门别类存放
- 需要时再取出来
- 用搜索快速定位
- 在便签纸上记录中间结果

---

## 二、七个内置文件工具

| 工具 | 用途 | 类比 |
|------|------|------|
| `ls` | 列出目录中的文件和元信息 | 打开文件夹 |
| `read_file` | 读取文件内容，支持偏移量和限制条数 | 翻开资料阅读 |
| `write_file` | 创建文件或完整覆盖已有文件 | 写新备忘录 |
| `edit_file` | 对已有文件做精确字符串替换 | 用红笔修改 |
| `delete` | 删除文件或目录（v0.7 新增） | 清理资料 |
| `glob` | 按模式匹配查找文件 | 按标签找 |
| `grep` | 搜索文件内容，支持三种输出模式 | 全文检索 |

### v0.7 重要变更
- `write_file` 会**完整覆盖**同路径文件；只改局部用 `edit_file`
- `read_file` 返回分页信息和下一段偏移量
- `grep` 默认最多保留 **1,000 个匹配**
- 空的 `ls`/`glob` 返回 `No files found`
- `read_file` 行号与正文间用**两个空格**分隔（不再用 Tab）

---

## 三、read_file 的两大特性

### 3.1 分片读取
```python
# 默认读取前 100 行
read_file("/workspace/report.md")

# 从第 100 行开始，读取 50 行
read_file("/workspace/report.md", offset=100, limit=50)
```

### 3.2 原生多模态支持

| 类型 | 支持格式 |
|------|----------|
| 图片 | .png .jpg .jpeg .gif .webp .heic .heif |
| 视频 | .mp4 .mpeg .mov .avi .flv .mpg .webm .wmv .3gpp |
| 音频 | .wav .mp3 .aiff .aac .ogg .flac |
| 文档 | .pdf .ppt .pptx |

Agent 可以直接处理截图、录音、演示文稿。

---

## 四、grep 三种输出模式

```python
# 模式 1：只返回匹配的文件路径（快速定位）
grep("TODO", glob="**/*.py", output_mode="files_with_matches")

# 模式 2：返回匹配行及上下文（深入查看）
grep("def create_agent", output_mode="content")

# 模式 3：返回匹配数量（概览统计）
grep("error", output_mode="count")
```

---

## 五、上下文自动管理机制（核心）

### 5.1 大结果自动卸载
当工具调用的输入或输出超过 **20,000 tokens**（可配置 `tool_token_limit_before_evict`）时：
1. 完整内容自动写入虚拟文件系统
2. 对话历史中替换为文件路径引用 + 前 10 行预览
3. Agent 需要时可通过 `read_file` 或 `grep` 读回

**完全自动**，Agent 不需要手动管理。

### 5.2 对话历史总结
当上下文达到阈值时（默认模型窗口的 **85%**）：
1. 旧消息写入 Backend 保存
2. LLM 生成结构化摘要（意图、产出物、下一步）
3. 模型输入改为"摘要 + 近期消息"
4. State 中的原始消息仍然保留

**关键区分**："保存的消息历史"和"模型这次看到的内容"是两回事。

---

## 六、可插拔存储后端

| 后端 | 特点 | 适用场景 |
|------|------|----------|
| StateBackend（默认） | 临时存储，对话内有效 | 学习实验 |
| FilesystemBackend | 本地磁盘，永久修改 | 本地编程助手 |
| LocalShellBackend | 本地 Shell 执行（极高风险） | 个人开发机 |
| StoreBackend | 跨会话持久化 | 长期记忆 |
| CompositeBackend | 混合路由 | 临时+持久 |
| 沙箱后端 | 安全隔离执行 | 生产环境 |

### 关键代码示例

**StateBackend（默认）：**
```python
agent = create_deep_agent(model=model)  # 不需要显式指定
```

**FilesystemBackend：**
```python
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    model=model,
    backend=FilesystemBackend(root_dir=".", virtual_mode=True)
)
```

**CompositeBackend（混合路由）：**
```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    model=model,
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/memories/": StoreBackend(
                namespace=lambda rt: (rt.server_info.user.identity,),
            ),
        }
    ),
    store=InMemoryStore()
)
```

> 本地调试 StoreBackend namespace 兜底写法：
> ```python
> namespace=lambda rt: (
>     (rt.server_info.user.identity,)
>     if rt.server_info else
>     ("local-user",)
> ),
> ```

---

## 七、权限控制

### 声明式权限 FilesystemPermission

```python
from deepagents import create_deep_agent, FilesystemPermission

agent = create_deep_agent(
    model=model,
    permissions=[
        FilesystemPermission(
            operations=["write"],
            paths=["/policies/**"],
            mode="deny",
        ),
    ],
)
```

| mode | 行为 | 适用场景 |
|------|------|----------|
| `"allow"` | 允许执行 | 设置显式例外 |
| `"deny"` | 直接拒绝 | 禁止访问的路径 |
| `"interrupt"` | 暂停等待审批 | 敏感路径需审批 |

**求值规则**：按声明顺序求值，first-match-wins；无规则匹配则默认允许。

---

## 八、自定义后端

实现 `BackendProtocol` 接口：

```python
from deepagents.backends.protocol import (
    BackendProtocol, WriteResult, EditResult, DeleteResult,
    LsResult, ReadResult, GrepResult, GlobResult,
)

class S3Backend(BackendProtocol):
    def ls(self, path): ...
    def read(self, file_path, offset=0, limit=2000): ...
    def write(self, file_path, content): ...
    def edit(self, file_path, old_string, new_string, replace_all=False): ...
    def grep(self, pattern, path=None, glob=None): ...
    def glob(self, pattern, path="/"): ...
    def delete(self, file_path): ...
```

### 安全策略：PolicyWrapper

```python
class PolicyWrapper(BackendProtocol):
    def __init__(self, inner: BackendProtocol, deny_prefixes: list[str]):
        self.inner = inner
        self.deny_prefixes = deny_prefixes

    def write(self, file_path, content):
        if self._deny(file_path):
            return WriteResult(error="写入被拒绝")
        return self.inner.write(file_path, content)
    # ... 其他方法转发
```

---

## 九、后端选择指南

| 场景 | 推荐后端 | 理由 |
|------|----------|------|
| 学习和实验 | StateBackend（默认） | 零配置，自动清理 |
| 本地编程助手 | FilesystemBackend | 直接操作项目文件 |
| 需要跨会话记忆 | CompositeBackend | 混合临时 + 持久化 |
| 需要执行代码 | 沙箱后端 | 安全隔离 |
| 生产部署 | StoreBackend 或 CompositeBackend | 持久化 + 可伸缩 |

---

## 十、关键要点

1. **虚拟文件系统是 Context Engineering 的核心**——不是塞 prompt，而是按需读写
2. **7 个内置工具**覆盖文件操作全生命周期
3. **自动上下文管理**：大结果卸载为文件引用，旧消息自动摘要
4. **5 种后端**按需选择，StateBackend 是默认安全选项
5. **权限控制**通过 FilesystemPermission 声明式配置
6. **v0.7 变更**：backend 工厂函数移除，必须传实例；virtual_mode 建议开启

---

## 学习日期
2026-10-17