# SessionManager 会话管理深度解析

## 概述

**SessionManager** 负责管理所有对话会话的创建、持久化和生命周期。

**文件**: `nanobot/session/manager.py` (213 行)

**核心职责**:
1. 创建和获取会话
2. 持久化会话到磁盘
3. 管理会话历史和归档
4. 提供会话缓存机制

---

## 🏗️ 架构设计

### 核心类

```python
@dataclass
class Session:
    """
    对话会话

    以 JSONL 格式存储消息，便于阅读和持久化

    重要：消息是仅追加的（append-only），以提高 LLM 缓存效率
    归档过程会将摘要写入 MEMORY.md/HISTORY.md
    但不会修改消息列表或 get_history() 的输出
    """

    key: str                              # channel:chat_id
    messages: list[dict[str, Any]]        # 消息列表
    created_at: datetime                  # 创建时间
    updated_at: datetime                  # 更新时间
    metadata: dict[str, Any]              # 元数据
    last_consolidated: int                # 已归档到文件的消息数
```

### SessionManager

```python
class SessionManager:
    """
    管理对话会话

    会话作为 JSONL 文件存储在 sessions 目录中
    """

    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.sessions_dir = ensure_dir(workspace / "sessions")     # 工作区会话目录
        self.legacy_sessions_dir = Path.home() / ".nanobot" / "sessions"  # 旧版全局会话目录
        self._cache: dict[str, Session] = {}                       # 内存缓存
```

---

## 🔄 核心方法详解

### 1. `get_or_create()` - 获取或创建会话

**位置**: `manager.py:95-113`

```python
def get_or_create(self, key: str) -> Session:
    """
    获取现有会话或创建新会话

    Args:
        key: 会话键（通常是 channel:chat_id）

    Returns:
        会话对象
    """
    # 1. 检查内存缓存
    if key in self._cache:
        return self._cache[key]

    # 2. 尝试从磁盘加载
    session = self._load(key)
    if session is None:
        # 3. 创建新会话
        session = Session(key=key)

    # 4. 缓存会话
    self._cache[key] = session
    return session
```

**流程图**:

```
get_or_create(key)
    │
    ▼
检查内存缓存
    │
    ├─▶ 命中 → 返回缓存会话
    │
    └─▶ 未命中
        │
        ▼
    从磁盘加载
        │
        ├─▶ 成功 → 缓存并返回
        │
        └─▶ 失败
            │
            ▼
        创建新会话
            │
            ▼
        缓存并返回
```

---

### 2. `_load()` - 从磁盘加载会话

**位置**: `manager.py:115-160`

```python
def _load(self, key: str) -> Session | None:
    """从磁盘加载会话"""
    path = self._get_session_path(key)

    # 1. 检查新路径
    if not path.exists():
        # 2. 尝试迁移旧路径
        legacy_path = self._get_legacy_session_path(key)
        if legacy_path.exists():
            try:
                shutil.move(str(legacy_path), str(path))
                logger.info("Migrated session {} from legacy path", key)
            except Exception:
                logger.exception("Failed to migrate session {}", key)

    # 3. 文件不存在
    if not path.exists():
        return None

    # 4. 解析 JSONL 文件
    try:
        messages = []
        metadata = {}
        created_at = None
        last_consolidated = 0

        with open(path, encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if not line:
                    continue

                data = json.loads(line)

                # 第一行是元数据
                if data.get("_type") == "metadata":
                    metadata = data.get("metadata", {})
                    created_at = datetime.fromisoformat(data["created_at"]) if data.get("created_at") else None
                    last_consolidated = data.get("last_consolidated", 0)
                else:
                    # 后续行是消息
                    messages.append(data)

        return Session(
            key=key,
            messages=messages,
            created_at=created_at or datetime.now(),
            metadata=metadata,
            last_consolidated=last_consolidated
        )
    except Exception as e:
        logger.warning("Failed to load session {}: {}", key, e)
        return None
```

**JSONL 文件格式**:

```jsonl
{"_type": "metadata", "key": "telegram:123456", "created_at": "2026-02-25T18:30:00", "updated_at": "2026-02-25T18:35:00", "metadata": {}, "last_consolidated": 0}
{"role": "user", "content": "Hello", "timestamp": "2026-02-25T18:30:00"}
{"role": "assistant", "content": "Hi there!", "timestamp": "2026-02-25T18:30:05"}
{"role": "user", "content": "How are you?", "timestamp": "2026-02-25T18:31:00"}
```

