# 数据流追踪 - 实战分析

## 概述

本文档通过一个完整的实战场景，追踪从用户发送消息到 AI 响应的完整数据流和代码执行流程。

---

## 🎬 场景设定

**用户**: 通过 Telegram 发送消息 "帮我读取 README.md 文件"

**预期行为**:
1. 接收 Telegram 消息
2. 构建 LLM 上下文
3. LLM 决定调用 `read_file` 工具
4. 执行工具读取文件
5. 将结果返回给 LLM
6. LLM 生成最终回复
7. 发送回复到 Telegram

---

## 📍 代码位置索引

| 组件 | 文件位置 | 关键方法 |
|------|----------|----------|
| Telegram 适配器 | `channels/telegram.py` | `run()`, `_process_update()` |
| 消息总线 | `bus/queue.py` | `publish_inbound()`, `consume_inbound()` |
| 代理循环 | `agent/loop.py` | `run()`, `_process_message()`, `_run_agent_loop()` |
| 上下文构建 | `agent/context.py` | `build_messages()` |
| 会话管理 | `session/manager.py` | `get_or_create()`, `save()` |
| 工具注册表 | `agent/tools/registry.py` | `execute()` |
| 文件工具 | `agent/tools/filesystem.py` | `ReadFileTool.execute()` |

---

## 🔄 完整数据流

### 阶段 1: 接收消息 (Telegram → MessageBus)

```
Telegram 服务器
    │
    ▼
channels/telegram.py:run()
    │
    ├─▶ 接收 update
    │
    ▼
channels/telegram.py:_process_update()
    │
    ├─▶ 解析消息
    ├─▶ 提取文本内容
    ├─▶ 提取元数据
    │
    ▼
创建 InboundMessage
    │
    ├─▶ channel = "telegram"
    ├─▶ sender_id = "123456789"
    ├─▶ chat_id = "123456789"
    ├─▶ content = "帮我读取 README.md 文件"
    ├─▶ session_key = "telegram:123456789"
    │
    ▼
bus/queue.py:publish_inbound()
    │
    └─▶ await self.inbound.put(msg)
```

**代码追踪**:

**`channels/telegram.py:_process_update()`**

```python
async def _process_update(self, update: dict):
    """处理 Telegram 更新"""
    # 提取消息
    message = update.get("message", {})
    text = message.get("text", "")
    sender_id = str(message.get("from", {}).get("id", ""))
    chat_id = str(message.get("chat", {}).get("id", ""))

    # 创建入站消息
    msg = InboundMessage(
        channel="telegram",
        sender_id=sender_id,
        chat_id=chat_id,
        content=text,
        metadata={"message_id": message.get("message_id")}
    )

    # 发布到消息总线
    await self.bus.publish_inbound(msg)
```

---

### 阶段 2: 消息路由 (MessageBus → AgentLoop)

```
MessageBus Inbound Queue
    │
    ▼
agent/loop.py:run()
    │
    ├─▶ await self.bus.consume_inbound()
    │
    ▼
获取 InboundMessage
    │
    ▼
agent/loop.py:_process_message()
    │
    ├─▶ 提取 session_key = "telegram:123456789"
    │
    ▼
session/manager.py:get_or_create()
    │
    ├─▶ 检查缓存 → 未命中
    ├─▶ _load(key) → 未找到
    ├─▶ 创建新 Session(key="telegram:123456789")
    ├─▶ 缓存会话
    │
    ▼
返回 Session 对象
```

**代码追踪**:

**`agent/loop.py:run()`**

```python
async def run(self) -> None:
    """运行代理循环"""
    self._running = True
    await self._connect_mcp()

    while self._running:
        try:
            # 从消息队列获取消息
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )

            # 处理消息
            try:
                response = await self._process_message(msg)
                if response is not None:
                    await self.bus.publish_outbound(response)
            except Exception as e:
                logger.error("Error processing message: {}", e)
        except asyncio.TimeoutError:
            continue
```

---

### 阶段 3: 构建上下文 (Session → ContextBuilder)

