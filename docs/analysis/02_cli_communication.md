# CLI 通信机制深度分析

## 1. 通信架构概览

PAL MCP Server 的 CLI 通信采用 **进程间通信 (IPC)** 模式,通过标准输入/输出 (stdio) 与外部 AI CLI 工具进行交互。

```
┌────────────────────────────────────────────────────────────────┐
│                    PAL MCP Server (Python)                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   CLinkTool                              │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              BaseCLIAgent                        │    │   │
│  │  │  ┌─────────────────────────────────────────┐    │    │   │
│  │  │  │     asyncio.create_subprocess_exec      │    │    │   │
│  │  │  └──────────────┬──────────────────────────┘    │    │   │
│  │  └────────────────│────────────────────────────────┘    │   │
│  └───────────────────│─────────────────────────────────────┘   │
└──────────────────────│─────────────────────────────────────────┘
                       │
                       │ stdin: prompt (UTF-8)
                       │ stdout: response (JSON/JSONL)
                       │ stderr: logs/errors
                       ▼
┌────────────────────────────────────────────────────────────────┐
│              External CLI (gemini/claude/codex)                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            AI Model API Interaction                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

## 2. 进程创建与管理

### 2.1 子进程创建

使用 Python asyncio 的异步子进程 API:

```python
# clink/agents/base.py:110-119
process = await asyncio.create_subprocess_exec(
    *command_with_output_flag,
    stdin=asyncio.subprocess.PIPE,    # 用于发送 prompt
    stdout=asyncio.subprocess.PIPE,   # 捕获响应
    stderr=asyncio.subprocess.PIPE,   # 捕获错误/日志
    cwd=cwd,                          # 工作目录
    limit=limit,                      # 流缓冲限制 (10MB)
    env=env,                          # 环境变量
)
```

### 2.2 流缓冲限制

```python
# clink/constants.py:9
DEFAULT_STREAM_LIMIT = 10 * 1024 * 1024  # 10MB per stream
```

这个限制确保大型输出不会导致内存溢出。

### 2.3 可执行文件解析

跨平台兼容性处理:

```python
# clink/agents/base.py:72-79
executable_name = command[0]
resolved_executable = shutil.which(executable_name)
if resolved_executable is None:
    raise CLIAgentError(
        f"Executable '{executable_name}' not found in PATH..."
    )
command[0] = resolved_executable  # 使用完整路径
```

## 3. 数据传输协议

### 3.1 输入协议 (stdin)

**传输方式**: UTF-8 编码的纯文本

```python
# clink/agents/base.py:123-127
stdout_bytes, stderr_bytes = await asyncio.wait_for(
    process.communicate(prompt.encode("utf-8")),  # 通过 stdin 发送
    timeout=self.client.timeout_seconds,
)
```

**Prompt 结构**:

```
[System Prompt (可选,取决于 CLI)]

You are operating through the Gemini CLI agent...

=== USER REQUEST ===
[用户实际请求内容]

=== FILE REFERENCES ===
- /path/to/file.py (last modified 2024-..., 1234 bytes)

Provide your response below using your own CLI tools as needed:
```

### 3.2 输出协议 (stdout)

每个 CLI 使用不同的输出格式:

| CLI | 格式 | 内置参数 | 解析器 |
|-----|------|----------|--------|
| Gemini | JSON | `-o json` | `gemini_json` |
| Claude | JSON | `--output-format json` | `claude_json` |
| Codex | JSONL | (内置) | `codex_jsonl` |

### 3.3 文件输出模式

某些 CLI 可能写入临时文件而非 stdout:

```python
# clink/agents/base.py:94-104
if self.client.output_to_file:
    fd, tmp_path = tempfile.mkstemp(prefix="clink-", suffix=".json")
    os.close(fd)
    output_file_path = Path(tmp_path)

    # 使用模板渲染输出标志
    flag_template = self.client.output_to_file.flag_template
    rendered_flag = flag_template.format(path=str(output_file_path))
    command_with_output_flag.extend(shlex.split(rendered_flag))
