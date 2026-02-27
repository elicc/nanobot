# Memory 系统实现逻辑深度解析

## 概述

**Memory 系统**是 nanobot 的长期记忆管理模块，负责将对话历史压缩为结构化记忆。

**文件**: `nanobot/agent/memory.py` (151 行)

**核心功能**:
1. 长期记忆存储 (MEMORY.md)
2. 历史日志记录 (HISTORY.md)
3. 记忆归档 (consolidation)
4. 记忆检索和注入

---

## 🏗️ 架构设计

### 两层记忆结构

```
┌─────────────────────────────────────────────────────────┐
│                    Memory System                         │
│   ┌───────────────────────────────────────────────────┐ │
│   │  MEMORY.md (长期记忆)                              │ │
│   │  - 用户信息                                       │ │
│   │  - 偏好设置                                       │ │
│   │  - 项目上下文                                     │ │
│   │  - 重要事项                                       │ │
│   └───────────────────────────────────────────────────┘ │
│   ┌───────────────────────────────────────────────────┐ │
│   │  HISTORY.md (历史日志)                            │ │
│   │  - 时间线记录                                     │ │
│   │  - 关键事件                                       │ │
│   │  - 决策和主题                                     │ │
│   │  - 可搜索的日志                                   │ │
│   └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### MemoryStore 类

```python
class MemoryStore:
    """
    两层记忆系统：
    - MEMORY.md: 长期事实
    - HISTORY.md: 可搜索的日志
    """

    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"
```

---

## 🔄 核心方法详解

### 1. `read_long_term()` - 读取长期记忆

**位置**: `memory.py:53-56`

```python
def read_long_term(self) -> str:
    """读取长期记忆文件"""
    if self.memory_file.exists():
        return self.memory_file.read_text(encoding="utf-8")
    return ""
```

**用途**:
- 在构建上下文时加载长期记忆
- 作为归档的当前状态输入

### 2. `write_long_term()` - 写入长期记忆

**位置**: `memory.py:58-59`

```python
def write_long_term(self, content: str) -> None:
    """写入长期记忆文件"""
    self.memory_file.write_text(content, encoding="utf-8")
```

**特性**:
- 完全覆盖写入
- 保持 Markdown 格式
- 人类可读

### 3. `append_history()` - 追加历史日志

**位置**: `memory.py:61-63`

```python
def append_history(self, entry: str) -> None:
    """追加历史日志条目"""
    with open(self.history_file, "a", encoding="utf-8") as f:
        f.write(entry.rstrip() + "\n\n")
```

**特性**:
- 追加写入，不覆盖
- 自动添加换行
- 时间顺序增长

### 4. `get_memory_context()` - 获取记忆上下文

**位置**: `memory.py:65-67`

```python
def get_memory_context(self) -> str:
    """
    获取记忆上下文，用于注入到系统提示词

    Returns:
        格式化的长期记忆字符串
    """
    long_term = self.read_long_term()
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

**用途**:
- 在 `ContextBuilder.build_system_prompt()` 中调用
- 将长期记忆注入到 LLM 的系统提示词中

---

## 🗜️ 记忆归档 (Consolidation)

### 归档触发条件

**位置**: `agent/loop.py:363-380`

```python
# 当未归档消息超过窗口大小时，触发后台归档
unconsolidated = len(session.messages) - session.last_consolidated

if (unconsolidated >= self.memory_window
    and session.key not in self._consolidating):

    self._consolidating.add(session.key)
    lock = self._get_consolidation_lock(session.key)

    # 后台任务：归档记忆
    async def _consolidate_and_unlock():
        try:
            async with lock:
                await self._consolidate_memory(session)
        finally:
            self._consolidating.discard(session.key)
            self._prune_consolidation_lock(session.key, lock)

    _task = asyncio.create_task(_consolidate_and_unlock())
    self._consolidation_tasks.add(_task)
```

**触发条件**:
- 未归档消息数 ≥ memory_window (默认 100)
- 该会话未在进行归档

**执行方式**:
- 后台异步执行
- 不阻塞主消息处理
- 使用锁防止并发归档

### `consolidate()` 方法详解

**位置**: `memory.py:69-150`

这是最核心的方法，负责将对话历史压缩为结构化记忆。

