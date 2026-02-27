# ToolRegistry 工具系统深度解析

## 概述

**ToolRegistry** 是 nanobot 的工具系统，负责管理、验证和执行所有工具调用。

**核心文件**:
- `agent/tools/base.py` (103 行) - 工具抽象基类
- `agent/tools/registry.py` (67 行) - 工具注册表
- `agent/tools/*.py` - 具体工具实现

---

## 🏗️ 架构设计

### 核心组件

```
ToolRegistry (工具注册表)
    │
    ├── Tool (抽象基类)
    │   ├── name (工具名称)
    │   ├── description (工具描述)
    │   ├── parameters (JSON Schema)
    │   └── execute() (执行方法)
    │
    └── 具体工具实现
        ├── ReadFileTool
        ├── WriteFileTool
        ├── EditFileTool
        ├── ExecTool
        ├── WebSearchTool
        ├── WebFetchTool
        └── ...
```

### 类关系图

```
          ┌──────────────────┐
          │    Tool (ABC)    │
          └────────┬─────────┘
                   │
       ┌───────────┴───────────┐
       │                       │
   ┌───▼────┐            ┌────▼────┐
   │Registry│            │ 实现 1  │
   └────────┘            └─────────┘
       │                       │
       └──▶ 管理所有    ┌────▼────┐
           Tool 实例   │ 实现 2  │
                       └─────────┘
                            │
                       ┌────▼────┐
                       │ 实现 N  │
                       └─────────┘
```

---

## 🧱 Tool 基类

**文件**: `agent/tools/base.py` (103 行)

### 抽象方法定义

```python
class Tool(ABC):
    """
    代理工具的抽象基类

    工具是代理与环境交互的能力，如读取文件、执行命令等
    """

    # 类型映射表
    _TYPE_MAP = {
        "string": str,
        "integer": int,
        "number": (int, float),
        "boolean": bool,
        "array": list,
        "object": dict,
    }

    @property
    @abstractmethod
    def name(self) -> str:
        """函数调用中使用的工具名称"""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """工具功能的描述"""
        pass

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """工具参数的 JSON Schema"""
        pass

    @abstractmethod
    async def execute(self, **kwargs: Any) -> str:
        """
        使用给定参数执行工具

        Returns:
            工具执行的字符串结果
        """
        pass
```

### 参数验证

```python
def validate_params(self, params: dict[str, Any]) -> list[str]:
    """
    根据 JSON schema 验证工具参数

    Returns:
        错误列表（空列表表示有效）
    """
    schema = self.parameters or {}
    if schema.get("type", "object") != "object":
        raise ValueError(f"Schema must be object type, got {schema.get('type')!r}")
    return self._validate(params, {**schema, "type": "object"}, "")
```

### 递归验证

```python
def _validate(self, val: Any, schema: dict[str, Any], path: str) -> list[str]:
    """递归验证值是否符合 schema"""
    t, label = schema.get("type"), path or "parameter"

    # 1. 类型检查
    if t in self._TYPE_MAP and not isinstance(val, self._TYPE_MAP[t]):
        return [f"{label} should be {t}"]

    errors = []

    # 2. 枚举检查
    if "enum" in schema and val not in schema["enum"]:
        errors.append(f"{label} must be one of {schema['enum']}")

    # 3. 数值范围检查
    if t in ("integer", "number"):
        if "minimum" in schema and val < schema["minimum"]:
            errors.append(f"{label} must be >= {schema['minimum']}")
        if "maximum" in schema and val > schema["maximum"]:
            errors.append(f"{label} must be <= {schema['maximum']}")

    # 4. 字符串长度检查
    if t == "string":
        if "minLength" in schema and len(val) < schema["minLength"]:
            errors.append(f"{label} must be at least {schema['minLength']} chars")
        if "maxLength" in schema and len(val) > schema["maxLength"]:
            errors.append(f"{label} must be at most {schema['maxLength']} chars")

    # 5. 对象属性检查
    if t == "object":
        props = schema.get("properties", {})
        # 必需字段
        for k in schema.get("required", []):
            if k not in val:
                errors.append(f"missing required {path + '.' + k if path else k}")
        # 递归验证每个属性
        for k, v in val.items():
            if k in props:
                errors.extend(self._validate(v, props[k], path + '.' + k if path else k))

    # 6. 数组元素检查
    if t == "array" and "items" in schema:
        for i, item in enumerate(val):
            errors.extend(self._validate(item, schema["items"], f"{path}[{i}]" if path else f"[{i}]"))

    return errors
```