```
Session 对象
    │
    ├─▶ key = "telegram:123456789"
    ├─▶ messages = []
    └─▶ last_consolidated = 0
    │
    ▼
agent/loop.py:_process_message()
    │
    ├─▶ history = session.get_history(max_messages=100)
    │   └─▶ 返回 [] (新会话)
    │
    ├─▶ initial_messages = self.context.build_messages(
    │       history=[],
    │       current_message="帮我读取 README.md 文件",
    │       channel="telegram",
    │       chat_id="123456789"
    │   )
    │
    ▼
agent/context.py:build_messages()
    │
    ├─▶ build_system_prompt()
    │   ├─▶ _get_identity()
    │   ├─▶ _load_bootstrap_files()
    │   ├─▶ memory.get_memory_context()
    │   └─▶ skills.build_skills_summary()
    │
    ├─▶ 添加历史消息
    │
    ├─▶ _build_user_content()
    │   └─▶ "帮我读取 README.md 文件"
    │
    ├─▶ _inject_runtime_context()
    │   └─▶ 附加时间和渠道信息
    │
    ▼
返回完整消息列表
```

**代码追踪**:

**`agent/context.py:build_messages()`**

```python
def build_messages(
    self,
    history: list[dict[str, Any]],
    current_message: str,
    skill_names: list[str] | None = None,
    media: list[str] | None = None,
    channel: str | None = None,
    chat_id: str | None = None,
) -> list[dict[str, Any]]:
    """构建 LLM 调用的完整消息列表"""
    messages = []

    # 1. 系统提示词
    system_prompt = self.build_system_prompt(skill_names)
    messages.append({"role": "system", "content": system_prompt})

    # 2. 历史对话
    messages.extend(history)

    # 3. 当前消息
    user_content = self._build_user_content(current_message, media)
    user_content = self._inject_runtime_context(user_content, channel, chat_id)
    messages.append({"role": "user", "content": user_content})

    return messages
```

**构建的 messages**:

```python
[
    {
        "role": "system",
        "content": """
# nanobot 🐈

You are nanobot, a helpful AI assistant.

## Runtime
macOS arm64, Python 3.12.0

## Workspace
Your workspace is at: /Users/user/workspace
...

# Skills
<skills>
  <skill available="true">
    <name>filesystem</name>
    <description>File system operations</description>
    ...
  </skill>
</skills>
"""
    },
    {
        "role": "user",
        "content": """
帮我读取 README.md 文件

[Runtime Context]
Current Time: 2026-02-28 18:30 (Tuesday)
Channel: telegram
Chat ID: 123456789
"""
    }
]
```

---

### 阶段 4: LLM 调用 (ContextBuilder → LLM Provider)

```
完整消息列表
    │
    ▼
agent/loop.py:_run_agent_loop()
    │
    ├─▶ messages = initial_messages
    ├─▶ iteration = 0
    │
    ▼
调用 LLM
    │
    ▼
providers/base.py:chat()
    │
    ├─▶ 构建请求
    ├─▶ 添加工具定义
    │   └─▶ tools = [
    │       {"type": "function", "function": {"name": "read_file", ...}},
    │       {"type": "function", "function": {"name": "write_file", ...}},
    │       ...
    │   ]
    │
    ├─▶ 发送到 LLM API
    │
    ▼
等待 LLM 响应
    │
    ▼
LLM 返回响应
    │
    ├─▶ content = "好的，让我读取 README.md 文件。"
    ├─▶ has_tool_calls = True
    ├─▶ tool_calls = [
    │       {
    │           "id": "call_123",
    │           "name": "read_file",
    │           "arguments": {"path": "README.md"}
    │       }
    │   ]
```

**代码追踪**:

**`agent/loop.py:_run_agent_loop()`**

```python
async def _run_agent_loop(
    self,
    initial_messages: list[dict],
    on_progress: Callable | None = None,
) -> tuple[str | None, list[str], list[dict]]:
    """运行代理迭代循环"""
    messages = initial_messages
    iteration = 0
    final_content = None
    tools_used: list[str] = []

    while iteration < self.max_iterations:
        iteration += 1

        # 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),  # 工具定义
            model=self.model,
            temperature=self.temperature,
            max_tokens=self.max_tokens,
        )

        # 检查工具调用
        if response.has_tool_calls:
            # ... 执行工具
        else:
            final_content = response.content
            break

    return final_content, tools_used, messages
```