```python
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

    Args:
        session: 会话对象
        provider: LLM 提供商
        model: 模型名称
        archive_all: 是否归档所有消息（用于 /new 命令）
        memory_window: 记忆窗口大小

    Returns:
        成功返回 True（包括无操作），失败返回 False
    """
```

#### 步骤 1: 确定要归档的消息

```python
if archive_all:
    # /new 命令：归档所有消息
    old_messages = session.messages
    keep_count = 0
    logger.info("Memory consolidation (archive_all): {} messages", len(session.messages))
else:
    # 自动归档：保留最近一半的消息
    keep_count = memory_window // 2  # 25 条

    # 检查是否需要归档
    if len(session.messages) <= keep_count:
        return True  # 消息太少，无需归档

    if len(session.messages) - session.last_consolidated <= 0:
        return True  # 没有新消息需要归档

    # 归档 last_consolidated 到 -keep_count 之间的消息
    old_messages = session.messages[session.last_consolidated:-keep_count]
    if not old_messages:
        return True

    logger.info("Memory consolidation: {} to consolidate, {} keep",
                len(old_messages), keep_count)
```

**归档范围示意图**:

```
假设 memory_window = 50, keep_count = 25

Session.messages (100 条):
┌────────────────────────────────────────────────────────┐
│ [0] [1] ... [49] [50] ... [74] [75] ... [98] [99]      │
│                      ↑                        ↑          │
│           last_consolidated=50           最近25条       │
│                      │                        │          │
│                   归档范围               保留          │
│               [50-74] 共25条                          │
└────────────────────────────────────────────────────────┘

归档后:
- last_consolidated = 75
- LLM 只看到 [75-99] 共25条消息
- [0-74] 的内容已压缩到 MEMORY.md 和 HISTORY.md
```

#### 步骤 2: 格式化消息为文本

```python
lines = []
for m in old_messages:
    if not m.get("content"):
        continue

    # 添加工具使用信息
    tools = f" [tools: {', '.join(m['tools_used'])}]" if m.get("tools_used") else ""

    # 格式化为时间线格式
    lines.append(f"[{m.get('timestamp', '?')[:16]}] {m['role'].upper()}{tools}: {m['content']}")
```

**输出示例**:

```
[2026-02-28 18:30] USER: 帮我读取 README.md 文件
[2026-02-28 18:30] ASSISTANT [tools: read_file]: 好的，让我读取 README.md 文件。
[2026-02-28 18:30] TOOL: # README.md

This is a project description...
[2026-02-28 18:31] ASSISTANT: 我已经读取了 README.md 文件...
```

#### 步骤 3: 构建归档提示词

```python
# 读取当前长期记忆
current_memory = self.read_long_term()

# 构建提示词
prompt = f"""Process this conversation and call the save_memory tool with your consolidation.

## Current Long-term Memory
{current_memory or "(empty)"}

## Conversation to Process
{chr(10).join(lines)}"""
```

**提示词结构**:

```
Process this conversation and call the save_memory tool with your consolidation.

## Current Long-term Memory
## User Information
(现有的用户信息)

## Preferences
(现有的偏好设置)

...

## Conversation to Process
[2026-02-28 18:30] USER: 帮我读取 README.md 文件
[2026-02-28 18:30] ASSISTANT [tools: read_file]: 好的，让我读取 README.md 文件。
...
```

#### 步骤 4: 调用 LLM 生成摘要

```python
# 定义 save_memory 工具
_SAVE_MEMORY_TOOL = [
    {
        "type": "function",
        "function": {
            "name": "save_memory",
            "description": "Save the memory consolidation result to persistent storage.",
            "parameters": {
                "type": "object",
                "properties": {
                    "history_entry": {
                        "type": "string",
                        "description": "A paragraph (2-5 sentences) summarizing key events/decisions/topics. "
                                    "Start with [YYYY-MM-DD HH:MM]. Include detail useful for grep search.",
                    },
                    "memory_update": {
                        "type": "string",
                        "description": "Full updated long-term memory as markdown. Include all existing "
                                    "facts plus new ones. Return unchanged if nothing new.",
                    },
                },
                "required": ["history_entry", "memory_update"],
            },
        },
    }
]

# 调用 LLM
response = await provider.chat(
    messages=[
        {
            "role": "system",
            "content": "You are a memory consolidation agent. Call the save_memory tool with your consolidation of the conversation."
        },
        {"role": "user", "content": prompt},
    ],
    tools=_SAVE_MEMORY_TOOL,
    model=model,
)
```