### Schema 转换

```python
def to_schema(self) -> dict[str, Any]:
    """
    转换为 OpenAI 函数 schema 格式

    Returns:
        OpenAI 格式的工具定义
    """
    return {
        "type": "function",
        "function": {
            "name": self.name,
            "description": self.description,
            "parameters": self.parameters,
        }
    }
```

**输出示例**:

```json
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "Read the contents of a file at the given path.",
    "parameters": {
      "type": "object",
      "properties": {
        "path": {
          "type": "string",
          "description": "The file path to read"
        }
      },
      "required": ["path"]
    }
  }
}
```

---

## 📋 ToolRegistry 工具注册表

**文件**: `agent/tools/registry.py` (67 行)

### 类结构

```python
class ToolRegistry:
    """
    代理工具的注册表

    允许动态注册和执行工具
    """

    def __init__(self):
        self._tools: dict[str, Tool] = {}
```

### 核心方法

#### 1. `register()` - 注册工具

```python
def register(self, tool: Tool) -> None:
    """注册工具"""
    self._tools[tool.name] = tool
```

#### 2. `unregister()` - 注销工具

```python
def unregister(self, name: str) -> None:
    """按名称注销工具"""
    self._tools.pop(name, None)
```

#### 3. `get()` - 获取工具

```python
def get(self, name: str) -> Tool | None:
    """按名称获取工具"""
    return self._tools.get(name)
```

#### 4. `get_definitions()` - 获取所有定义

```python
def get_definitions(self) -> list[dict[str, Any]]:
    """
    获取所有工具定义（OpenAI 格式）

    Returns:
        工具 schema 列表
    """
    return [tool.to_schema() for tool in self._tools.values()]
```

#### 5. `execute()` - 执行工具

**位置**: `registry.py:38-55`

```python
async def execute(self, name: str, params: dict[str, Any]) -> str:
    """
    按名称执行工具并传递给定参数

    Args:
        name: 工具名称
        params: 参数字典

    Returns:
        执行结果字符串
    """
    _HINT = "\n\n[Analyze the error above and try a different approach.]"

    # 1. 获取工具
    tool = self._tools.get(name)
    if not tool:
        return f"Error: Tool '{name}' not found. Available: {', '.join(self.tool_names)}"

    try:
        # 2. 验证参数
        errors = tool.validate_params(params)
        if errors:
            return f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors) + _HINT

        # 3. 执行工具
        result = await tool.execute(**params)

        # 4. 错误提示
        if isinstance(result, str) and result.startswith("Error"):
            return result + _HINT
        return result

    except Exception as e:
        return f"Error executing {name}: {str(e)}" + _HINT
```

**执行流程**:

```
execute(name, params)
    │
    ▼
查找工具
    │
    ├─▶ 未找到 → 返回错误 + 可用工具列表
    │
    └─▶ 找到
        │
        ▼
    验证参数
        │
        ├─▶ 验证失败 → 返回错误 + 重试提示
        │
        └─▶ 验证通过
            │
            ▼
        执行工具
            │
            ├─▶ 异常 → 返回错误 + 重试提示
            │
            └─▶ 成功 → 返回结果
```

---

## 🔧 具体工具实现

### 1. ReadFileTool - 读取文件

**文件**: `agent/tools/filesystem.py:24-65`

```python
class ReadFileTool(Tool):
    """读取文件内容的工具"""

    @property
    def name(self) -> str:
        return "read_file"

    @property
    def description(self) -> str:
        return "Read the contents of a file at the given path."

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to read"
                }
            },
            "required": ["path"]
        }

    async def execute(self, path: str, **kwargs: Any) -> str:
        try:
            file_path = _resolve_path(path, self._workspace, self._allowed_dir)
            if not file_path.exists():
                return f"Error: File not found: {path}"
            if not file_path.is_file():
                return f"Error: Not a file: {path}"

            content = file_path.read_text(encoding="utf-8")
            return content
        except PermissionError as e:
            return f"Error: {e}"
        except Exception as e:
            return f"Error reading file: {str(e)}"
```