**优势**:
- 每行一个 JSON 对象，易于解析
- 可追加写入，性能好
- 支持流式读取
- 人类可读

---

### 3. `save()` - 保存会话到磁盘

**位置**: `manager.py:162-179`

```python
def save(self, session: Session) -> None:
    """保存会话到磁盘"""
    path = self._get_session_path(session.key)

    with open(path, "w", encoding="utf-8") as f:
        # 1. 写入元数据行
        metadata_line = {
            "_type": "metadata",
            "key": session.key,
            "created_at": session.created_at.isoformat(),
            "updated_at": session.updated_at.isoformat(),
            "metadata": session.metadata,
            "last_consolidated": session.last_consolidated
        }
        f.write(json.dumps(metadata_line, ensure_ascii=False) + "\n")

        # 2. 写入所有消息
        for msg in session.messages:
            f.write(json.dumps(msg, ensure_ascii=False) + "\n")

    # 3. 更新缓存
    self._cache[session.key] = session
```

**关键点**:
- 覆盖写入（不追加）
- 确保数据一致性
- 更新内存缓存

---

### 4. `Session.get_history()` - 获取对话历史

**位置**: `manager.py:45-63`

```python
def get_history(self, max_messages: int = 500) -> list[dict[str, Any]]:
    """
    返回未归档的消息用于 LLM 输入，对齐到用户轮次

    Args:
        max_messages: 最大消息数

    Returns:
        消息列表
    """
    # 1. 获取未归档的消息
    unconsolidated = self.messages[self.last_consolidated:]

    # 2. 限制消息数量
    sliced = unconsolidated[-max_messages:]

    # 3. 删除前导非用户消息（避免孤立的 tool_result 块）
    for i, m in enumerate(sliced):
        if m.get("role") == "user":
            sliced = sliced[i:]
            break

    # 4. 清理消息格式
    out: list[dict[str, Any]] = []
    for m in sliced:
        entry: dict[str, Any] = {"role": m["role"], "content": m.get("content", "")}
        for k in ("tool_calls", "tool_call_id", "name"):
            if k in m:
                entry[k] = m[k]
        out.append(entry)
    return out
```

**对齐到用户轮次**:

```
原始消息序列:
[assistant, tool, tool, user, assistant, tool, user, assistant]
                                              ↑
                                    last_consolidated

未归档消息:
[assistant, tool, user, assistant, tool, user, assistant]

对齐后（从第一个 user 开始）:
[user, assistant, tool, user, assistant]
```

**为什么重要**:
- 避免孤立的 `tool_result` 消息
- 确保 LLM 看到完整的对话轮次
- 提高响应质量

---

### 5. `Session.clear()` - 清空会话

**位置**: `manager.py:65-69`

```python
def clear(self) -> None:
    """清空所有消息并将会话重置到初始状态"""
    self.messages = []
    self.last_consolidated = 0
    self.updated_at = datetime.now()
```

**用途**: `/new` 命令实现

---

### 6. `list_sessions()` - 列出所有会话

**位置**: `manager.py:185-212`

```python
def list_sessions(self) -> list[dict[str, Any]]:
    """
    列出所有会话

    Returns:
        会话信息字典列表
    """
    sessions = []

    for path in self.sessions_dir.glob("*.jsonl"):
        try:
            # 只读取元数据行
            with open(path, encoding="utf-8") as f:
                first_line = f.readline().strip()
                if first_line:
                    data = json.loads(first_line)
                    if data.get("_type") == "metadata":
                        key = data.get("key") or path.stem.replace("_", ":", 1)
                        sessions.append({
                            "key": key,
                            "created_at": data.get("created_at"),
                            "updated_at": data.get("updated_at"),
                            "path": str(path)
                        })
        except Exception:
            continue

    # 按更新时间排序（最新的在前）
    return sorted(sessions, key=lambda x: x.get("updated_at", ""), reverse=True)
```

**性能优化**:
- 只读取第一行（元数据）
- 不加载所有消息
- 快速列出所有会话

---

## 📊 会话键 (Session Key)

### 格式

```
channel:chat_id
```

### 示例

| 渠道 | Session Key |
|------|-------------|
| Telegram | `telegram:123456789` |
| Discord | `discord:123456789012345678` |
| CLI | `cli:direct` |
| WhatsApp | `whatsapp:1234567890` |
| Feishu | `feishu:ou_xxxxx` |

