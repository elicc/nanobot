# AgentLoop 核心逻辑深度解析

## 概述

**AgentLoop** 是 nanobot 的心脏，负责处理所有消息并协调 LLM、工具、会话和记忆。

**文件**: `nanobot/agent/loop.py` (459 行)

---

## 🏗️ 架构设计

### 类结构

```python
class AgentLoop:
    """
    核心处理引擎

    职责：
    1. 从消息总线接收消息
    2. 构建上下文（历史、记忆、技能）
    3. 调用 LLM
    4. 执行工具调用
    5. 发送响应
    """

    def __init__(self, bus, provider, workspace, ...):
        # 依赖注入
        self.bus = bus                    # 消息总线
        self.provider = provider          # LLM 提供商
        self.workspace = workspace        # 工作空间

        # 组件
        self.context = ContextBuilder(workspace)      # 上下文构建器
        self.sessions = SessionManager(workspace)     # 会话管理器
        self.tools = ToolRegistry()                  # 工具注册表
        self.subagents = SubagentManager(...)         # 子代理管理器

        # 配置
        self.model = model
        self.max_iterations = max_iterations    # 最大工具调用次数
        self.temperature = temperature
        self.max_tokens = max_tokens
        self.memory_window = memory_window      # 记忆窗口大小
```

---

## 🔄 核心方法详解

### 1. `run()` - 主循环

**位置**: `loop.py:240-269`

```python
async def run(self) -> None:
    """运行代理循环，处理来自总线的消息"""
    self._running = True
    await self._connect_mcp()  # 连接 MCP 服务器
    logger.info("Agent loop started")

    while self._running:
        try:
            # 1. 从消息队列获取消息（超时 1 秒）
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )

            # 2. 处理消息
            try:
                response = await self._process_message(msg)

                # 3. 发送响应
                if response is not None:
                    await self.bus.publish_outbound(response)
                # CLI 模式下，即使没有响应也要发送空消息（解除阻塞）
                elif msg.channel == "cli":
                    await self.bus.publish_outbound(OutboundMessage(
                        channel=msg.channel,
                        chat_id=msg.chat_id,
                        content="",
                        metadata=msg.metadata or {},
                    ))

            # 4. 错误处理
            except Exception as e:
                logger.error("Error processing message: {}", e)
                await self.bus.publish_outbound(OutboundMessage(
                    channel=msg.channel,
                    chat_id=msg.chat_id,
                    content=f"Sorry, I encountered an error: {str(e)}"
                ))

        except asyncio.TimeoutError:
            # 超时继续循环
            continue
```

**关键点**:
- 无限循环处理消息
- 超时机制允许检查 `_running` 标志
- 错误捕获保证单个消息错误不会终止整个服务
- CLI 特殊处理：必须发送响应解除阻塞

---

### 2. `_process_message()` - 消息处理

**位置**: `loop.py:296-423`

这是最核心的方法，处理单个消息的完整流程。

#### 2.1 会话管理

```python
# 从消息中获取会话键
key = session_key or msg.session_key

# 获取或创建会话
session = self.sessions.get_or_create(key)
```

**Session 键的格式**: `"channel:chat_id"`
- 例如: `"telegram:123456"`, `"discord:789012"`, `"cli:direct"`

#### 2.2 斜杠命令

```python
cmd = msg.content.strip().lower()

# /new: 开始新对话
if cmd == "/new":
    # 1. 归档当前会话记忆
    if not await self._consolidate_memory(session, archive_all=True):
        return OutboundMessage(
            content="Memory archival failed, session not cleared. Please try again."
        )

    # 2. 清空会话
    session.clear()

    # 3. 保存并失效
    self.sessions.save(session)
    self.sessions.invalidate(session.key)

    return OutboundMessage(content="New session started.")

# /help: 显示帮助
if cmd == "/help":
    return OutboundMessage(
        content="🐈 nanobot commands:\n/new — Start a new conversation\n/help — Show available commands"
    )
```

#### 2.3 自动记忆归档

```python
unconsolidated = len(session.messages) - session.last_consolidated

# 当未归档消息超过窗口大小时，触发后台归档
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

**机制**:
- 当会话积累太多消息时，自动在后台压缩旧对话
- 使用锁防止并发归档
- 不阻塞当前消息处理

#### 2.4 构建上下文

```python
# 设置工具上下文（路由信息）
self._set_tool_context(msg.channel, msg.chat_id, msg.metadata.get("message_id"))