### 2. WriteFileTool - 写入文件

**文件**: `agent/tools/filesystem.py:68-109`

```python
class WriteFileTool(Tool):
    """写入内容到文件的工具"""

    @property
    def name(self) -> str:
        return "write_file"

    @property
    def description(self) -> str:
        return "Write content to a file at the given path. Creates parent directories if needed."

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to write to"
                },
                "content": {
                    "type": "string",
                    "description": "The content to write"
                }
            },
            "required": ["path", "content"]
        }

    async def execute(self, path: str, content: str, **kwargs: Any) -> str:
        try:
            file_path = _resolve_path(path, self._workspace, self._allowed_dir)
            # 自动创建父目录
            file_path.parent.mkdir(parents=True, exist_ok=True)
            file_path.write_text(content, encoding="utf-8")
            return f"Successfully wrote {len(content)} bytes to {file_path}"
        except PermissionError as e:
            return f"Error: {e}"
        except Exception as e:
            return f"Error writing file: {str(e)}"
```

### 3. EditFileTool - 编辑文件

**文件**: `agent/tools/filesystem.py:112-193`

```python
class EditFileTool(Tool):
    """通过替换文本编辑文件的工具"""

    @property
    def name(self) -> str:
        return "edit_file"

    @property
    def description(self) -> str:
        return "Edit a file by replacing old_text with new_text. The old_text must exist exactly in the file."

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to edit"
                },
                "old_text": {
                    "type": "string",
                    "description": "The exact text to find and replace"
                },
                "new_text": {
                    "type": "string",
                    "description": "The text to replace with"
                }
            },
            "required": ["path", "old_text", "new_text"]
        }

    async def execute(self, path: str, old_text: str, new_text: str, **kwargs: Any) -> str:
        try:
            file_path = _resolve_path(path, self._workspace, self._allowed_dir)
            if not file_path.exists():
                return f"Error: File not found: {path}"

            content = file_path.read_text(encoding="utf-8")

            # 检查旧文本是否存在
            if old_text not in content:
                return self._not_found_message(old_text, content, path)

            # 统计出现次数
            count = content.count(old_text)
            if count > 1:
                return f"Warning: old_text appears {count} times. Please provide more context to make it unique."

            # 执行替换
            new_content = content.replace(old_text, new_text, 1)
            file_path.write_text(new_content, encoding="utf-8")

            return f"Successfully edited {file_path}"
        except PermissionError as e:
            return f"Error: {e}"
        except Exception as e:
            return f"Error editing file: {str(e)}"
```

**智能错误提示**:

```python
@staticmethod
def _not_found_message(old_text: str, content: str, path: str) -> str:
    """构建有用的错误信息"""
    lines = content.splitlines(keepends=True)
    old_lines = old_text.splitlines(keepends=True)
    window = len(old_lines)

    # 寻找最佳匹配
    best_ratio, best_start = 0.0, 0
    for i in range(max(1, len(lines) - window + 1)):
        ratio = difflib.SequenceMatcher(None, old_lines, lines[i : i + window]).ratio()
        if ratio > best_ratio:
            best_ratio, best_start = ratio, i

    # 如果找到相似文本，显示 diff
    if best_ratio > 0.5:
        diff = "\n".join(difflib.unified_diff(
            old_lines, lines[best_start : best_start + window],
            fromfile="old_text (provided)",
            tofile=f"{path} (actual, line {best_start + 1})",
            lineterm="",
        ))
        return f"Error: old_text not found in {path}.\nBest match ({best_ratio:.0%} similar) at line {best_start + 1}:\n{diff}"
    return f"Error: old_text not found in {path}. No similar text found. Verify the file content."
```

### 4. ExecTool - Shell 执行

**文件**: `agent/tools/shell.py:12-151`