**关键点**:
- 使用专门的 LLM 调用进行归档
- 提供 `save_memory` 工具
- 系统提示词明确角色为 "memory consolidation agent"

#### 步骤 5: 处理 LLM 响应

```python
# 检查工具调用
if not response.has_tool_calls:
    logger.warning("Memory consolidation: LLM did not call save_memory, skipping")
    return False

# 提取参数
args = response.tool_calls[0].arguments

# 处理参数类型差异
if isinstance(args, str):
    args = json.loads(args)
if not isinstance(args, dict):
    logger.warning("Memory consolidation: unexpected arguments type {}", type(args).__name__)
    return False

# 保存历史条目
if entry := args.get("history_entry"):
    if not isinstance(entry, str):
        entry = json.dumps(entry, ensure_ascii=False)
    self.append_history(entry)

# 更新长期记忆
if update := args.get("memory_update"):
    if not isinstance(update, str):
        update = json.dumps(update, ensure_ascii=False)
    if update != current_memory:
        self.write_long_term(update)

# 更新归档标记
session.last_consolidated = 0 if archive_all else len(session.messages) - keep_count
logger.info("Memory consolidation done: {} messages, last_consolidated={}",
            len(session.messages), session.last_consolidated)
return True
```

---

## 📊 归档工具详解

### `save_memory` 工具定义

```json
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
          "description": "摘要段落（2-5 句），总结关键事件/决策/主题。"
                      "以 [YYYY-MM-DD HH:MM] 开头。包含对 grep 搜索有用的细节。"
        },
        "memory_update": {
          "type": "string",
          "description": "完整更新的长期记忆（markdown 格式）。"
                      "包含所有现有事实加上新事实。如果没有新内容则返回不变。"
        }
      },
      "required": ["history_entry", "memory_update"]
    }
  }
}
```

### 参数说明

#### `history_entry` - 历史条目

**目的**: 记录时间线事件，便于 grep 搜索

**格式**:
```
[YYYY-MM-DD HH:MM] 关键事件的详细描述，包括谁做了什么、为什么、结果如何。
```

**示例**:
```
[2026-02-28 18:30] User John requested help reading README.md file.
Assistant used read_file tool to retrieve the file contents and provided a summary of the project description and installation instructions.
```

#### `memory_update` - 记忆更新

**目的**: 更新结构化的长期记忆

**格式**: Markdown

**示例**:
```markdown
# Long-term Memory

## User Information
- **Name**: John
- **First Contact**: 2026-02-28
- **Interest**: Learning about nanobot project

## Preferences
- Likes detailed explanations
- Prefers step-by-step guides

## Project Context
- User is exploring the nanobot codebase
- Asked about README.md contents
- Interested in understanding the architecture

## Important Notes
- User's first interaction with the system
```

---

## 🎯 归档流程图

### 完整流程

```
触发归档
    │
    ▼
┌─────────────────────────────────────┐
│ 1. 确定归档范围                     │
│    - archive_all: 所有消息          │
│    - 自动: memory_window // 2       │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 2. 格式化消息为文本                 │
│    - 时间戳 + 角色 + 内容            │
│    - 标记工具使用                    │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 3. 读取当前记忆                     │
│    - MEMORY.md 内容                 │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 4. 构建归档提示词                   │
│    - 当前记忆 + 待处理对话            │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 5. 调用 LLM 生成摘要                 │
│    - 使用 save_memory 工具           │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 6. 处理 LLM 响应                    │
│    - 提取 history_entry             │
│    - 提取 memory_update             │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ 7. 保存归档结果                     │
│    - 追加到 HISTORY.md               │
│    - 更新 MEMORY.md                 │
│    - 更新 session.last_consolidated  │
└─────────────────────────────────────┘
```

---

## 📁 文件结构

### 目录结构

```
workspace/
├── memory/
│   ├── MEMORY.md       # 长期记忆（结构化事实）
│   └── HISTORY.md      # 历史日志（时间线）
└── sessions/
    └── telegram_123456.jsonl  # 会话文件
```

### MEMORY.md 示例