```

配置示例:
```json
{
  "output_to_file": {
    "flag_template": "--output {path}",
    "cleanup": true
  }
}
```

## 4. 超时与错误处理

### 4.1 超时管理

```python
# clink/constants.py:8
DEFAULT_TIMEOUT_SECONDS = 1800  # 30 分钟

# clink/agents/base.py:123-134
try:
    stdout_bytes, stderr_bytes = await asyncio.wait_for(
        process.communicate(prompt.encode("utf-8")),
        timeout=self.client.timeout_seconds,
    )
except asyncio.TimeoutError as exc:
    process.kill()                    # 强制终止
    await process.communicate()       # 清理僵尸进程
    raise CLIAgentError(
        f"CLI '{self.client.name}' timed out after {self.client.timeout_seconds} seconds",
    )
```

### 4.2 退出码处理

```python
# 非零退出码处理流程
if return_code != 0:
    # 1. 尝试错误恢复
    recovered = self._recover_from_error(...)
    if recovered is not None:
        return recovered

    # 2. 恢复失败,抛出错误
    raise CLIAgentError(
        f"CLI '{self.client.name}' exited with status {return_code}",
        returncode=return_code,
        stdout=stdout_text,
        stderr=stderr_text,
    )
```

### 4.3 错误恢复策略

**Gemini Agent 错误恢复**:

```python
# clink/agents/gemini.py:20-86
def _recover_from_error(self, *, returncode, stdout, stderr, ...):
    combined = "\n".join(part for part in (stderr, stdout) if part)

    # 尝试在输出中找到 JSON 错误块
    brace_index = combined.find("{")
    if brace_index == -1:
        return None

    json_candidate = combined[brace_index:]
    payload = json.loads(json_candidate)

    # 解析错误结构
    error_block = payload.get("error")
    code = error_block.get("code")
    err_type = error_block.get("type")
    detail_message = error_block.get("message")

    # 构建可读的错误消息
    message = f"Gemini CLI reported a tool failure ({code}).\n{detail_message}"

    return AgentOutput(
        parsed=ParsedCLIResponse(content=message, metadata={...}),
        ...
    )
```

**Claude/Codex Agent 错误恢复**:

```python
# clink/agents/claude.py:25-49
def _recover_from_error(self, ...):
    try:
        # 尝试用标准解析器解析输出
        parsed = self._parser.parse(stdout, stderr)
    except ParserError:
        return None  # 无法恢复

    # 即使退出码非零,如果能解析出有效内容,也视为成功
    return AgentOutput(parsed=parsed, ...)
```

## 5. 环境变量传递

### 5.1 环境构建

```python
# clink/agents/base.py:201-204
def _build_environment(self) -> dict[str, str]:
    env = os.environ.copy()      # 继承当前环境
    env.update(self.client.env)  # 合并配置的环境变量
    return env
```

### 5.2 配置层次

```python
# clink/registry.py:185-194
def _merge_env(self, raw, internal_defaults):
    merged: dict[str, str] = {}
    # 1. 内部默认值
    if internal_defaults and internal_defaults.env:
        merged.update(internal_defaults.env)
    # 2. 用户配置覆盖
    merged.update(raw.env)
    return merged
```

## 6. Claude CLI 特殊处理

Claude CLI 支持通过命令行参数注入系统提示:

```python
# clink/agents/claude.py:14-23
class ClaudeAgent(BaseCLIAgent):
    def _build_command(self, *, role, system_prompt):
        command = list(self.client.executable)
        command.extend(self.client.internal_args)
        command.extend(self.client.config_args)

        # 特殊: 通过 --append-system-prompt 注入
        if system_prompt and "--append-system-prompt" not in self.client.config_args:
            command.extend(["--append-system-prompt", system_prompt])

        command.extend(role.role_args)
        return command
```

对应的 CLinkTool 处理:

```python
# tools/clink.py:301-303
def _use_external_system_prompt(self, client: ResolvedCLIClient) -> bool:
    runner_name = (client.runner or client.name).lower()
    return runner_name == "claude"  # Claude 使用外部系统提示