# 获取历史消息
history = session.get_history(max_messages=self.memory_window)

# 构建完整的 LLM 消息列表
initial_messages = self.context.build_messages(
    history=history,
    current_message=msg.content,
    media=msg.media if msg.media else None,
    channel=msg.channel,
    chat_id=msg.chat_id,
)
```

**上下文包含**:
- 系统提示词
- 技能定义
- 历史对话
- 当前用户消息
- 附件（如果有）

#### 2.5 运行代理循环

```python
# 定义进度回调（流式输出）
async def _bus_progress(content: str, *, tool_hint: bool = False) -> None:
    meta = dict(msg.metadata or {})
    meta["_progress"] = True
    meta["_tool_hint"] = tool_hint
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=content,
        metadata=meta,
    ))

# 运行 LLM + 工具调用循环
final_content, _, all_msgs = await self._run_agent_loop(
    initial_messages,
    on_progress=on_progress or _bus_progress,
)
```

#### 2.6 保存会话

```python
# 保存新消息到会话
self._save_turn(session, all_msgs, 1 + len(history))

# 持久化会话
self.sessions.save(session)

# 如果工具已经发送了消息，不需要重复发送
if message_tool := self.tools.get("message"):
    if isinstance(message_tool, MessageTool) and message_tool._sent_in_turn:
        return None  # 不发送额外响应

return OutboundMessage(
    channel=msg.channel,
    chat_id=msg.chat_id,
    content=final_content,
    metadata=msg.metadata or {},
)
```

---

### 3. `_run_agent_loop()` - LLM + 工具循环

**位置**: `loop.py:174-238`

这是实现 ReAct (Reasoning + Acting) 模式的核心。

```python
async def _run_agent_loop(
    self,
    initial_messages: list[dict],
    on_progress: Callable[..., Awaitable[None]] | None = None,
) -> tuple[str | None, list[str], list[dict]]:
    """
    运行代理迭代循环

    返回: (最终内容, 使用的工具, 所有消息)
    """
    messages = initial_messages
    iteration = 0
    final_content = None
    tools_used: list[str] = []

    while iteration < self.max_iterations:
        iteration += 1

        # 1. 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),  # 工具定义
            model=self.model,
            temperature=self.temperature,
            max_tokens=self.max_tokens,
        )

        # 2. 检查是否有工具调用
        if response.has_tool_calls:
            # 2.1 发送进度（LLM 的思考内容）
            if on_progress:
                clean = self._strip_think(response.content)
                if clean:
                    await on_progress(clean)

                # 2.2 发送工具调用提示
                await on_progress(
                    self._tool_hint(response.tool_calls),
                    tool_hint=True
                )

            # 2.3 添加助手消息到历史
            tool_call_dicts = [
                {
                    "id": tc.id,
                    "type": "function",
                    "function": {
                        "name": tc.name,
                        "arguments": json.dumps(tc.arguments, ensure_ascii=False)
                    }
                }
                for tc in response.tool_calls
            ]
            messages = self.context.add_assistant_message(
                messages,
                response.content,
                tool_call_dicts,
                reasoning_content=response.reasoning_content,
            )

            # 2.4 执行所有工具调用
            for tool_call in response.tool_calls:
                tools_used.append(tool_call.name)
                args_str = json.dumps(tool_call.arguments, ensure_ascii=False)
                logger.info("Tool call: {}({})", tool_call.name, args_str[:200])

                # 执行工具
                result = await self.tools.execute(
                    tool_call.name,
                    tool_call.arguments
                )

                # 添加工具结果到历史
                messages = self.context.add_tool_result(
                    messages,
                    tool_call.id,
                    tool_call.name,
                    result
                )

        # 3. 没有工具调用，获取最终响应
        else:
            final_content = self._strip_think(response.content)
            break

    # 4. 达到最大迭代次数
    if final_content is None and iteration >= self.max_iterations:
        logger.warning("Max iterations ({}) reached", self.max_iterations)
        final_content = (
            f"I reached the maximum number of tool call iterations ({self.max_iterations}) "
            "without completing the task. You can try breaking the task into smaller steps."
        )

    return final_content, tools_used, messages