---

### 阶段 5: 工具执行 (LLM → ToolRegistry → ReadFileTool)

```
LLM 响应
    │
    ├─▶ tool_calls = [
    │       {"id": "call_123", "name": "read_file", "arguments": {"path": "README.md"}}
    │   ]
    │
    ▼
agent/loop.py:_run_agent_loop()
    │
    ├─▶ 发送进度（可选）
    │   └─▶ await on_progress("好的，让我读取 README.md 文件。")
    │
    ├─▶ 添加助手消息到历史
    │   └─▶ messages = context.add_assistant_message(
    │           messages,
    │           "好的，让我读取 README.md 文件。",
    │           tool_call_dicts=[...]
    │       )
    │
    ▼
执行工具
    │
    ▼
tools/registry.py:execute()
    │
    ├─▶ tool = self._tools.get("read_file")
    │   └─▶ 返回 ReadFileTool 实例
    │
    ├─▶ 验证参数
    │   └─▶ errors = tool.validate_params({"path": "README.md"})
    │       └─▶ 返回 [] (验证通过)
    │
    ├─▶ result = await tool.execute(path="README.md")
    │   │
    │   ▼
    │ tools/filesystem.py:ReadFileTool.execute()
    │   │
    │   ├─▶ file_path = _resolve_path("README.md", workspace, allowed_dir)
    │   │   └─▶ /Users/user/workspace/README.md
    │   │
    │   ├─▶ 检查文件是否存在
    │   │   └─▶ file_path.exists() → True
    │   │
    │   ├─▶ 读取文件内容
    │   │   └─▶ content = file_path.read_text(encoding="utf-8")
    │   │
    │   └─▶ 返回文件内容
    │
    ▼
返回工具结果
    │
    ▼
添加工具结果到历史
    │
    └─▶ messages = context.add_tool_result(
            messages,
            "call_123",
            "read_file",
            "# README.md\n\n这是项目说明文件..."
        )
```

**代码追踪**:

**`agent/tools/registry.py:execute()`**

```python
async def execute(self, name: str, params: dict[str, Any]) -> str:
    """执行工具"""
    _HINT = "\n\n[Analyze the error above and try a different approach.]"

    # 1. 获取工具
    tool = self._tools.get(name)
    if not tool:
        return f"Error: Tool '{name}' not found..."

    # 2. 验证参数
    errors = tool.validate_params(params)
    if errors:
        return f"Error: Invalid parameters: " + "; ".join(errors) + _HINT

    # 3. 执行工具
    result = await tool.execute(**params)

    # 4. 错误提示
    if isinstance(result, str) and result.startswith("Error"):
        return result + _HINT
    return result
```

**`agent/tools/filesystem.py:ReadFileTool.execute()`**

```python
async def execute(self, path: str, **kwargs: Any) -> str:
    """执行文件读取"""
    try:
        # 解析路径
        file_path = _resolve_path(path, self._workspace, self._allowed_dir)

        # 检查文件
        if not file_path.exists():
            return f"Error: File not found: {path}"
        if not file_path.is_file():
            return f"Error: Not a file: {path}"

        # 读取内容
        content = file_path.read_text(encoding="utf-8")
        return content
    except PermissionError as e:
        return f"Error: {e}"
    except Exception as e:
        return f"Error reading file: {str(e)}"
```

---

### 阶段 6: 继续对话 (Tool Result → LLM → Final Response)

