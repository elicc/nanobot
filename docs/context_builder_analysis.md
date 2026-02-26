# ContextBuilder 上下文构建器深度解析

## 概述

**ContextBuilder** 负责构建 LLM 的完整提示词，是 Agent 和 LLM 之间的桥梁。

**文件**: `nanobot/agent/context.py` (254 行)

**核心职责**:
1. 构建系统提示词
2. 组装完整消息列表（系统提示词 + 历史对话 + 当前消息）
3. 管理工具调用消息
4. 处理多媒体附件

---

## 🏗️ 架构设计

### 类结构

```python
class ContextBuilder:
    """
    构建代理的上下文（系统提示词 + 消息）

    将引导文件、记忆、技能和对话历史组装成连贯的 LLM 提示词
    """

    BOOTSTRAP_FILES = [
        "AGENTS.md",    # 代理定义
        "SOUL.md",      # 灵魂/人格
        "USER.md",      # 用户偏好
        "TOOLS.md",     # 工具使用指南
        "IDENTITY.md"   # 身份定义
    ]

    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.memory = MemoryStore(workspace)      # 记忆存储
        self.skills = SkillsLoader(workspace)      # 技能加载器
```

### 依赖组件

```
ContextBuilder
    ├── MemoryStore      # 长期记忆管理
    ├── SkillsLoader     # 技能加载器
    └── Bootstrap Files  # 引导文件（工作区根目录）
```

---

## 🔄 核心方法详解

### 1. `build_system_prompt()` - 构建系统提示词

**位置**: `context.py:30-73`

这是最核心的方法，负责构建完整的系统提示词。

```python
def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
    """
    从引导文件、记忆和技能构建系统提示词

    Args:
        skill_names: 可选的要包含的技能列表

    Returns:
        完整的系统提示词
    """
    parts = []

    # 1. 核心身份
    parts.append(self._get_identity())

    # 2. 引导文件
    bootstrap = self._load_bootstrap_files()
    if bootstrap:
        parts.append(bootstrap)

    # 3. 记忆上下文
    memory = self.memory.get_memory_context()
    if memory:
        parts.append(f"# Memory\n\n{memory}")

    # 4. 技能 - 渐进式加载
    # 4.1 始终加载的技能：包含完整内容
    always_skills = self.skills.get_always_skills()
    if always_skills:
        always_content = self.skills.load_skills_for_context(always_skills)
        if always_content:
            parts.append(f"# Active Skills\n\n{always_content}")

    # 4.2 可用技能：仅显示摘要（代理使用 read_file 工具加载）
    skills_summary = self.skills.build_skills_summary()
    if skills_summary:
        parts.append(f"""# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first - you can try installing them with apt/brew.

{skills_summary}""")

    return "\n\n---\n\n".join(parts)
```

#### 系统提示词结构

```
┌────────────────────────────────────────┐
│ 1. 核心身份 (_get_identity)            │
│    - nanobot 简介                      │
│    - 运行时信息                        │
│    - 工作区路径                        │
│    - 工具使用指南                      │
└────────────┬───────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ 2. 引导文件 (_load_bootstrap_files)    │
│    - AGENTS.md                         │
│    - SOUL.md                           │
│    - USER.md                           │
│    - TOOLS.md                          │
│    - IDENTITY.md                       │
└────────────┬───────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ 3. 记忆上下文 (MemoryStore)            │
│    - 长期记忆内容                      │
└────────────┬───────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ 4. 技能 (SkillsLoader)                 │
│    4.1 Active Skills (always=true)     │
│        - 完整技能内容                  │
│    4.2 Skills Summary                  │
│        - 技能列表和描述                │
│        - 可用性状态                    │
└────────────────────────────────────────┘
```

---

### 2. `_get_identity()` - 核心身份

**位置**: `context.py:75-105`