```

**流程图**:

```
┌─────────────────────────────────────┐
│ 初始化: messages = initial_messages  │
└────────────┬────────────────────────┘
             │
             ▼
       ┌─────┴─────┐
       │ iteration │
       │ < max     │
       └─────┬─────┘
             │ Yes
             ▼
    ┌────────────────┐
    │ 调用 LLM       │
    │ (messages +    │
    │  tools)        │
    └────────┬───────┘
             │
             ▼
       ┌─────┴─────┐
       │ 有工具调用? │
       └─────┬─────┘
             │
      ┌──────┴──────┐
      │             │
     Yes           No
      │             │
      ▼             ▼
 ┌────────┐    ┌────────┐
 │执行工具 │    │返回内容 │
 │添加结果 │    │  break │
 └───┬────┘    └────────┘
     │
     └──────▶ (继续循环)
```

**关键点**:
- 迭代直到没有工具调用或达到最大次数
- 每次迭代都包含完整的 LLM 调用和工具执行
- 工具结果被添加到消息历史
- 支持流式输出（通过 `on_progress` 回调）

---

### 4. 辅助方法

#### `_save_turn()` - 保存对话轮次

**位置**: `loop.py:427-438`

```python
def _save_turn(self, session: Session, messages: list[dict], skip: int) -> None:
    """
    保存新轮次的消息到会话，截断大型工具结果
    """
    from datetime import datetime

    for m in messages[skip:]:
        entry = {k: v for k, v in m.items() if k != "reasoning_content"}

        # 截断大型工具结果（节省存储）
        if entry.get("role") == "tool" and isinstance(entry.get("content"), str):
            content = entry["content"]
            if len(content) > self._TOOL_RESULT_MAX_CHARS:  # 500 字符
                entry["content"] = content[:self._TOOL_RESULT_MAX_CHARS] + "\n... (truncated)"

        entry.setdefault("timestamp", datetime.now().isoformat())
        session.messages.append(entry)

    session.updated_at = datetime.now()
```

#### `_strip_think()` - 去除思考块

**位置**: `loop.py:157-162`

```python
@staticmethod
def _strip_think(text: str | None) -> str | None:
    """去除某些模型嵌入在内容中的思考块"""
    if not text:
        return None
    return re.sub(r"<thinking>[\s\S]*?</thinking>", "", text).strip() or None
```

**用途**: 处理某些模型（如 DeepSeek-R1）在响应中嵌入的思考标签

#### `_tool_hint()` - 格式化工具提示

**位置**: `loop.py:164-172`

```python
@staticmethod
def _tool_hint(tool_calls: list) -> str:
    """
    格式化工具调用为简洁提示，例如 'web_search("query")'
    """
    def _fmt(tc):
        val = next(iter(tc.arguments.values()), None) if tc.arguments else None
        if not isinstance(val, str):
            return tc.name
        return f'{tc.name}("{val[:40]}…")' if len(val) > 40 else f'{tc.name}("{val}")'

    return ", ".join(_fmt(tc) for tc in tool_calls)
```

**示例**:
- 输入: `[ToolCall(name="web_search", arguments={"query": "latest AI news"})]`
- 输出: `web_search("latest AI news")`

---

## 🎨 设计模式和原则

### 1. 依赖注入 (Dependency Injection)

```python
def __init__(self, bus: MessageBus, provider: LLMProvider, ...):
    self.bus = bus
    self.provider = provider
```

**优点**:
- 易于测试（可以注入 mock 对象）
- 松耦合
- 灵活配置

### 2. 单一职责原则 (Single Responsibility)

每个方法只做一件事:
- `run()`: 消息循环
- `_process_message()`: 处理单条消息
- `_run_agent_loop()`: LLM + 工具循环
- `_save_turn()`: 保存消息

### 3. 策略模式 (Strategy Pattern)

不同的 LLM 提供商可以互换：
```python
self.provider = provider  # 可以是任何 LLMProvider 实现
```

### 4. 观察者模式 (Observer Pattern)

通过回调函数实现进度通知：
```python
async def _bus_progress(content: str, ...):
    await self.bus.publish_outbound(...)
```

### 5. 异步编程 (Async/Await)

全面使用 `async/await`：
```python
await self.bus.consume_inbound()
await self._process_message(msg)
await self.provider.chat(...)
await self.tools.execute(...)
```

**优点**:
- 高并发处理
- 非阻塞 I/O
- 资源高效利用

---

## 🔒 并发控制

### 记忆归档锁

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
- 防止同一会话的并发归档
- 使用会话键作为锁的标识

### 归档状态追踪

```python
self._consolidating: set[str] = set()  # 正在归档的会话
```

**用途**:
- 防止重复归档
- 标记正在处理的会话

---

## 📊 关键数据结构

### Session (会话)

```python
class Session:
    key: str                    # "channel:chat_id"
    messages: list[dict]        # 消息历史
    created_at: datetime        # 创建时间
    updated_at: datetime        # 更新时间
    last_consolidated: int      # 最后归档位置