```

## 7. 通信流程时序图

```
CLinkTool              BaseCLIAgent           CLI Process          Parser
    │                       │                      │                  │
    │  run(role, prompt)    │                      │                  │
    ├──────────────────────>│                      │                  │
    │                       │                      │                  │
    │                       │  build_command()     │                  │
    │                       ├─────────────┐        │                  │
    │                       │<────────────┘        │                  │
    │                       │                      │                  │
    │                       │  create_subprocess   │                  │
    │                       ├─────────────────────>│                  │
    │                       │                      │                  │
    │                       │  stdin: prompt       │                  │
    │                       ├─────────────────────>│                  │
    │                       │                      │                  │
    │                       │      [执行 AI 推理]    │                  │
    │                       │                      │                  │
    │                       │  stdout: JSON        │                  │
    │                       │<─────────────────────┤                  │
    │                       │                      │                  │
    │                       │  parse(stdout)       │                  │
    │                       ├──────────────────────┼─────────────────>│
    │                       │                      │                  │
    │                       │  ParsedCLIResponse   │                  │
    │                       │<─────────────────────┼──────────────────┤
    │                       │                      │                  │
    │   AgentOutput         │                      │                  │
    │<──────────────────────┤                      │                  │
    │                       │                      │                  │
```

## 8. 并发与资源管理

### 8.1 并发限制

当前实现没有显式的并发限制,每个 MCP 请求创建独立的子进程。

### 8.2 资源清理

```python
# 超时时的进程清理
process.kill()
await process.communicate()  # 等待进程终止,避免僵尸进程

# 临时文件清理
if self.client.output_to_file and self.client.output_to_file.cleanup:
    try:
        output_file_path.unlink()
    except OSError:
        pass  # 尽力清理
```

## 9. 安全考虑

### 9.1 命令注入防护

- 使用 `shlex.split()` 安全解析命令参数
- 使用 `subprocess_exec` 而非 `shell=True`
- 可执行文件路径通过 `shutil.which()` 验证

### 9.2 敏感信息处理

```python
# 命令行被记录为 sanitized_command
sanitized_command = list(command)  # 复制以避免暴露原始数据
```

### 9.3 工作目录隔离

```python
# 支持配置独立的工作目录
cwd = str(self.client.working_dir) if self.client.working_dir else None
```

## 10. Rust 迁移通信层考虑

| 功能 | Python 实现 | Rust 对应方案 |
|------|------------|---------------|
| 异步子进程 | `asyncio.create_subprocess_exec` | `tokio::process::Command` |
| 超时控制 | `asyncio.wait_for` | `tokio::time::timeout` |
| 管道通信 | `PIPE` + `communicate` | `Stdio::piped()` + `spawn` |
| 环境变量 | `os.environ.copy()` | `std::env::vars()` |
| 可执行文件查找 | `shutil.which` | `which` crate |
| 流缓冲 | `limit` 参数 | `BufReader` with capacity |

### Rust 示例代码结构

```rust
use tokio::process::Command;
use tokio::time::{timeout, Duration};

async fn run_cli(config: &CliConfig, prompt: &str) -> Result<AgentOutput> {
    let mut cmd = Command::new(&config.executable[0]);
    cmd.args(&config.executable[1..])
       .args(&config.internal_args)
       .args(&config.config_args)
       .stdin(Stdio::piped())
       .stdout(Stdio::piped())
       .stderr(Stdio::piped())
       .envs(&config.env);

    if let Some(cwd) = &config.working_dir {
        cmd.current_dir(cwd);
    }

    let mut child = cmd.spawn()?;

    // 写入 stdin
    if let Some(stdin) = child.stdin.take() {
        stdin.write_all(prompt.as_bytes()).await?;
    }

    // 带超时等待
    let output = timeout(
        Duration::from_secs(config.timeout_seconds),
        child.wait_with_output()
    ).await??;

    // 解析输出
    let parser = get_parser(&config.parser)?;
    parser.parse(&output.stdout, &output.stderr)
}
```