### 安全处理

```python
def _get_session_path(self, key: str) -> Path:
    """获取会话的文件路径"""
    safe_key = safe_filename(key.replace(":", "_"))
    return self.sessions_dir / f"{safe_key}.jsonl"
```

- `:` 替换为 `_`
- 应用 `safe_filename` 过滤非法字符
- 示例: `telegram:123456` → `telegram_123456.jsonl`

---

## 🗂️ 文件组织

### 目录结构

```
workspace/
├── sessions/                 # 会话目录
│   ├── telegram_123456.jsonl
│   ├── discord_789012.jsonl
│   └── cli_direct.jsonl
└── memory/                   # 记忆目录
    ├── MEMORY.md             # 长期记忆
    └── HISTORY.md            # 历史日志
```

### 旧版迁移

```
~/.nanobot/sessions/          # 旧版全局会话
    └── telegram_123456.jsonl
         │
         ▼ (自动迁移)
workspace/sessions/           # 新版工作区会话
    └── telegram_123456.jsonl
```

---

## 💾 持久化机制

### JSONL 格式

**优势**:
1. **追加友好**: 每行独立，易于追加
2. **流式处理**: 可以逐行读取，不需要全部加载
3. **容错性**: 单行损坏不影响其他行
4. **可读性**: 人类可读，易于调试
5. **兼容性**: 标准 JSON 格式

**文件示例**:

```jsonl
{"_type": "metadata", "key": "telegram:123456", "created_at": "2026-02-25T18:00:00", "updated_at": "2026-02-25T18:30:00", "metadata": {}, "last_consolidated": 10}
{"role": "user", "content": "Hello", "timestamp": "2026-02-25T18:00:00"}
{"role": "assistant", "content": "Hi!", "timestamp": "2026-02-25T18:00:05"}
{"role": "user", "content": "What can you do?", "timestamp": "2026-02-25T18:01:00"}
{"role": "assistant", "content": "I can help you with...", "timestamp": "2026-02-25T18:01:05", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "call_123", "name": "web_search", "content": "..."}
{"role": "assistant", "content": "Here's what I found...", "timestamp": "2026-02-25T18:01:10"}
```

### Append-Only 设计

```python
# 重要：消息是仅追加的
# 归档过程不会修改 messages 列表
# 而是通过 last_consolidated 标记已归档的位置
```

**优势**:
- **LLM 缓存友好**: 消息 ID 不变
- **版本控制友好**: 易于追踪变更
- **性能**: 不需要复制或删除数据

**实现**:

```python
# 获取未归档的消息
unconsolidated = session.messages[session.last_consolidated:]

# 归档后更新标记
session.last_consolidated = len(session.messages) - keep_count
```

---

## 🧠 记忆归档 (Consolidation)

### 归档机制

**文件**: `nanobot/agent/memory.py` (151 行)

```python
class MemoryStore:
    """
    两层记忆系统：
    - MEMORY.md: 长期事实
    - HISTORY.md: 可搜索的日志
    """

    async def consolidate(
        self,
        session: Session,
        provider: LLMProvider,
        model: str,
        *,
        archive_all: bool = False,
        memory_window: int = 50,
    ) -> bool:
        """
        将旧消息归档到 MEMORY.md + HISTORY.md

        Returns:
            成功返回 True（包括无操作），失败返回 False
        """
```

### 归档流程

```
1. 确定要归档的消息
   │
   ├─▶ archive_all=True: 归档所有消息
   │
   └─▶ archive_all=False:
       ├─▶ 保留最近 memory_window // 2 条消息
       └─▶ 归档 last_consolidated 到 -keep_count 之间的消息
   │
   ▼
2. 格式化消息为文本
   │
   ▼
3. 调用 LLM 生成摘要
   │
   ├─▶ history_entry: 添加到 HISTORY.md
   └─▶ memory_update: 更新 MEMORY.md
   │
   ▼
4. 更新 session.last_consolidated
   │
   ▼
5. 返回成功/失败
```

### 归档工具定义