```
工具结果已添加到 messages
    │
    ▼
messages 状态:
    [
        {"role": "system", "content": "..."},
        {"role": "user", "content": "帮我读取 README.md 文件"},
        {
            "role": "assistant",
            "content": "好的，让我读取 README.md 文件。",
            "tool_calls": [
                {"id": "call_123", "type": "function", "function": {"name": "read_file", "arguments": "{\"path\": \"README.md\"}"}}
            ]
        },
        {
            "role": "tool",
            "tool_call_id": "call_123",
            "name": "read_file",
            "content": "# README.md\n\n这是项目说明文件..."
        }
    ]
    │
    ▼
agent/loop.py:_run_agent_loop()
    │
    ├─▶ 继续循环 (iteration = 2)
    │
    ├─▶ 再次调用 LLM
    │   └─▶ response = await provider.chat(messages, tools=...)
    │
    ▼
LLM 返回最终响应
    │
    ├─▶ content = "我已经读取了 README.md 文件。这是一个项目的说明文档，包含了项目介绍、安装步骤和使用说明。"
    ├─▶ has_tool_calls = False
    │
    ▼
退出循环
    │
    ▼
返回 (final_content, tools_used, messages)
    │
    ├─▶ final_content = "我已经读取了 README.md 文件..."
    ├─▶ tools_used = ["read_file"]
    └─▶ messages = [...]
```

---

### 阶段 7: 保存会话 (AgentLoop → SessionManager)

```
返回到 _process_message()
    │
    ├─▶ final_content = "我已经读取了 README.md 文件..."
    ├─▶ all_msgs = messages (完整的消息列表)
    │
    ▼
agent/loop.py:_save_turn()
    │
    ├─▶ 遍历新消息
    ├─▶ 截断大型工具结果
    ├─▶ 添加时间戳
    ├─▶ 追加到 session.messages
    │
    ▼
session/manager.py:save()
    │
    ├─▶ 序列化为 JSONL
    │
    ▼
写入磁盘
    │
    └─▶ workspace/sessions/telegram_123456789.jsonl
```

**代码追踪**:

**`agent/loop.py:_save_turn()`**

```python
def _save_turn(self, session: Session, messages: list[dict], skip: int) -> None:
    """保存新轮次的消息到会话"""
    from datetime import datetime

    for m in messages[skip:]:
        entry = {k: v for k, v in m.items() if k != "reasoning_content"}

        # 截断大型工具结果
        if entry.get("role") == "tool" and isinstance(entry.get("content"), str):
            content = entry["content"]
            if len(content) > self._TOOL_RESULT_MAX_CHARS:  # 500 字符
                entry["content"] = content[:self._TOOL_RESULT_MAX_CHARS] + "\n... (truncated)"

        entry.setdefault("timestamp", datetime.now().isoformat())
        session.messages.append(entry)

    session.updated_at = datetime.now()
```

**保存的 JSONL 文件**:

```jsonl
{"_type": "metadata", "key": "telegram:123456789", "created_at": "2026-02-28T18:30:00", "updated_at": "2026-02-28T18:31:00", "metadata": {}, "last_consolidated": 0}
{"role": "user", "content": "帮我读取 README.md 文件", "timestamp": "2026-02-28T18:30:00"}
{"role": "assistant", "content": "好的，让我读取 README.md 文件。", "tool_calls": [...], "timestamp": "2026-02-28T18:30:05"}
{"role": "tool", "tool_call_id": "call_123", "name": "read_file", "content": "# README.md\n\n...", "timestamp": "2026-02-28T18:30:06"}
{"role": "assistant", "content": "我已经读取了 README.md 文件...", "timestamp": "2026-02-28T18:30:10"}
```

---

### 阶段 8: 发送响应 (AgentLoop → MessageBus → Telegram)

```
_process_message() 返回
    │
    ▼
创建 OutboundMessage
    │
    ├─▶ channel = "telegram"
    ├─▶ chat_id = "123456789"
    ├─▶ content = "我已经读取了 README.md 文件。这是一个项目的说明文档..."
    ├─▶ metadata = {"message_id": "123"}
    │
    ▼
agent/loop.py:run()
    │
    └─▶ await self.bus.publish_outbound(response)
        │
        ▼
    bus/queue.py:publish_outbound()
        │
        └─▶ await self.outbound.put(msg)
            │
            ▼
        MessageBus Outbound Queue
            │
            ▼
        channels/telegram.py:run()
            │
            └─▶ response = await self.bus.consume_outbound()
                │
                ▼
            channels/telegram.py:_send_response()
                │
                ├─▶ 提取内容
                ├─▶ 构建 Telegram 消息
                │
                ▼
            发送到 Telegram API
                │
                ▼
            用户收到消息
```