```python
class ExecTool(Tool):
    """执行 shell 命令的工具"""

    def __init__(
        self,
        timeout: int = 60,
        working_dir: str | None = None,
        deny_patterns: list[str] | None = None,
        allow_patterns: list[str] | None = None,
        restrict_to_workspace: bool = False,
    ):
        self.timeout = timeout
        self.working_dir = working_dir
        self.deny_patterns = deny_patterns or [
            r"\brm\s+-[rf]{1,2}\b",          # rm -r, rm -rf, rm -fr
            r"\bdel\s+/[fq]\b",              # del /f, del /q
            r"\brmdir\s+/s\b",               # rmdir /s
            r"(?:^|[;&|]\s*)format\b",       # format
            r"\b(mkfs|diskpart)\b",          # disk operations
            r"\bdd\s+if=",                   # dd
            r">\s*/dev/sd",                  # write to disk
            r"\b(shutdown|reboot|poweroff)\b",  # system power
            r":\(\)\s*\{.*\};\s*:",          # fork bomb
        ]
        self.allow_patterns = allow_patterns or []
        self.restrict_to_workspace = restrict_to_workspace

    @property
    def name(self) -> str:
        return "exec"

    @property
    def description(self) -> str:
        return "Execute a shell command and return its output. Use with caution."

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to execute"
                },
                "working_dir": {
                    "type": "string",
                    "description": "Optional working directory for the command"
                }
            },
            "required": ["command"]
        }

    async def execute(self, command: str, working_dir: str | None = None, **kwargs: Any) -> str:
        cwd = working_dir or self.working_dir or os.getcwd()

        # 安全检查
        guard_error = self._guard_command(command, cwd)
        if guard_error:
            return guard_error

        try:
            # 创建子进程
            process = await asyncio.create_subprocess_shell(
                command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                cwd=cwd,
            )

            # 执行命令（带超时）
            try:
                stdout, stderr = await asyncio.wait_for(
                    process.communicate(),
                    timeout=self.timeout
                )
            except asyncio.TimeoutError:
                process.kill()
                try:
                    await asyncio.wait_for(process.wait(), timeout=5.0)
                except asyncio.TimeoutError:
                    pass
                return f"Error: Command timed out after {self.timeout} seconds"

            # 收集输出
            output_parts = []

            if stdout:
                output_parts.append(stdout.decode("utf-8", errors="replace"))

            if stderr:
                stderr_text = stderr.decode("utf-8", errors="replace")
                if stderr_text.strip():
                    output_parts.append(f"STDERR:\n{stderr_text}")

            if process.returncode != 0:
                output_parts.append(f"\nExit code: {process.returncode}")

            result = "\n".join(output_parts) if output_parts else "(no output)"

            # 截断超长输出
            max_len = 10000
            if len(result) > max_len:
                result = result[:max_len] + f"\n... (truncated, {len(result) - max_len} more chars)"

            return result

        except Exception as e:
            return f"Error executing command: {str(e)}"
```

**安全守卫**:

```python
def _guard_command(self, command: str, cwd: str) -> str | None:
    """对潜在危险命令的最佳安全守卫"""
    cmd = command.strip()
    lower = cmd.lower()

    # 1. 黑名单检查
    for pattern in self.deny_patterns:
        if re.search(pattern, lower):
            return "Error: Command blocked by safety guard (dangerous pattern detected)"

    # 2. 白名单检查
    if self.allow_patterns:
        if not any(re.search(p, lower) for p in self.allow_patterns):
            return "Error: Command blocked by safety guard (not in allowlist)"

    # 3. 路径遍历检查
    if self.restrict_to_workspace:
        if "..\\" in cmd or "../" in cmd:
            return "Error: Command blocked by safety guard (path traversal detected)"

        cwd_path = Path(cwd).resolve()

        # 提取路径
        win_paths = re.findall(r"[A-Za-z]:\\[^\\\"']+", cmd)
        posix_paths = re.findall(r"(?:^|[\s|>])(/[^\s\"'>]+)", cmd)

        for raw in win_paths + posix_paths:
            try:
                p = Path(raw.strip()).resolve()
            except Exception:
                continue
            # 检查路径是否在工作区外
            if p.is_absolute() and cwd_path not in p.parents and p != cwd_path:
                return "Error: Command blocked by safety guard (path outside working dir)"

    return None
```