```python
_SAVE_MEMORY_TOOL = [
    {
        "type": "function",
        "function": {
            "name": "save_memory",
            "description": "保存记忆归档结果到持久化存储",
            "parameters": {
                "type": "object",
                "properties": {
                    "history_entry": {
                        "type": "string",
                        "description": "摘要段落（2-5 句话），总结关键事件/决策/主题。"
                                    "以 [YYYY-MM-DD HH:MM] 开头。包含对 grep 搜索有用的细节。"
                    },
                    "memory_update": {
                        "type": "string",
                        "description": "完整更新的长期记忆（markdown 格式）。"
                                    "包含所有现有事实加上新事实。如果没有新内容则返回不变。"
                    }
                },
                "required": ["history_entry", "memory_update"],
            },
        },
    }
]
```

### 归档示例

**输入消息**:

```
[2026-02-25 18:00] USER: Hello, I'm John
[2026-02-25 18:00] ASSISTANT: Hi John! How can I help you?
[2026-02-25 18:01] USER: I need help with Python
[2026-02-25 18:01] ASSISTANT: [tools: web_search] Let me search for Python resources...
[2026-02-25 18:02] TOOL: ... (search results)
[2026-02-25 18:02] ASSISTANT: Here are some great Python resources...
```

**LLM 输出**:

```json
{
  "history_entry": "[2026-02-25 18:00] User John introduced himself and asked for help with Python. Assistant provided Python learning resources and tutorials after searching.",
  "memory_update": "## Long-term Memory\n\n### Users\n- **John**: Interested in learning Python, asked for resources on 2026-02-25.\n\n### Topics\n- Python programming: Provided learning resources and tutorials."
}
```

**更新文件**:

```markdown
# HISTORY.md
[2026-02-25 18:00] User John introduced himself and asked for help with Python. Assistant provided Python learning resources and tutorials after searching.

# MEMORY.md
## Long-term Memory

### Users
- **John**: Interested in learning Python, asked for resources on 2026-02-25.

### Topics
- Python programming: Provided learning resources and tutorials.
```

---

## 🎯 设计模式和原则

### 1. Dataclass 模式

```python
@dataclass
class Session:
    key: str
    messages: list[dict[str, Any]] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
```

**优势**:
- 自动生成 `__init__`, `__eq__`, `__repr__`
- 不可变性（使用 `frozen=True`）
- 类型提示

### 2. 缓存模式

```python
def get_or_create(self, key: str) -> Session:
    if key in self._cache:
        return self._cache[key]
    # ... 加载或创建
    self._cache[key] = session
    return session
```

**优势**:
- 减少磁盘 I/O
- 提高性能
- 简单有效

### 3. 策略模式

```python
# 不同的归档策略
archive_all: bool = False   # 策略选择
memory_window: int = 50     # 策略参数
```

### 4. 迁移模式

```python
# 自动从旧路径迁移到新路径
if not path.exists():
    legacy_path = self._get_legacy_session_path(key)
    if legacy_path.exists():
        shutil.move(str(legacy_path), str(path))
```

---

## 🚀 性能优化

### 1. 内存缓存

```python
self._cache: dict[str, Session] = {}
```

**优势**:
- 避免重复加载
- 快速访问
- 自动失效

### 2. 延迟加载

```python
# 只在需要时加载会话
session = self._load(key)
```

### 3. 流式读取

```python
# JSONL 格式支持逐行读取
with open(path, encoding="utf-8") as f:
    for line in f:
        data = json.loads(line)
```

### 4. 部分读取

```python
# list_sessions 只读取元数据行
first_line = f.readline().strip()
```

---

## 🔒 并发控制

### 会话锁定

虽然代码中没有显式锁定，但：
- 每个会话独立文件
- 单进程设计（通常）
- 原子写入（完整覆盖）

### 缓存一致性

```python
def save(self, session: Session) -> None:
    # ... 写入磁盘
    self._cache[session.key] = session  # 更新缓存
```

### 失效机制

```python
def invalidate(self, key: str) -> None:
    """从内存缓存中移除会话"""
    self._cache.pop(key, None)
```

**用途**:
- `/new` 命令后
- 会话清空后
- 强制重新加载

---

## 🧪 测试建议

### 单元测试

1. **会话创建**:
   ```python
   def test_session_creation():
       session = Session(key="test:123")
       assert session.key == "test:123"
       assert len(session.messages) == 0
       assert session.last_consolidated == 0
   ```

2. **消息添加**:
   ```python
   def test_add_message():
       session = Session(key="test:123")
       session.add_message("user", "Hello")
       assert len(session.messages) == 1
       assert session.messages[0]["role"] == "user"
   ```