```

### 消息格式

```python
{
    "role": "user" | "assistant" | "system" | "tool",
    "content": str,
    "timestamp": str,
    # 可选字段
    "tool_calls": list[dict],   # 助手消息
    "tool_call_id": str,        # 工具消息
    "name": str,                # 工具名称
}
```

---

## 🎯 核心概念总结

### 1. 会话管理

每个用户对话对应一个 Session，通过 `session_key` 标识：
- 格式: `"channel:chat_id"`
- 持久化到磁盘
- 自动归档旧消息

### 2. 记忆窗口

```python
self.memory_window = 100  # 只保留最近 100 条消息
```

**机制**:
- LLM 只看到最近 N 条消息
- 旧消息被压缩成摘要
- 节省 token 使用

### 3. 工具调用循环

ReAct 模式：
1. **Reason**: LLM 决定调用哪个工具
2. **Act**: 执行工具
3. **Observe**: 将结果反馈给 LLM
4. **Repeat**: 直到任务完成

### 4. 流式输出

通过回调函数实现：
```python
async def _bus_progress(content: str, *, tool_hint: bool = False):
    await self.bus.publish_outbound(OutboundMessage(
        content=content,
        metadata={"_progress": True, "_tool_hint": tool_hint}
    ))
```

**用途**:
- 实时显示 AI 思考过程
- 显示工具调用提示
- 改善用户体验

---

## 🚀 性能优化

### 1. 后台归档

```python
_task = asyncio.create_task(_consolidate_and_unlock())
```

不阻塞主消息处理流程

### 2. 消息截断

```python
if len(content) > self._TOOL_RESULT_MAX_CHARS:
    entry["content"] = content[:500] + "\n... (truncated)"
```

节省存储空间

### 3. 超时机制

```python
msg = await asyncio.wait_for(
    self.bus.consume_inbound(),
    timeout=1.0
)
```

允许定期检查 `_running` 标志

---

## 🧪 测试建议

### 单元测试

1. **消息处理**: 测试 `_process_message()` 的各种场景
2. **工具调用**: 测试 `_run_agent_loop()` 的迭代逻辑
3. **错误处理**: 测试异常情况的处理
4. **会话管理**: 测试会话创建、更新、归档

### 集成测试

1. **端到端**: 发送消息，验证响应
2. **并发**: 多个消息同时处理
3. **持久化**: 重启后恢复会话

---

## 📚 相关文件

- `agent/context.py` - 上下文构建
- `agent/tools/registry.py` - 工具注册表
- `session/manager.py` - 会话管理
- `agent/memory.py` - 记忆存储
- `bus/queue.py` - 消息总线
- `providers/base.py` - LLM 提供商基类

---

## 🎓 学习要点

### 必须理解

1. **消息循环**: `run()` 如何持续处理消息
2. **会话管理**: 如何管理用户对话历史
3. **LLM 循环**: `_run_agent_loop()` 如何实现 ReAct
4. **工具执行**: 如何注册、查找、执行工具
5. **上下文构建**: 如何构建完整的 LLM 提示词
6. **记忆归档**: 如何自动归档旧对话

### 代码阅读路径

1. 从 `run()` 开始（主循环）
2. 进入 `_process_message()`（消息处理）
3. 深入 `_run_agent_loop()`（LLM 循环）
4. 研究辅助方法（保存、格式化等）
5. 理解会话和记忆管理

---

## 🔍 调试技巧

### 日志

```python
logger.info("Processing message from {}:{}", msg.channel, msg.sender_id)
logger.info("Tool call: {}({})", tool_call.name, args_str)
logger.warning("Max iterations ({}) reached", self.max_iterations)
```

### 断点

关键位置:
- `_process_message()` 开始
- `_run_agent_loop()` 循环内
- 工具执行前后

---

## 📖 扩展阅读

- **ReAct 论文**: "ReAct: Synergizing Reasoning and Acting in Language Models"
- **Agent 设计模式**: LangChain Agents, AutoGPT
- **异步编程**: Python asyncio 文档

---

现在你已经深入理解了 AgentLoop 的核心逻辑！

想继续学习哪个模块？
- ContextBuilder - 上下文构建
- ToolRegistry - 工具系统
- SessionManager - 会话管理
- MemoryStore - 记忆系统
- Channel Adapter - 渠道集成