### 5. WebSearchTool - 网页搜索

**文件**: `agent/tools/web.py:46-99`

```python
class WebSearchTool(Tool):
    """使用 Brave Search API 搜索网页"""

    name = "web_search"
    description = "Search the web. Returns titles, URLs, and snippets."
    parameters = {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Search query"},
            "count": {"type": "integer", "description": "Results (1-10)", "minimum": 1, "maximum": 10}
        },
        "required": ["query"]
    }

    def __init__(self, api_key: str | None = None, max_results: int = 5):
        self._init_api_key = api_key
        self.max_results = max_results

    @property
    def api_key(self) -> str:
        """在调用时解析 API key，以便环境/配置更改被生效"""
        return self._init_api_key or os.environ.get("BRAVE_API_KEY", "")

    async def execute(self, query: str, count: int | None = None, **kwargs: Any) -> str:
        if not self.api_key:
            return (
                "Error: Brave Search API key not configured. "
                "Set it in ~/.nanobot/config.json under tools.web.search.apiKey "
                "(or export BRAVE_API_KEY), then restart the gateway."
            )

        try:
            n = min(max(count or self.max_results, 1), 10)
            async with httpx.AsyncClient() as client:
                r = await client.get(
                    "https://api.search.brave.com/res/v1/web/search",
                    params={"q": query, "count": n},
                    headers={"Accept": "application/json", "X-Subscription-Token": api_key},
                    timeout=10.0
                )
                r.raise_for_status()

            results = r.json().get("web", {}).get("results", [])
            if not results:
                return f"No results for: {query}"

            lines = [f"Results for: {query}\n"]
            for i, item in enumerate(results[:n], 1):
                lines.append(f"{i}. {item.get('title', '')}\n   {item.get('url', '')}")
                if desc := item.get("description"):
                    lines.append(f"   {desc}")
            return "\n".join(lines)
        except Exception as e:
            return f"Error: {e}"
```

### 6. WebFetchTool - 网页抓取

**文件**: `agent/tools/web.py:102-173`

```python
class WebFetchTool(Tool):
    """使用 Readability 从 URL 获取并提取内容"""

    name = "web_fetch"
    description = "Fetch URL and extract readable content (HTML → markdown/text)."
    parameters = {
        "type": "object",
        "properties": {
            "url": {"type": "string", "description": "URL to fetch"},
            "extractMode": {"type": "string", "enum": ["markdown", "text"], "default": "markdown"},
            "maxChars": {"type": "integer", "minimum": 100}
        },
        "required": ["url"]
    }

    def __init__(self, max_chars: int = 50000):
        self.max_chars = max_chars

    async def execute(self, url: str, extractMode: str = "markdown", maxChars: int | None = None, **kwargs: Any) -> str:
        from readability import Document

        max_chars = maxChars or self.max_chars

        # 验证 URL
        is_valid, error_msg = _validate_url(url)
        if not is_valid:
            return json.dumps({"error": f"URL validation failed: {error_msg}", "url": url}, ensure_ascii=False)

        try:
            async with httpx.AsyncClient(
                follow_redirects=True,
                max_redirects=MAX_REDIRECTS,
                timeout=30.0
            ) as client:
                r = await client.get(url, headers={"User-Agent": USER_AGENT})
                r.raise_for_status()

            ctype = r.headers.get("content-type", "")

            # JSON
            if "application/json" in ctype:
                text, extractor = json.dumps(r.json(), indent=2, ensure_ascii=False), "json"
            # HTML
            elif "text/html" in ctype or r.text[:256].lower().startswith(("<!doctype", "<html")):
                doc = Document(r.text)
                content = self._to_markdown(doc.summary()) if extractMode == "markdown" else _strip_tags(doc.summary())
                text = f"# {doc.title()}\n\n{content}" if doc.title() else content
                extractor = "readability"
            else:
                text, extractor = r.text, "raw"

            # 截断
            truncated = len(text) > max_chars
            if truncated:
                text = text[:max_chars]

            return json.dumps({
                "url": url,
                "finalUrl": str(r.url),
                "status": r.status_code,
                "extractor": extractor,
                "truncated": truncated,
                "length": len(text),
                "text": text
            }, ensure_ascii=False)
        except Exception as e:
            return json.dumps({"error": str(e), "url": url}, ensure_ascii=False)
```