```python
def _get_identity(self) -> str:
    """获取核心身份部分"""
    workspace_path = str(self.workspace.expanduser().resolve())
    system = platform.system()
    runtime = f"{'macOS' if system == 'Darwin' else system} {platform.machine()}, Python {platform.python_version()}"

    return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant.

## Runtime
{runtime}

## Workspace
Your workspace is at: {workspace_path}
- Long-term memory: {workspace_path}/memory/MEMORY.md
- History log: {workspace_path}/memory/HISTORY.md (grep-searchable)
- Custom skills: {workspace_path}/skills/{{skill-name}}/SKILL.md

Reply directly with text for conversations. Only use the 'message' tool to send to a specific chat channel.

## Tool Call Guidelines
- Before calling tools, you may briefly state your intent (e.g. "Let me check that"), but NEVER predict or describe the expected result before receiving it.
- Before modifying a file, read it first to confirm its current content.
- Do not assume a file or directory exists — use list_dir or read_file to verify.
- After writing or editing a file, re-read it if accuracy matters.
- If a tool call fails, analyze the error before retrying with a different approach.

## Memory
- Remember important facts: write to {workspace_path}/memory/MEMORY.md
- Recall past events: grep {workspace_path}/memory/HISTORY.md"""
```

**关键信息**:
- **Runtime**: 运行环境（操作系统、架构、Python 版本）
- **Workspace**: 工作区路径和关键文件位置
- **Tool Call Guidelines**: 工具调用最佳实践
- **Memory**: 记忆管理指南

---

### 3. `_load_bootstrap_files()` - 加载引导文件

**位置**: `context.py:124-134`

```python
def _load_bootstrap_files(self) -> str:
    """从工作区加载所有引导文件"""
    parts = []

    for filename in self.BOOTSTRAP_FILES:
        file_path = self.workspace / filename
        if file_path.exists():
            content = file_path.read_text(encoding="utf-8")
            parts.append(f"## {filename}\n\n{content}")

    return "\n\n".join(parts) if parts else ""
```

**引导文件说明**:

| 文件 | 用途 | 示例内容 |
|------|------|----------|
| `AGENTS.md` | 定义代理角色和行为模式 | "你是一个代码助手..." |
| `SOUL.md` | 定义人格、语气、价值观 | "友好、专业、乐于助人..." |
| `USER.md` | 用户偏好和自定义指令 | "我喜欢简洁的回答..." |
| `TOOLS.md` | 工具使用指南和最佳实践 | "如何有效使用文件操作..." |
| `IDENTITY.md` | 身份和职责定义 | "你是 nanobot AI 助手..." |

**优先级**: 工作区文件 > 内置默认

---

### 4. `build_messages()` - 构建完整消息列表

**位置**: `context.py:136-173`

这是 AgentLoop 调用的主要方法，构建 LLM API 需要的完整消息列表。

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
    """
    构建 LLM 调用的完整消息列表

    Args:
        history: 之前的对话消息
        current_message: 新的用户消息
        skill_names: 可选的要包含的技能
        media: 可选的图片/媒体文件本地路径列表
        channel: 当前渠道（telegram、feishu 等）
        chat_id: 当前聊天/用户 ID

    Returns:
        包括系统提示词的消息列表
    """
    messages = []

    # 1. 系统提示词
    system_prompt = self.build_system_prompt(skill_names)
    messages.append({"role": "system", "content": system_prompt})

    # 2. 历史对话
    messages.extend(history)

    # 3. 当前消息（可选附加图片）
    user_content = self._build_user_content(current_message, media)
    user_content = self._inject_runtime_context(user_content, channel, chat_id)
    messages.append({"role": "user", "content": user_content})

    return messages