```markdown
# Long-term Memory

## User Information
- **Name**: Alice
- **Email**: alice@example.com
- **Role**: Software Developer

## Preferences
- Prefers concise answers
- Likes code examples
- Uses Python 3.12

## Project Context
- Working on nanobot integration
- Interested in agent architecture
- Focus on tool system

## Important Notes
- First interaction: 2026-02-25
- Primary use case: personal assistant
- Preferred channel: Telegram

---

*This file is automatically updated by nanobot when important information should be remembered.*
```

### HISTORY.md 示例

```markdown
[2026-02-25 10:00] User Alice introduced herself as a software developer interested in nanobot. Asked about the agent architecture and tool system. Provided overview of the codebase structure and explained the ReAct pattern used in AgentLoop.

[2026-02-25 14:30] Alice requested help understanding the tool registration system. Explained ToolRegistry, Tool base class, and parameter validation mechanism. Provided code examples for creating custom tools.

[2026-02-26 09:15] User asked about memory consolidation feature. Explained the two-layer memory system (MEMORY.md + HISTORY.md) and how the consolidation process works using LLM to compress conversation history.

[2026-02-28 18:30] User requested help reading README.md file. Used read_file tool to retrieve the file contents and provided a summary of the project description and installation instructions.

```

---

## 🔍 关键设计决策

### 1. 两层记忆架构

**为什么分离 MEMORY 和 HISTORY？**

| 类型 | MEMORY.md | HISTORY.md |
|------|-----------|------------|
| **内容** | 结构化事实 | 时间线日志 |
| **格式** | Markdown（分类） | 纯文本（时间顺序） |
| **用途** | LLM 上下文注入 | grep 搜索 |
| **更新** | 完全覆盖 | 追加 |
| **大小** | 较小（精简） | 较大（详细） |

**优势**:
- MEMORY.md: 快速注入到系统提示词
- HISTORY.md: 便于搜索历史事件

### 2. Append-Only 设计

**为什么使用 `last_consolidated` 标记？**

```
Session.messages: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
                               ↑
                        last_consolidated = 6

LLM 看到: [6, 7, 8, 9] (未归档的消息)
已归档: [0, 1, 2, 3, 4, 5]
```

**优势**:
- 消息列表不变（LLM 缓存友好）
- 通过标记控制可见范围
- 避免复制或删除数据

### 3. 异步后台归档

**为什么在后台执行？**

```python
# 创建后台任务
_task = asyncio.create_task(_consolidate_and_unlock())
```

**优势**:
- 不阻塞主消息处理
- 用户立即得到响应
- 提高系统吞吐量

### 4. 保留窗口策略

**为什么保留最近 25 条消息？**

```python
keep_count = memory_window // 2  # 25 条
```

**原因**:
- 提供足够的上下文连续性
- 避免丢失最近的对话线索
- 平衡记忆和性能

---

## 🔒 并发控制

### 会话级别锁

```python
def _get_consolidation_lock(self, session_key: str) -> asyncio.Lock:
    """获取会话的归档锁"""
    lock = self._consolidation_locks.get(session_key)
    if lock is None:
        lock = asyncio.Lock()
        self._consolidation_locks[session_key] = lock
    return lock
```

**用途**:
- 同一会话只允许一个归档任务
- 防止并发写入 MEMORY.md
- 避免归档状态混乱

### 归档状态追踪

```python
self._consolidating: set[str] = set()  # 正在归档的会话
```

**用途**:
- 防止重复归档
- 标记正在处理的会话

---

## 📈 性能考虑

### Token 节省

**归档前**:
```
Session.messages: 100 条
LLM 输入: 100 条消息 ≈ 10,000 tokens
```

**归档后**:
```
Session.messages: 100 条
last_consolidated: 75
LLM 输入: 25 条消息 ≈ 2,500 tokens
MEMORY.md 注入: ≈ 500 tokens
总计: ≈ 3,000 tokens

节省: 7,000 tokens (70%)
```

### I/O 优化

```python
# HISTORY.md 追加写入（高效）
with open(self.history_file, "a", encoding="utf-8") as f:
    f.write(entry.rstrip() + "\n\n")

# MEMORY.md 覆盖写入（较少）
self.memory_file.write_text(content, encoding="utf-8")
```

---

## 🧪 测试场景

### 场景 1: 首次归档