---

## 🎯 工具注册流程

### AgentLoop 中的工具注册

**位置**: `agent/loop.py:104-119`

```python
def _register_default_tools(self) -> None:
    """注册默认工具集合"""
    allowed_dir = self.workspace if self.restrict_to_workspace else None

    # 1. 文件系统工具
    for cls in (ReadFileTool, WriteFileTool, EditFileTool, ListDirTool):
        self.tools.register(cls(
            workspace=self.workspace,
            allowed_dir=allowed_dir
        ))

    # 2. Shell 执行工具
    self.tools.register(ExecTool(
        working_dir=str(self.workspace),
        timeout=self.exec_config.timeout,
        restrict_to_workspace=self.restrict_to_workspace,
    ))

    # 3. Web 工具
    self.tools.register(WebSearchTool(api_key=self.brave_api_key))
    self.tools.register(WebFetchTool())

    # 4. 消息工具
    self.tools.register(MessageTool(send_callback=self.bus.publish_outbound))

    # 5. 子代理工具
    self.tools.register(SpawnTool(manager=self.subagents))

    # 6. 定时任务工具
    if self.cron_service:
        self.tools.register(CronTool(self.cron_service))
```

---

## 🔄 工具执行流程

### 完整流程

```
LLM 决定调用工具
    │
    ▼
AgentLoop._run_agent_loop()
    │
    ├─▶ response.tool_calls = [
    │       {"name": "read_file", "arguments": {"path": "test.txt"}}
    │   ]
    │
    ▼
ToolRegistry.execute(name, arguments)
    │
    ├─▶ 查找工具
    │
    ├─▶ 验证参数 (Tool.validate_params)
    │   │
    │   ├─▶ 类型检查
    │   ├─▶ 必需字段检查
    │   ├─▶ 数值范围检查
    │   ├─▶ 字符串长度检查
    │   └─▶ 枚举检查
    │
    ├─▶ 执行工具 (Tool.execute)
    │   │
    │   ├─▶ ReadFileTool → 读取文件
    │   ├─▶ WriteFileTool → 写入文件
    │   ├─▶ ExecTool → 执行命令
    │   └─▶ ...
    │
    ▼
返回结果
    │
    ▼
添加到消息历史
    │
    ▼
继续 LLM 循环
```

### 参数验证示例

**输入**:
```json
{
  "path": 123,  // 错误：应该是字符串
  "line": 5
}
```

**验证过程**:

```python
# 1. 检查 path
errors.append("parameter should be string")  # ❌ 类型错误

# 2. 检查 line
# 如果 line 不在 properties 中，忽略（允许额外参数）

# 返回错误
["parameter should be string"]
```

---

## 🎨 设计模式和原则

### 1. 策略模式 (Strategy Pattern)

```python
class Tool(ABC):
    @abstractmethod
    async def execute(self, **kwargs) -> str:
        pass

# 每个工具是不同的策略
class ReadFileTool(Tool): ...
class WriteFileTool(Tool): ...
class ExecTool(Tool): ...
```

### 2. 注册表模式 (Registry Pattern)

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool):
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool:
        return self._tools.get(name)
```

### 3. 模板方法模式 (Template Method)

```python
class Tool(ABC):
    async def execute(self, **kwargs) -> str:
        # 子类实现具体逻辑
        pass
```

### 4. 验证器模式 (Validator Pattern)

```python
def validate_params(self, params: dict) -> list[str]:
    """验证参数，返回错误列表"""
    return self._validate(params, self.parameters, "")
```

---

## 🔒 安全机制

### 1. 路径限制

```python
def _resolve_path(path: str, workspace: Path, allowed_dir: Path) -> Path:
    """解析路径并强制目录限制"""
    p = Path(path).expanduser()
    if not p.is_absolute() and workspace:
        p = workspace / p
    resolved = p.resolve()

    if allowed_dir:
        try:
            resolved.relative_to(allowed_dir.resolve())
        except ValueError:
            raise PermissionError(f"Path {path} is outside allowed directory {allowed_dir}")

    return resolved