```

**消息结构**:

```python
[
    {
        "role": "system",
        "content": "完整的系统提示词..."
    },
    # ... 历史消息
    {
        "role": "user",
        "content": "用户消息内容"
    },
    {
        "role": "assistant",
        "content": "AI 回复",
        "tool_calls": [...]  # 可选：工具调用
    },
    {
        "role": "tool",
        "tool_call_id": "...",
        "name": "tool_name",
        "content": "工具执行结果"
    },
    # ...
]
```

---

### 5. `_build_user_content()` - 构建用户内容

**位置**: `context.py:175-191`

```python
def _build_user_content(self, text: str, media: list[str] | None) -> str | list[dict[str, Any]]:
    """
    构建包含可选 base64 编码图片的用户消息内容
    """
    if not media:
        return text

    images = []
    for path in media:
        p = Path(path)
        mime, _ = mimetypes.guess_type(path)
        if not p.is_file() or not mime or not mime.startswith("image/"):
            continue
        b64 = base64.b64encode(p.read_bytes()).decode()
        images.append({
            "type": "image_url",
            "image_url": {"url": f"data:{mime};base64,{b64}"}
        })

    if not images:
        return text
    return images + [{"type": "text", "text": text}]
```

**多模态支持**:

```python
# 纯文本
"Hello, how are you?"

# 文本 + 图片
[
    {
        "type": "image_url",
        "image_url": {"url": "data:image/png;base64,iVBORw0KG..."}
    },
    {
        "type": "text",
        "text": "What's in this image?"
    }
]
```

---

### 6. `_inject_runtime_context()` - 注入运行时上下文

**位置**: `context.py:108-122`

```python
@staticmethod
def _inject_runtime_context(
    user_content: str | list[dict[str, Any]],
    channel: str | None,
    chat_id: str | None,
) -> str | list[dict[str, Any]]:
    """
    在用户消息末尾附加动态运行时上下文
    """
    now = datetime.now().strftime("%Y-%m-%d %H:%M (%A)")
    tz = time.strftime("%Z") or "UTC"
    lines = [f"Current Time: {now} ({tz})"]
    if channel and chat_id:
        lines += [f"Channel: {channel}", f"Chat ID: {chat_id}"]
    block = "[Runtime Context]\n" + "\n".join(lines)

    if isinstance(user_content, str):
        return f"{user_content}\n\n{block}"
    return [*user_content, {"type": "text", "text": block}]
```

**运行时上下文示例**:

```
[Runtime Context]
Current Time: 2026-02-25 18:30 (Tuesday)
Channel: telegram
Chat ID: 123456789
```

**用途**:
- 让 AI 知道当前时间
- 让 AI 知道消息来源
- 帮助 AI 个性化响应

---

### 7. `add_assistant_message()` - 添加助手消息

**位置**: `context.py:220-253`

```python
def add_assistant_message(
    self,
    messages: list[dict[str, Any]],
    content: str | None,
    tool_calls: list[dict[str, Any]] | None = None,
    reasoning_content: str | None = None,
) -> list[dict[str, Any]]:
    """
    添加助手消息到消息列表

    Args:
        messages: 当前消息列表
        content: 消息内容
        tool_calls: 可选的工具调用
        reasoning_content: 思考输出（Kimi、DeepSeek-R1 等）

    Returns:
        更新后的消息列表
    """
    msg: dict[str, Any] = {"role": "assistant"}

    # 始终包含 content —— 某些提供商（如 StepFun）会拒绝
    # 完全省略该键的助手消息
    msg["content"] = content

    if tool_calls:
        msg["tool_calls"] = tool_calls

    # 提供时包含推理内容（某些思考模型需要）
    if reasoning_content is not None:
        msg["reasoning_content"] = reasoning_content

    messages.append(msg)
    return messages
```

**支持的思考模型**:
- DeepSeek-R1
- Kimi (月之暗面)
- 其他支持 `reasoning_content` 的模型

---

### 8. `add_tool_result()` - 添加工具结果

**位置**: `context.py:193-218`

```python
def add_tool_result(
    self,
    messages: list[dict[str, Any]],
    tool_call_id: str,
    tool_name: str,
    result: str
) -> list[dict[str, Any]]:
    """
    添加工具结果到消息列表

    Args:
        messages: 当前消息列表
        tool_call_id: 工具调用的 ID
        tool_name: 工具名称
        result: 工具执行结果

    Returns:
        更新后的消息列表
    """
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call_id,
        "name": tool_name,
        "content": result
    })
    return messages