3. **历史获取**:
   ```python
   def test_get_history():
       session = Session(key="test:123")
       session.add_message("user", "Hello")
       session.add_message("assistant", "Hi")
       history = session.get_history()
       assert len(history) == 2
   ```

4. **会话持久化**:
   ```python
   def test_save_and_load():
       manager = SessionManager(workspace)
       session = Session(key="test:123")
       session.add_message("user", "Hello")
       manager.save(session)

       loaded = manager.get_or_create("test:123")
       assert len(loaded.messages) == 1
   ```

### 集成测试

1. **归档流程**:
   ```python
   async def test_consolidation():
       session = Session(key="test:123")
       # 添加 100 条消息
       for i in range(100):
           session.add_message("user", f"Message {i}")

       memory = MemoryStore(workspace)
       success = await memory.consolidate(
           session, provider, model, memory_window=50
       )
       assert success is True
       assert session.last_consolidated > 0
   ```

2. **会话迁移**:
   ```python
   def test_legacy_migration():
       # 在旧路径创建会话
       legacy_path = Path.home() / ".nanobot" / "sessions" / "test_123.jsonl"
       legacy_path.parent.mkdir(parents=True, exist_ok=True)
       legacy_path.write_text('{"role": "user", "content": "Hello"}')

       manager = SessionManager(workspace)
       session = manager.get_or_create("test:123")

       # 验证迁移
       new_path = workspace / "sessions" / "test_123.jsonl"
       assert new_path.exists()
   ```

---

## 🔍 关键概念总结

### 1. Session Key（会话键）

格式: `channel:chat_id`
- 唯一标识一个会话
- 用于文件命名和缓存
- 安全处理特殊字符

### 2. JSONL 格式

- 每行一个 JSON 对象
- 第一行是元数据
- 后续行是消息
- 人类可读，易于解析

### 3. Append-Only 设计

- 消息只追加，不修改
- 通过 `last_consolidated` 标记归档位置
- LLM 缓存友好

### 4. 两层记忆

- **MEMORY.md**: 长期事实和知识
- **HISTORY.md**: 可搜索的事件日志

### 5. 记忆归档

- 自动将旧对话压缩成摘要
- 使用 LLM 生成结构化记忆
- 节省 token 使用

### 6. 内存缓存

- 避免重复加载
- 提高访问速度
- 支持手动失效

---

## 📖 完整数据流

### 创建会话

```
用户发送消息
    │
    ▼
AgentLoop._process_message()
    │
    ▼
SessionManager.get_or_create(key)
    │
    ├─▶ 检查缓存 → 命中 → 返回
    │
    └─▶ 未命中
        │
        ├─▶ _load(key) → 成功 → 缓存并返回
        │
        └─▶ 失败
            │
            ▼
        Session(key=key) → 缓存并返回
```

### 保存会话

```
AgentLoop._save_turn()
    │
    ▼
Session.messages.append(new_messages)
    │
    ▼
SessionManager.save(session)
    │
    ├─▶ 序列化为 JSONL
    ├─▶ 写入磁盘
    └─▶ 更新缓存
```

### 记忆归档

```
触发归档条件
    │
    ▼
MemoryStore.consolidate()
    │
    ├─▶ 确定要归档的消息
    ├─▶ 调用 LLM 生成摘要
    ├─▶ 更新 MEMORY.md
    ├─▶ 更新 HISTORY.md
    └─▶ 更新 session.last_consolidated
```

---

## 🎓 学习要点

### 必须理解

1. **会话键设计**: `channel:chat_id` 格式和唯一性
2. **JSONL 格式**: 为什么选择 JSONL 而不是 JSON
3. **Append-Only**: 为什么消息不删除，而是标记归档
4. **记忆归档**: 如何使用 LLM 压缩对话
5. **缓存策略**: 何时加载、缓存、失效会话
6. **历史对齐**: 为什么要对齐到用户轮次

### 代码阅读路径

1. 从 `Session` 数据类开始（数据结构）
2. 进入 `SessionManager.get_or_create()`（主要入口）
3. 研究 `_load()` 和 `save()`（持久化）
4. 理解 `get_history()`（历史获取）
5. 学习 `MemoryStore.consolidate()`（记忆归档）

---

## 🚀 下一步

现在你已经深入理解了 SessionManager！

想继续学习哪个模块？
- ToolRegistry - 工具系统
- MemoryStore - 记忆系统（已部分涉及）
- Channel Adapter - 渠道集成
- Provider System - LLM 提供商
