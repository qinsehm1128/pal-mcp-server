# 输出解析系统深度分析

## 1. 解析器架构

PAL MCP Server 的输出解析系统采用策略模式 (Strategy Pattern),每种 CLI 输出格式对应一个专门的解析器。

```
┌─────────────────────────────────────────────────────────────────┐
│                      Parser Registry                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  _PARSER_CLASSES: dict[str, type[BaseParser]]           │    │
│  │     "gemini_json"  → GeminiJSONParser                   │    │
│  │     "codex_jsonl"  → CodexJSONLParser                   │    │
│  │     "claude_json"  → ClaudeJSONParser                   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ get_parser(name)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BaseParser (Abstract)                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  name: str = "base"                                      │    │
│  │  def parse(stdout, stderr) -> ParsedCLIResponse          │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
          ▲                   ▲                   ▲
          │                   │                   │
    ┌─────┴─────┐      ┌─────┴─────┐      ┌──────┴──────┐
    │ GeminiJSON│      │ CodexJSONL│      │ ClaudeJSON  │
    │  Parser   │      │  Parser   │      │  Parser     │
    └───────────┘      └───────────┘      └─────────────┘
```

## 2. 核心数据模型

### 2.1 解析结果

```python
# clink/parsers/base.py:9-14
@dataclass
class ParsedCLIResponse:
    content: str              # 主要响应内容
    metadata: dict[str, Any]  # 附加元数据(统计信息、错误等)
```

### 2.2 解析错误

```python
# clink/parsers/base.py:17-18
class ParserError(RuntimeError):
    """当 CLI 输出无法解析为结构化响应时抛出"""
```

## 3. 解析器注册表

### 3.1 注册表实现

```python
# clink/parsers/__init__.py:10-22
_PARSER_CLASSES: dict[str, type[BaseParser]] = {
    CodexJSONLParser.name: CodexJSONLParser,  # "codex_jsonl"
    GeminiJSONParser.name: GeminiJSONParser,  # "gemini_json"
    ClaudeJSONParser.name: ClaudeJSONParser,  # "claude_json"
}

def get_parser(name: str) -> BaseParser:
    normalized = (name or "").lower()
    if normalized not in _PARSER_CLASSES:
        raise ParserError(f"No parser registered for '{name}'")
    parser_cls = _PARSER_CLASSES[normalized]
    return parser_cls()  # 每次创建新实例
```

### 3.2 解析器使用

```python
# clink/agents/base.py:52
self._parser: BaseParser = get_parser(client.parser)

# clink/agents/base.py:172-180
try:
    parsed = self._parser.parse(stdout_text, stderr_text)
except ParserError as exc:
    raise CLIAgentError(
        f"Failed to parse output from CLI '{self.client.name}': {exc}",
        returncode=return_code,
        stdout=stdout_text,
        stderr=stderr_text,
    )
```

## 4. Gemini JSON 解析器

### 4.1 输入格式

```json
{
  "response": "这是 Gemini 的响应文本...",
  "stats": {
    "models": {
      "gemini-2.0-flash": {
        "tokens": {
          "input": 1234,
          "output": 567
        },
        "api": {
          "totalLatencyMs": 2500,
          "totalRequests": 1,
          "totalErrors": 0
        }
      }
    }
  }
}
```

### 4.2 解析逻辑

```python
# clink/parsers/gemini.py:16-57
def parse(self, stdout: str, stderr: str) -> ParsedCLIResponse:
    # 1. 空输出检查
    if not stdout.strip():
        raise ParserError("Gemini CLI returned empty stdout...")

    # 2. JSON 解析
    payload = json.loads(stdout)

    # 3. 提取响应文本
    response = payload.get("response")
    response_text = response.strip() if isinstance(response, str) else ""

    # 4. 提取元数据
    metadata = {"raw": payload}
    stats = payload.get("stats")
    if isinstance(stats, dict):
        # 提取模型信息
        models = stats.get("models")
        model_name = next(iter(models.keys()))
        metadata["model_used"] = model_name

        # 提取 token 使用量
        model_stats = models.get(model_name) or {}
        tokens = model_stats.get("tokens")
        if isinstance(tokens, dict):
            metadata["token_usage"] = tokens

        # 提取延迟信息
        api_stats = model_stats.get("api")
        if isinstance(api_stats, dict):
            metadata["latency_ms"] = api_stats.get("totalLatencyMs")

    # 5. 返回结果或尝试回退
    if response_text:
        return ParsedCLIResponse(content=response_text, metadata=metadata)

    # 回退处理空响应
    fallback_message, extra = self._build_fallback_message(payload, stderr)
    ...
```