```

---

## 🎨 技能系统 (SkillsLoader)

**文件**: `nanobot/agent/skills.py` (229 行)

### 技能结构

```
skills/
├── skill-name/
│   ├── SKILL.md           # 技能说明（必需）
│   ├── requirements.txt   # Python 依赖（可选）
│   └── ...                # 其他资源
```

### SKILL.md 格式

```markdown
---
description: "技能的简短描述"
always: true              # 可选：始终加载
requires:
  bins:
    - ffmpeg               # 需要的命令行工具
  env:
    - OPENAI_API_KEY       # 需要的环境变量
metadata: '{"nanobot": {...}}'  # nanobot 元数据（JSON）
---

# 技能标题

详细的技能说明...

## 使用示例

...
```

### 核心方法

#### 1. `list_skills()` - 列出所有技能

**位置**: `skills.py:26-57`

```python
def list_skills(self, filter_unavailable: bool = True) -> list[dict[str, str]]:
    """
    列出所有可用技能

    Args:
        filter_unavailable: 如果为 True，过滤掉不满足要求的技能

    Returns:
        技能信息字典列表，包含 'name', 'path', 'source'
    """
    skills = []

    # 工作区技能（最高优先级）
    if self.workspace_skills.exists():
        for skill_dir in self.workspace_skills.iterdir():
            if skill_dir.is_dir():
                skill_file = skill_dir / "SKILL.md"
                if skill_file.exists():
                    skills.append({
                        "name": skill_dir.name,
                        "path": str(skill_file),
                        "source": "workspace"
                    })

    # 内置技能
    if self.builtin_skills and self.builtin_skills.exists():
        for skill_dir in self.builtin_skills.iterdir():
            if skill_dir.is_dir():
                skill_file = skill_dir / "SKILL.md"
                if skill_file.exists() and not any(s["name"] == skill_dir.name for s in skills):
                    skills.append({
                        "name": skill_dir.name,
                        "path": str(skill_file),
                        "source": "builtin"
                    })

    # 按要求过滤
    if filter_unavailable:
        return [s for s in skills if self._check_requirements(self._get_skill_meta(s["name"]))]
    return skills
```

**优先级**: 工作区技能 > 内置技能

#### 2. `load_skill()` - 加载技能

**位置**: `skills.py:59-80`

```python
def load_skill(self, name: str) -> str | None:
    """
    按名称加载技能

    Args:
        name: 技能名称（目录名）

    Returns:
        技能内容，如果未找到则返回 None
    """
    # 首先检查工作区
    workspace_skill = self.workspace_skills / name / "SKILL.md"
    if workspace_skill.exists():
        return workspace_skill.read_text(encoding="utf-8")

    # 检查内置
    if self.builtin_skills:
        builtin_skill = self.builtin_skills / name / "SKILL.md"
        if builtin_skill.exists():
            return builtin_skill.read_text(encoding="utf-8")

    return None
```

#### 3. `build_skills_summary()` - 构建技能摘要

**位置**: `skills.py:101-140`

```python
def build_skills_summary(self) -> str:
    """
    构建所有技能的摘要（名称、描述、路径、可用性）

    用于渐进式加载 —— 代理可以在需要时使用 read_file
    读取完整的技能内容

    Returns:
        XML 格式的技能摘要
    """
    all_skills = self.list_skills(filter_unavailable=False)
    if not all_skills:
        return ""

    lines = ["<skills>"]
    for s in all_skills:
        name = escape_xml(s["name"])
        path = s["path"]
        desc = escape_xml(self._get_skill_description(s["name"]))
        skill_meta = self._get_skill_meta(s["name"])
        available = self._check_requirements(skill_meta)

        lines.append(f'  <skill available="{str(available).lower()}">')
        lines.append(f"    <name>{name}</name>")
        lines.append(f"    <description>{desc}</description>")
        lines.append(f"    <location>{path}</location>")

        # 显示不可用技能的缺失要求
        if not available:
            missing = self._get_missing_requirements(skill_meta)
            if missing:
                lines.append(f"    <requires>{escape_xml(missing)}</requires>")

        lines.append(f"  </skill>")
    lines.append("</skills>")

    return "\n".join(lines)