**输入**:
```python
session.messages = [
    {"role": "user", "content": "Hello", "timestamp": "2026-02-28T18:00:00"},
    {"role": "assistant", "content": "Hi!", "timestamp": "2026-02-28T18:00:05"},
    # ... 50 条消息
]
session.last_consolidated = 0
```

**预期输出**:
- HISTORY.md 追加新的历史条目
- MEMORY.md 创建并更新
- `session.last_consolidated = 25`

### 场景 2: 增量归档

**输入**:
```python
session.messages = 100 条消息
session.last_consolidated = 50  # 前 50 条已归档
```

**归档范围**:
- 归档 [50-74] 共 25 条
- 保留 [75-99] 共 25 条
- 更新 `last_consolidated = 75`

### 场景 3: 全量归档 (/new 命令)

**输入**:
```python
archive_all = True
session.messages = 100 条消息
```

**归档范围**:
- 归档所有 100 条消息
- `last_consolidated = 0` (重置)

---

## 🎓 最佳实践

### 1. 定期归档

```python
# 当消息数超过窗口时自动归档
if unconsolidated >= self.memory_window:
    await self._consolidate_memory(session)
```

### 2. 手动归档

```python
# /new 命令
await self._consolidate_memory(session, archive_all=True)
session.clear()
```

### 3. 记忆注入

```python
# 在系统提示词中注入长期记忆
memory_context = self.memory.get_memory_context()
system_prompt = f"{identity}\n\n{memory_context}"
```

### 4. 搜索历史

```bash
# 使用 grep 搜索历史事件
grep "README.md" workspace/memory/HISTORY.md
```

---

## 🔍 调试技巧

### 1. 查看归档日志

```python
logger.info("Memory consolidation: {} to consolidate, {} keep",
            len(old_messages), keep_count)
```

### 2. 检查归档状态

```python
print(f"Total messages: {len(session.messages)}")
print(f"Last consolidated: {session.last_consolidated}")
print(f"Unconsolidated: {len(session.messages) - session.last_consolidated}")
```

### 3. 验证记忆文件

```bash
# 查看长期记忆
cat workspace/memory/MEMORY.md

# 查看历史日志
cat workspace/memory/HISTORY.md

# 搜索特定事件
grep "Alice" workspace/memory/HISTORY.md
```

---

## 📖 相关文件

- `agent/memory.py` - 记忆系统实现
- `agent/loop.py:440-445` - 归档调用
- `agent/context.py:51-53` - 记忆注入
- `session/manager.py:32` - `last_consolidated` 字段
- `templates/memory/MEMORY.md` - 记忆模板

---

## 🎓 学习要点

### 必须理解

1. **两层记忆**: MEMORY.md (结构化) + HISTORY.md (时间线)
2. **归档流程**: 确定范围 → 格式化 → LLM 处理 → 保存结果
3. **标记机制**: `last_consolidated` 标记已归档位置
4. **异步处理**: 后台执行，不阻塞主流程
5. **Token 节省**: 归档减少 LLM 输入大小
6. **并发控制**: 会话级别锁防止冲突

### 代码阅读路径

1. 从 `MemoryStore.consolidate()` 开始（核心方法）
2. 理解归档范围计算
3. 学习 LLM 工具调用机制
4. 研究文件 I/O 操作
5. 理解 `last_consolidated` 的作用

---

## 🚀 扩展方向

### 可能的改进

1. **智能归档**: 根据消息重要性决定是否归档
2. **分类记忆**: 按主题分类存储记忆
3. **记忆检索**: 实现 RAG (Retrieval Augmented Generation)
4. **记忆过期**: 自动清理过时的记忆
5. **多会话记忆**: 跨会话共享记忆

---

## 📝 总结

Memory 系统是 nanobot 的长期记忆管理核心，通过两层架构（MEMORY.md + HISTORY.md）实现：

- 📚 **结构化记忆**: 存储用户信息、偏好、项目上下文
- 📝 **时间线日志**: 记录关键事件和决策
- 🗜️ **智能归档**: 使用 LLM 压缩对话历史
- ⚡️ **性能优化**: 减少 token 使用，提高响应速度
- 🔒 **并发安全**: 会话级别锁防止冲突

理解 Memory 系统对于构建持久化、个性化的 AI 助手至关重要！