### 4.3 回退消息构建

处理 API 错误(如 429 速率限制):

```python
# clink/parsers/gemini.py:59-98
def _build_fallback_message(self, payload, stderr):
    stderr_lower = stderr.strip().lower()

    # 检测速率限制
    if "429" in stderr_lower or "rate limit" in stderr_lower:
        return (
            "Gemini request returned no content because the API reported a 429 rate limit.",
            {"rate_limit_status": 429}
        )

    # 检测 API 错误
    stats = payload.get("stats")
    if stats:
        # ... 检查 totalErrors ...
        if total_errors > 0:
            return (
                f"Gemini CLI returned no textual output. The API reported {total_errors} error(s)",
                {"api_total_errors": total_errors}
            )

    # 最后的回退
    if stderr.strip():
        return ("...Raw stderr was preserved for troubleshooting.", {})

    return None, {}
```

## 5. Codex JSONL 解析器

### 5.1 输入格式

Codex CLI 使用 JSON Lines 格式,每行一个 JSON 事件:

```jsonl
{"type": "item.completed", "item": {"type": "agent_message", "text": "消息1"}}
{"type": "item.completed", "item": {"type": "agent_message", "text": "消息2"}}
{"type": "error", "message": "错误信息"}
{"type": "turn.completed", "usage": {"input_tokens": 100, "output_tokens": 50}}
```

### 5.2 解析逻辑

```python
# clink/parsers/codex.py:16-63
def parse(self, stdout: str, stderr: str) -> ParsedCLIResponse:
    lines = [line.strip() for line in (stdout or "").splitlines() if line.strip()]

    events: list[dict] = []
    agent_messages: list[str] = []
    errors: list[str] = []
    usage: dict | None = None

    for line in lines:
        # 跳过非 JSON 行
        if not line.startswith("{"):
            continue

        try:
            event = json.loads(line)
        except json.JSONDecodeError:
            continue

        events.append(event)
        event_type = event.get("type")

        # 收集 agent_message 类型的消息
        if event_type == "item.completed":
            item = event.get("item") or {}
            if item.get("type") == "agent_message":
                text = item.get("text")
                if isinstance(text, str) and text.strip():
                    agent_messages.append(text.strip())

        # 收集错误
        elif event_type == "error":
            message = event.get("message")
            if isinstance(message, str) and message.strip():
                errors.append(message.strip())

        # 收集使用量统计
        elif event_type == "turn.completed":
            turn_usage = event.get("usage")
            if isinstance(turn_usage, dict):
                usage = turn_usage

    # 如果没有消息但有错误,使用错误作为消息
    if not agent_messages and errors:
        agent_messages.extend(errors)

    if not agent_messages:
        raise ParserError("Codex CLI JSONL output did not include an agent_message item")

    # 合并所有消息
    content = "\n\n".join(agent_messages).strip()

    metadata = {"events": events}
    if errors:
        metadata["errors"] = errors
    if usage:
        metadata["usage"] = usage

    return ParsedCLIResponse(content=content, metadata=metadata)
```

## 6. Claude JSON 解析器

### 6.1 输入格式

Claude CLI 可能返回单个对象或事件数组:

**单对象格式**:
```json
{
  "type": "result",
  "result": "响应内容...",
  "is_error": false,
  "duration_ms": 1500,
  "usage": {"input_tokens": 100, "output_tokens": 50}
}
```

**数组格式**:
```json
[
  {"type": "assistant", "message": "中间消息..."},
  {"type": "result", "result": "最终响应...", "usage": {...}}
]
```

### 6.2 解析逻辑