```

**输出示例**:

```xml
<skills>
  <skill available="true">
    <name>image-analysis</name>
    <description>Image analysis and OCR capabilities</description>
    <location>/workspace/skills/image-analysis/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>video-editing</name>
    <description>Video editing and processing</description>
    <location>/workspace/skills/video-editing/SKILL.md</location>
    <requires>CLI: ffmpeg</requires>
  </skill>
</skills>
```

#### 4. `get_always_skills()` - 获取始终加载的技能

**位置**: `skills.py:193-201`

```python
def get_always_skills(self) -> list[str]:
    """获取标记为 always=true 且满足要求的技能"""
    result = []
    for s in self.list_skills(filter_unavailable=True):
        meta = self.get_skill_metadata(s["name"]) or {}
        skill_meta = self._parse_nanobot_metadata(meta.get("metadata", ""))
        if skill_meta.get("always") or meta.get("always"):
            result.append(s["name"])
    return result
```

---

## 📊 完整数据流

```
AgentLoop._process_message()
    │
    ▼
ContextBuilder.build_messages()
    │
    ├─▶ build_system_prompt()
    │   │
    │   ├─▶ _get_identity()
    │   │   └─▶ 运行时信息、工作区路径、工具指南
    │   │
    │   ├─▶ _load_bootstrap_files()
    │   │   └─▶ AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md
    │   │
    │   ├─▶ MemoryStore.get_memory_context()
    │   │   └─▶ 长期记忆内容
    │   │
    │   └─▶ SkillsLoader
    │       ├─▶ get_always_skills() → 始终加载的技能（完整内容）
    │       └─▶ build_skills_summary() → 所有技能摘要
    │
    ├─▶ history（会话历史）
    │
    └─▶ current_message（当前用户消息）
        ├─▶ _build_user_content()（处理图片）
        └─▶ _inject_runtime_context()（时间、渠道信息）
    │
    ▼
[完整消息列表]
    │
    ├─▶ {"role": "system", "content": "系统提示词"}
    ├─▶ {"role": "user", "content": "历史消息 1"}
    ├─▶ {"role": "assistant", "content": "历史回复 1"}
    ├─▶ ...
    └─▶ {"role": "user", "content": "当前消息 + 运行时上下文"}
```

---

## 🎯 设计模式和原则

### 1. 组合模式 (Composite Pattern)

```python
class ContextBuilder:
    def __init__(self, workspace: Path):
        self.memory = MemoryStore(workspace)
        self.skills = SkillsLoader(workspace)
```

将多个组件组合在一起构建复杂对象。

### 2. 模板方法模式 (Template Method Pattern)

```python
def build_system_prompt(self) -> str:
    parts = []
    parts.append(self._get_identity())          # 步骤 1
    parts.append(self._load_bootstrap_files())  # 步骤 2
    parts.append(self.memory.get_memory_context())  # 步骤 3
    parts.append(self._build_skills())          # 步骤 4
    return "\n\n---\n\n".join(parts)
```

定义算法骨架，子步骤由不同方法实现。

### 3. 渐进式加载 (Progressive Loading)

```python
# 始终加载的技能：完整内容
always_skills = self.skills.get_always_skills()
always_content = self.skills.load_skills_for_context(always_skills)

# 其他技能：仅摘要
skills_summary = self.skills.build_skills_summary()
```

**优势**:
- 减少初始提示词大小
- 按需加载技能内容
- 节省 token 使用

### 4. 优先级系统

```
工作区技能 > 内置技能
```

允许用户覆盖默认技能。

---

## 🚀 性能优化

### 1. 延迟加载技能

```python
# 不在系统提示词中包含所有技能
# 而是提供摘要，让 AI 按需读取
skills_summary = self.skills.build_skills_summary()
```

### 2. 条件包含

```python
memory = self.memory.get_memory_context()
if memory:
    parts.append(f"# Memory\n\n{memory}")