```

### 2. 命令黑名单

```python
self.deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",      # rm -r, rm -rf
    r"\bdd\s+if=",               # dd
    r">\s*/dev/sd",              # write to disk
    r"\b(shutdown|reboot)\b",    # system power
]
```

### 3. 超时保护

```python
try:
    stdout, stderr = await asyncio.wait_for(
        process.communicate(),
        timeout=self.timeout
    )
except asyncio.TimeoutError:
    process.kill()
    return f"Error: Command timed out after {self.timeout} seconds"
```

### 4. 输出截断

```python
# 截断超长输出
max_len = 10000
if len(result) > max_len:
    result = result[:max_len] + f"\n... (truncated, {len(result) - max_len} more chars)"
```

---

## 🧪 测试建议

### 单元测试

1. **工具注册**:
   ```python
   def test_tool_registration():
       registry = ToolRegistry()
       tool = ReadFileTool()
       registry.register(tool)
       assert registry.has("read_file")
       assert registry.get("read_file") == tool
   ```

2. **参数验证**:
   ```python
   def test_parameter_validation():
       tool = ReadFileTool()
       errors = tool.validate_params({"path": 123})
       assert len(errors) > 0
       assert "should be string" in errors[0]
   ```

3. **工具执行**:
   ```python
   async def test_tool_execution():
       tool = WriteFileTool(workspace=Path("/tmp"))
       result = await tool.execute(path="test.txt", content="Hello")
       assert "Successfully wrote" in result
   ```

### 集成测试

1. **完整流程**:
   ```python
   async def test_full_workflow():
       registry = ToolRegistry()
       registry.register(ReadFileTool())

       # 执行
       result = await registry.execute("read_file", {"path": "test.txt"})

       # 验证
       assert result is not None
   ```

2. **错误处理**:
   ```python
   async def test_error_handling():
       registry = ToolRegistry()
       result = await registry.execute("nonexistent", {})
       assert "not found" in result
   ```

---

## 🔍 关键概念总结

### 1. Tool 抽象

所有工具都继承自 `Tool` 基类，实现统一的接口。

### 2. 参数验证

使用 JSON Schema 定义参数，自动验证类型、必需字段、范围等。

### 3. 注册表模式

工具通过名称注册和查找，支持动态管理。

### 4. 异步执行

所有工具都是异步的，使用 `async/await`。

### 5. 错误处理

统一的错误处理和提示，帮助 AI 理解和重试。

### 6. 安全机制

路径限制、命令黑名单、超时保护等。

---

## 📖 扩展阅读

### 相关文件

- `agent/tools/base.py` - 工具基类
- `agent/tools/registry.py` - 工具注册表
- `agent/tools/filesystem.py` - 文件系统工具
- `agent/tools/shell.py` - Shell 执行工具
- `agent/tools/web.py` - Web 工具
- `agent/tools/mcp.py` - MCP 集成

### 设计模式

- **Strategy**: 每个工具是一个策略
- **Registry**: 集中管理工具
- **Template Method**: 统一的执行流程
- **Validator**: 参数验证

---

## 🎓 学习要点

### 必须理解

1. **Tool 抽象**: name, description, parameters, execute
2. **参数验证**: JSON Schema，递归验证
3. **注册表模式**: 动态注册和查找
4. **异步执行**: asyncio.create_subprocess_shell
5. **安全机制**: 路径限制、命令黑名单、超时
6. **错误处理**: 统一格式，帮助 AI 重试

### 代码阅读路径

1. 从 `Tool` 基类开始（抽象定义）
2. 进入 `ToolRegistry`（管理逻辑）
3. 研究具体工具实现（ReadFileTool, ExecTool）
4. 理解参数验证机制
5. 学习安全守卫

---

## 🚀 下一步

现在你已经深入理解了 ToolRegistry！

想继续学习哪个模块？
- Channel Adapter - 渠道适配器
- Memory Store - 记忆系统
- Provider System - LLM 提供商
- MCP Integration - MCP 集成