**代码追踪**:

**`channels/telegram.py:_send_response()`**

```python
async def _send_response(self, response: OutboundMessage):
    """发送响应到 Telegram"""
    # 提取内容
    content = response.content

    # 如果是进度消息，使用不同的样式
    is_progress = response.metadata.get("_progress", False)
    is_tool_hint = response.metadata.get("_tool_hint", False)

    # 发送消息
    await self.client.send_message(
        chat_id=response.chat_id,
        text=content,
        parse_mode="Markdown"
    )
```

---

## 🎯 完整时序图

```
用户                Telegram         MessageBus       AgentLoop        LLM          ToolRegistry
 │                      │                │              │             │                │
 ├─ 发送消息 ────────▶  │                │              │             │                │
 │                      │                │              │             │                │
 │                      ├─ publish ─────▶ │              │             │                │
 │                      │                │              │             │                │
 │                      │                ├─ consume ──▶ │             │                │
 │                      │                │              │             │                │
 │                      │                │              ├─ build ───▶ │                │
 │                      │                │              │             │                │
 │                      │                │              │             ├─ chat ──────▶ │
 │                      │                │              │             │                │
 │                      │                │              │             │◀─ 响应 ───────┤
 │                      │                │              │             │                │
 │                      │                │              ├─ execute ─────────────────▶ │
 │                      │                │              │             │                │
 │                      │                │              │             │                │ ├─ read_file
 │                      │                │              │             │                │
 │                      │                │              │             │                │◀─ 结果 ────┤
 │                      │                │              │             │                │
 │                      │                │              ├─ chat ───▶ │                │
 │                      │                │              │             │                │
 │                      │                │              │◀─ 最终响应 ──┤                │
 │                      │                │              │             │                │
 │                      │                │ ├─ publish ─▶ │             │                │
 │                      │                │              │             │                │
 │                      │◀───────────────┤              │             │                │
 │                      │  consume       │              │             │                │
 │                      │                │              │             │                │
 │◀─ 收到回复 ──────────┤                │              │             │                │
```

---

## 📊 数据结构变化

### 消息列表的演变

**初始状态**:

```python
messages = [
    {"role": "system", "content": "系统提示词..."},
    {"role": "user", "content": "帮我读取 README.md 文件"}
]
```

**第一次 LLM 调用后**:

```python
messages = [
    {"role": "system", "content": "系统提示词..."},
    {"role": "user", "content": "帮我读取 README.md 文件"},
    {
        "role": "assistant",
        "content": "好的，让我读取 README.md 文件。",
        "tool_calls": [
            {
                "id": "call_123",
                "type": "function",
                "function": {
                    "name": "read_file",
                    "arguments": "{\"path\": \"README.md\"}"
                }
            }
        ]
    }
]
```

**工具执行后**:

```python
messages = [
    {"role": "system", "content": "系统提示词..."},
    {"role": "user", "content": "帮我读取 README.md 文件"},
    {
        "role": "assistant",
        "content": "好的，让我读取 README.md 文件。",
        "tool_calls": [...]
    },
    {
        "role": "tool",
        "tool_call_id": "call_123",
        "name": "read_file",
        "content": "# README.md\n\n这是项目说明文件..."
    }
]
```

**第二次 LLM 调用后**:

```python
messages = [
    {"role": "system", "content": "系统提示词..."},
    {"role": "user", "content": "帮我读取 README.md 文件"},
    {"role": "assistant", "content": "好的，让我读取 README.md 文件。", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "call_123", "name": "read_file", "content": "..."},
    {
        "role": "assistant",
        "content": "我已经读取了 README.md 文件。这是一个项目的说明文档..."
    }
]
```

---

## 🔍 关键观察

### 1. 异步流水线