```python
# clink/parsers/claude.py:16-77
def parse(self, stdout: str, stderr: str) -> ParsedCLIResponse:
    if not stdout.strip():
        raise ParserError("Claude CLI returned empty stdout...")

    loaded = json.loads(stdout)
    events: list[dict] | None = None
    assistant_entry: dict | None = None

    # 处理不同的 JSON 结构
    if isinstance(loaded, dict):
        payload = loaded
    elif isinstance(loaded, list):
        events = [item for item in loaded if isinstance(item, dict)]

        # 查找 result 类型的条目
        result_entry = next(
            (item for item in events if item.get("type") == "result" or "result" in item),
            None,
        )

        # 查找最后一个 assistant 类型的条目(从后向前搜索)
        assistant_entry = next(
            (item for item in reversed(events) if item.get("type") == "assistant"),
            None,
        )

        payload = result_entry or assistant_entry or (events[-1] if events else {})
    else:
        raise ParserError("Claude CLI returned unexpected JSON payload")

    # 构建元数据
    metadata = self._build_metadata(payload, stderr)

    # 提取响应内容
    result = payload.get("result")
    if isinstance(result, str):
        content = result.strip()
    elif isinstance(result, list):
        # 某些流程返回字符串列表
        joined = [part.strip() for part in result if isinstance(part, str)]
        content = "\n".join(joined)

    if content:
        return ParsedCLIResponse(content=content, metadata=metadata)

    # 尝试从 message 字段提取
    message = self._extract_message(payload)
    if message is None and assistant_entry:
        message = self._extract_message(assistant_entry)

    if message:
        return ParsedCLIResponse(content=message, metadata=metadata)

    # 最后的回退
    if stderr.strip():
        return ParsedCLIResponse(
            content="Claude CLI returned no textual result...",
            metadata=metadata,
        )

    raise ParserError("Claude CLI response did not contain a textual result")
```

### 6.3 元数据提取

```python
# clink/parsers/claude.py:79-124
def _build_metadata(self, payload: dict, stderr: str) -> dict:
    metadata = {
        "raw": payload,
        "is_error": bool(payload.get("is_error")),
    }

    # 类型信息
    if type_field := payload.get("type"):
        metadata["type"] = type_field
    if subtype_field := payload.get("subtype"):
        metadata["subtype"] = subtype_field

    # 时间信息
    if duration_ms := payload.get("duration_ms"):
        metadata["duration_ms"] = duration_ms
    if api_duration := payload.get("duration_api_ms"):
        metadata["duration_api_ms"] = api_duration

    # 使用量信息
    if usage := payload.get("usage"):
        metadata["usage"] = usage

    # 模型使用信息
    if model_usage := payload.get("modelUsage"):
        metadata["model_usage"] = model_usage
        metadata["model_used"] = next(iter(model_usage.keys()))

    # 权限拒绝信息
    if permission_denials := payload.get("permission_denials"):
        metadata["permission_denials"] = permission_denials

    # 会话标识
    if session_id := payload.get("session_id"):
        metadata["session_id"] = session_id
    if uuid_field := payload.get("uuid"):
        metadata["uuid"] = uuid_field

    return metadata
```

## 7. 解析器选择流程

```
CLI 配置 (JSON)
    │
    │ "parser": "gemini_json"
    ▼
INTERNAL_DEFAULTS
    │
    │ parser = "gemini_json"
    ▼
ResolvedCLIClient
    │
    │ parser: str
    ▼
BaseCLIAgent.__init__
    │
    │ self._parser = get_parser(client.parser)
    ▼
_PARSER_CLASSES["gemini_json"]
    │
    ▼
GeminiJSONParser()
```

## 8. 元数据标准化

各解析器提取的标准元数据字段:

| 字段 | 描述 | Gemini | Codex | Claude |
|------|------|--------|-------|--------|
| `raw` | 原始 JSON 响应 | ✓ | ✓ (events) | ✓ |
| `model_used` | 使用的模型名称 | ✓ | ✗ | ✓ |
| `token_usage`/`usage` | Token 使用量 | ✓ | ✓ | ✓ |
| `latency_ms`/`duration_ms` | 响应延迟 | ✓ | ✗ | ✓ |
| `errors` | 错误列表 | ✗ | ✓ | ✗ |
| `stderr` | 标准错误输出 | ✓ | ✓ | ✓ |
| `is_error` | 是否为错误响应 | ✗ | ✗ | ✓ |

## 9. 错误处理策略

### 9.1 解析失败处理