```

只在有内容时添加部分。

### 3. 缓存机制

虽然代码中没有显式缓存，但：
- MemoryStore 内部可能缓存记忆
- SkillsLoader 的文件读取可以被缓存

---

## 🧪 测试建议

### 单元测试

1. **系统提示词构建**:
   ```python
   def test_build_system_prompt():
       builder = ContextBuilder(workspace)
       prompt = builder.build_system_prompt()
       assert "nanobot" in prompt
       assert "Workspace" in prompt
   ```

2. **技能加载**:
   ```python
   def test_load_skill():
       skills = SkillsLoader(workspace)
       content = skills.load_skill("test-skill")
       assert content is not None
   ```

3. **依赖检查**:
   ```python
   def test_skill_requirements():
       skills = SkillsLoader(workspace)
       available = skills._check_requirements({
           "bins": ["ls"],
           "env": ["HOME"]
       })
       assert available is True
   ```

### 集成测试

1. **完整消息构建**:
   ```python
   def test_build_messages():
       builder = ContextBuilder(workspace)
       messages = builder.build_messages(
           history=[],
           current_message="Hello",
           channel="telegram",
           chat_id="123"
       )
       assert len(messages) == 2  # system + user
       assert "Runtime Context" in messages[1]["content"]
   ```

2. **多模态支持**:
   ```python
   def test_build_user_content_with_images():
       builder = ContextBuilder(workspace)
       content = builder._build_user_content(
           "What's this?",
           media=["/path/to/image.png"]
       )
       assert isinstance(content, list)
       assert any("image_url" in str(item) for item in content)
   ```

---

## 🔍 关键概念总结

### 1. 引导文件 (Bootstrap Files)

工作区根目录的 Markdown 文件，用于自定义 AI 的行为：
- **AGENTS.md**: 定义代理角色
- **SOUL.md**: 定义人格
- **USER.md**: 用户偏好
- **TOOLS.md**: 工具使用指南
- **IDENTITY.md**: 身份定义

### 2. 技能系统 (Skills)

可加载的能力模块，通过 SKILL.md 文件定义：
- 支持元数据（描述、依赖）
- 支持始终加载标记 (`always: true`)
- 渐进式加载（摘要 vs 完整内容）
- 依赖检查（命令行工具、环境变量）

### 3. 记忆系统 (Memory)

长期记忆存储，通过 MemoryStore 管理：
- 全局记忆
- 会话特定记忆
- 自动归档

### 4. 运行时上下文

动态注入到用户消息的上下文信息：
- 当前时间
- 渠道信息
- 聊天 ID

### 5. 多模态支持

支持在消息中包含图片：
- Base64 编码
- MIME 类型检测
- OpenAI 格式兼容

---

## 📖 扩展阅读

### 相关文件

- `agent/memory.py` - 记忆存储实现
- `agent/skills.py` - 技能加载器
- `session/manager.py` - 会话管理

### 设计模式

- **Composite Pattern**: 组合多个组件
- **Template Method**: 模板方法
- **Strategy Pattern**: 策略模式（不同的技能加载策略）

---

## 🎓 学习要点

### 必须理解

1. **系统提示词结构**: 身份 → 引导文件 → 记忆 → 技能
2. **技能渐进式加载**: 始终加载 vs 按需加载
3. **多模态支持**: 文本 + 图片
4. **运行时上下文**: 动态注入时间、渠道信息
5. **优先级系统**: 工作区 > 内置

### 代码阅读路径

1. 从 `build_messages()` 开始（主要入口）
2. 进入 `build_system_prompt()`（系统提示词）
3. 研究各个辅助方法（身份、引导文件、技能）
4. 理解 SkillsLoader 的技能管理

---

## 🚀 下一步

现在你已经深入理解了 ContextBuilder！

想继续学习哪个模块？
- ToolRegistry - 工具系统
- SessionManager - 会话管理
- MemoryStore - 记忆系统
- Channel Adapter - 渠道集成
