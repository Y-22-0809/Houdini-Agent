# Houdini Agent Plugin Development Guide
# Houdini Agent 插件开发指南

---

## Quick Start / 快速开始

1. Create a `.py` file in this `plugins/` directory (files starting with `_` are ignored)
2. Define a `PLUGIN_INFO` dict and a `register(ctx)` function
3. Restart Houdini Agent or click "Reload All" in Plugin Manager

---

## Plugin File Structure / 插件文件结构

```python
# plugins/my_plugin.py
# -*- coding: utf-8 -*-

# 1. Required: Plugin metadata / 必需：插件元数据
PLUGIN_INFO = {
    "name": "My Plugin",           # Display name / 显示名称
    "version": "1.0.0",            # Version / 版本号
    "author": "Your Name",         # Author / 作者
    "description": "What it does", # Description / 描述

    # Optional: Settings schema / 可选：设置项定义
    "settings": [
        {"key": "my_option", "type": "bool", "label": "Enable Feature", "default": True},
        {"key": "my_text",   "type": "string", "label": "Custom Text", "default": "hello"},
        {"key": "my_choice", "type": "string", "label": "Mode", "default": "fast",
         "options": ["fast", "balanced", "quality"]},
    ],
}


# 2. Required: Entry point / 必需：入口函数
def register(ctx):
    """Called when the plugin is loaded. ctx is a PluginContext instance."""
    ctx.log("My Plugin loaded!")
    # ... register hooks, tools, buttons here ...
```

---

## PluginContext API

The `ctx` object passed to `register()` provides the following methods:

### Event Hooks / 事件钩子

```python
ctx.on(event_name: str, callback: Callable, priority: int = 0)
```

Register a callback for a lifecycle event. Lower `priority` runs first (default 0).

| Event | Callback Signature | Description |
|-------|-------------------|-------------|
| `on_before_request` | `(messages: list) -> list or None` | Fired before each AI API call. Can modify the messages list. |
| `on_after_response` | `(result: dict)` | Fired after AI response is complete. |
| `on_before_tool` | `(tool_name: str, args: dict)` | Fired before a tool is executed. |
| `on_after_tool` | `(tool_name: str, args: dict, result: dict)` | Fired after a tool is executed. |
| `on_content_chunk` | `(chunk: str) -> str or None` | Fired for each AI output text chunk. Can modify the text. |
| `on_session_start` | `(session_id: str)` | Fired when a new chat session starts. |
| `on_session_end` | `(session_id: str)` | Fired when a chat session ends. |

**Example:**

```python
def my_hook(tool_name, args, result):
    ctx.log(f"Tool {tool_name} returned: {result.get('success')}")

ctx.on("on_after_tool", my_hook)
```

### Custom Tools / 自定义工具

```python
ctx.register_tool(
    name: str,             # Tool name (must be unique)
    description: str,      # Tool description (shown to AI)
    schema: dict,          # OpenAI function calling parameter schema
    handler: Callable,     # Function(args: dict) -> dict
)
```

Register a tool that the AI can invoke via function calling.

**Example:**

```python
ctx.register_tool(
    name="count_words",
    description="Count words in a text string",
    schema={
        "type": "object",
        "properties": {
            "text": {"type": "string", "description": "The text to count words in"}
        },
        "required": ["text"]
    },
    handler=lambda args: {
        "success": True,
        "result": f"Word count: {len(args.get('text', '').split())}"
    }
)
```

### UI Buttons / 工具栏按钮

```python
ctx.register_button(icon: str, tooltip: str, callback: Callable)
```

Add a button to the input area toolbar.

**Example:**

```python
ctx.register_button(
    icon="📊",
    tooltip="Show Stats",
    callback=lambda: ctx.log("Button clicked!")
)
```

### Chat Cards / 聊天卡片

```python
ctx.insert_chat_card(widget: QWidget)
```

Insert a custom Qt widget into the chat area. Thread-safe.

**Example:**

```python
from houdini_agent.qt_compat import QtWidgets

label = QtWidgets.QLabel("Hello from my plugin!")
label.setStyleSheet("color: #e2e8f0; padding: 8px;")
ctx.insert_chat_card(label)
```

### Settings / 设置

```python
ctx.get_setting(key: str, default=None) -> Any
ctx.set_setting(key: str, value: Any)
```

Read/write plugin settings. Settings are auto-persisted to `config/plugins.json`.
Define available settings in `PLUGIN_INFO["settings"]` to get an auto-generated settings UI.

### Logging / 日志

```python
ctx.log(msg: str)
```

Output a log message prefixed with `[Plugin:YourPluginName]`.

---

## Settings Schema / 设置项定义

Each setting in `PLUGIN_INFO["settings"]` is a dict:

| Key | Type | Description |
|-----|------|-------------|
| `key` | str | Setting identifier |
| `type` | str | `"bool"`, `"string"`, `"int"` |
| `label` | str | Display label in settings UI |
| `default` | any | Default value |
| `options` | list | (Optional) Dropdown choices for string type |
| `min` | int | (Optional) Minimum value for int type |
| `max` | int | (Optional) Maximum value for int type |

---

## Full Example / 完整示例

See `_example_plugin.py` in this directory for a complete working example.

---

## Notes / 注意事项

- Files starting with `_` (e.g., `_example_plugin.py`) are **not** auto-loaded
- Plugins run in the main Houdini process — avoid blocking operations
- Use `ctx.log()` instead of `print()` for consistent log formatting
- Tool handlers should return `{"success": True/False, "result": "..."}` format
- Use `houdini_agent.qt_compat` for Qt imports to ensure PySide2/PySide6 compatibility
- Plugin settings are stored in `config/plugins.json`
- You can manage plugins from the `⋯` menu → Plugin Manager