```
stdout 为空?
    │
    ├─ Yes → ParserError("empty stdout...")
    │
    ▼ No
JSON 解析失败?
    │
    ├─ Yes → ParserError("Failed to decode...")
    │
    ▼ No
找不到内容字段?
    │
    ├─ 尝试回退策略
    │    │
    │    ├─ stderr 可用 → 返回带 stderr 的响应
    │    │
    │    └─ 无可用数据 → ParserError("missing field...")
    │
    ▼
返回 ParsedCLIResponse
```

### 9.2 优雅降级

各解析器都实现了优雅降级:

1. **首选**: 主响应字段 (`response`, `result`, `agent_message`)
2. **回退 1**: 错误消息字段
3. **回退 2**: 带有诊断信息的 stderr
4. **最终**: 抛出 ParserError

## 10. CLinkTool 输出处理

### 10.1 输出长度限制

```python
# tools/clink.py:25
MAX_RESPONSE_CHARS = 20_000

# tools/clink.py:333-397
def _apply_output_limit(self, client, content, metadata):
    if len(content) <= MAX_RESPONSE_CHARS:
        return content, metadata

    # 尝试提取 <SUMMARY> 标签
    summary = self._extract_summary(content)
    if summary:
        return summary, {**metadata, "output_summarized": True}

    # 截断处理
    excerpt_limit = min(4000, MAX_RESPONSE_CHARS // 2)
    excerpt = content[:excerpt_limit]
    return f"...(truncated)...\n{excerpt}", {**metadata, "output_truncated": True}
```

### 10.2 Summary 提取

```python
# tools/clink.py:26
SUMMARY_PATTERN = re.compile(r"<SUMMARY>(.*?)</SUMMARY>", re.IGNORECASE | re.DOTALL)

# tools/clink.py:399-404
def _extract_summary(self, content: str) -> str | None:
    match = SUMMARY_PATTERN.search(content)
    if not match:
        return None
    return match.group(1).strip() or None
```

这与系统提示中的要求对应:
```
Always conclude with `<SUMMARY>...</SUMMARY>` containing a terse (≤500 words) recap...
```

## 11. Rust 迁移解析器考虑

| 功能 | Python | Rust |
|------|--------|------|
| JSON 解析 | `json.loads()` | `serde_json::from_str()` |
| JSONL 解析 | 逐行 + `json.loads()` | `serde_json::Deserializer::from_str().into_iter()` |
| 正则匹配 | `re.compile()` | `regex` crate |
| 可选字段处理 | `dict.get()` | `Option<T>` + `serde` |
| 类型验证 | `isinstance()` | 编译时类型检查 |
| 错误处理 | 异常 | `Result<T, E>` |

### Rust 解析器 trait 示例

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone)]
pub struct ParsedCLIResponse {
    pub content: String,
    pub metadata: serde_json::Value,
}

#[derive(Debug, thiserror::Error)]
pub enum ParserError {
    #[error("Empty stdout")]
    EmptyStdout,

    #[error("JSON decode error: {0}")]
    JsonDecode(#[from] serde_json::Error),

    #[error("Missing required field: {0}")]
    MissingField(String),
}

pub trait CliParser: Send + Sync {
    fn name(&self) -> &'static str;
    fn parse(&self, stdout: &str, stderr: &str) -> Result<ParsedCLIResponse, ParserError>;
}

// Gemini 解析器实现
#[derive(Debug, Deserialize)]
struct GeminiPayload {
    response: Option<String>,
    stats: Option<GeminiStats>,
}

pub struct GeminiJsonParser;

impl CliParser for GeminiJsonParser {
    fn name(&self) -> &'static str {
        "gemini_json"
    }

    fn parse(&self, stdout: &str, stderr: &str) -> Result<ParsedCLIResponse, ParserError> {
        if stdout.trim().is_empty() {
            return Err(ParserError::EmptyStdout);
        }

        let payload: GeminiPayload = serde_json::from_str(stdout)?;

        let content = payload.response
            .as_ref()
            .map(|s| s.trim().to_string())
            .filter(|s| !s.is_empty())
            .ok_or_else(|| ParserError::MissingField("response".into()))?;

        let metadata = serde_json::to_value(&payload)?;

        Ok(ParsedCLIResponse { content, metadata })
    }
}
```