整个流程完全异步，使用 `async/await`:
- 消息接收不阻塞
- LLM 调用不阻塞
- 工具执行不阻塞
- 支持多个用户并发

### 2. 消息累积

消息列表在每次循环中追加，保留完整上下文:
- 用户消息
- 助手消息
- 工具调用
- 工具结果

### 3. 工具调用循环

```
LLM → 工具调用 → 执行工具 → 添加结果 → LLM → ...
```

直到 LLM 不再调用工具或达到最大迭代次数。

### 4. 状态管理

- **Session**: 持久化到磁盘
- **Memory**: 自动归档旧消息
- **Cache**: 提高性能

### 5. 错误处理

每层都有错误处理:
- 工具执行失败 → 返回错误消息 → LLM 重试
- LLM 调用失败 → 返回错误给用户
- 网络错误 → 重试或降级

---

## 📈 性能分析

### 时间消耗分布

| 阶段 | 预估时间 | 说明 |
|------|----------|------|
| 接收消息 | <10ms | Telegram Bot API |
| 构建上下文 | 10-50ms | 读取文件、加载技能 |
| LLM 调用 (第1次) | 500-2000ms | 取决于模型和输入 |
| 工具执行 | <100ms | 文件 I/O |
| LLM 调用 (第2次) | 500-2000ms | 生成最终回复 |
| 保存会话 | <50ms | 写入磁盘 |
| 发送响应 | <10ms | Telegram Bot API |
| **总计** | **1-4秒** |  |

### 优化点

1. **并行处理**: 多个工具可以并行执行
2. **缓存**: 系统提示词可以缓存
3. **流式输出**: 提前返回部分结果
4. **会话缓存**: 避免重复加载

---

## 🎓 学习要点

### 必须理解

1. **消息流转**: Telegram → MessageBus → AgentLoop → LLM → ToolRegistry
2. **异步执行**: 所有 I/O 操作都是异步的
3. **工具调用循环**: LLM 和工具的交互模式
4. **会话管理**: 消息持久化和历史管理
5. **错误处理**: 每层都有错误处理和重试机制

### 调试技巧

1. **日志追踪**: 使用 logger.info() 跟踪关键步骤
2. **断点**: 在关键方法设置断点
3. **消息检查**: 打印 messages 列表查看上下文
4. **工具日志**: 记录工具调用和结果

---

## 🚀 扩展场景

### 场景 1: 多工具调用

用户: "帮我读取 README.md 和 LICENSE.txt"

流程:
1. LLM 决定调用 `read_file` 两次
2. 两个工具调用可以并行或串行执行
3. 两个结果都添加到消息历史
4. LLM 基于两个文件内容生成回复

### 场景 2: 工具调用失败

用户: "读取不存在的文件.txt"

流程:
1. LLM 调用 `read_file(path="不存在的文件.txt")`
2. 工具返回错误: "Error: File not found: 不存在的文件.txt"
3. 错误消息添加到历史
4. LLM 收到错误，生成用户友好的回复

### 场景 3: 长对话

用户进行多轮对话:

流程:
1. 第一轮: 消息数 = 2 (user + assistant)
2. 第二轮: 消息数 = 4 (user + assistant + tool + assistant)
3. 第三轮: 消息数 = 6
4. ...
5. 当消息数超过 memory_window 时，触发归档

---

## 📝 总结

通过这个实战分析，我们深入理解了 nanobot 的核心工作流程：

1. **消息接收**: 从聊天平台接收并转换为标准格式
2. **上下文构建**: 组装系统提示词、历史、技能
3. **LLM 交互**: 调用 LLM 并处理工具调用
4. **工具执行**: 验证参数、执行工具、返回结果
5. **会话管理**: 保存消息、持久化到磁盘
6. **响应发送**: 发送回复到聊天平台

这个流程是 nanobot 的核心，理解它对于扩展和调试非常重要！

---

## 🎯 下一步

现在你已经完全理解了数据流！

想继续学习什么？
- Channel Adapter - 具体的渠道实现
- Provider System - LLM 提供商集成
- 实战练习 - 添加自定义工具或渠道
