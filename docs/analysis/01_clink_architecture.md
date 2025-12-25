# clink 模块架构深度分析

## 1. 概述

clink (CLI Link) 是 PAL MCP Server 的核心组件,用于桥接 MCP 协议请求到外部 AI CLI 工具(如 Gemini CLI, Claude CLI, Codex CLI)。它实现了一个可扩展的 CLI 集成框架,允许通过统一接口调用不同的 AI 命令行工具。

## 2. 模块结构

```
clink/
├── __init__.py           # 公共接口导出
├── constants.py          # 常量和内部默认值
├── models.py             # Pydantic 数据模型
├── registry.py           # CLI 客户端配置注册表
├── agents/               # CLI 执行代理
│   ├── __init__.py       # 代理工厂
│   ├── base.py           # 基础代理实现
│   ├── claude.py         # Claude CLI 特化
│   ├── codex.py          # Codex CLI 特化
│   └── gemini.py         # Gemini CLI 特化
└── parsers/              # 输出解析器
    ├── __init__.py       # 解析器注册表
    ├── base.py           # 基础解析器接口
    ├── claude.py         # Claude JSON 解析
    ├── codex.py          # Codex JSONL 解析
    └── gemini.py         # Gemini JSON 解析
```

## 3. 核心设计模式

### 3.1 配置注册表模式 (Registry Pattern)

`ClinkRegistry` 类实现了 CLI 客户端的集中配置管理:

```python
class ClinkRegistry:
    """加载 CLI 客户端定义并暴露给 schema 生成和运行时使用"""

    def __init__(self) -> None:
        self._clients: dict[str, ResolvedCLIClient] = {}
        self._load()
```

**配置加载顺序**(优先级从低到高):
1. 内置配置: `conf/cli_clients/`
2. 环境变量覆盖: `CLI_CLIENTS_CONFIG_PATH`
3. 用户配置: `~/.pal/cli_clients/`

### 3.2 工厂模式 (Factory Pattern)

代理创建使用工厂函数:

```python
_AGENTS: dict[str, type[BaseCLIAgent]] = {
    "gemini": GeminiAgent,
    "codex": CodexAgent,
    "claude": ClaudeAgent,
}

def create_agent(client: ResolvedCLIClient) -> BaseCLIAgent:
    agent_key = (client.runner or client.name).lower()
    agent_cls = _AGENTS.get(agent_key, BaseCLIAgent)
    return agent_cls(client)
```

### 3.3 模板方法模式 (Template Method)

`BaseCLIAgent` 定义执行骨架,子类可覆盖特定钩子:

```python
class BaseCLIAgent:
    async def run(self, ...) -> AgentOutput:
        command = self._build_command(...)      # 可覆盖
        env = self._build_environment()         # 可覆盖
        # ... 执行逻辑 ...
        recovered = self._recover_from_error()  # 可覆盖
        # ...
```

## 4. 数据模型层次

### 4.1 配置模型

```
CLIClientConfig (原始配置)
        ↓ 解析与验证
ResolvedCLIClient (运行时配置)
    ├── name: str
    ├── executable: list[str]
    ├── internal_args: list[str]
    ├── config_args: list[str]
    ├── env: dict[str, str]
    ├── timeout_seconds: int
    ├── parser: str
    ├── runner: str | None
    └── roles: dict[str, ResolvedCLIRole]
```

### 4.2 角色系统

每个 CLI 客户端可定义多个"角色",代表不同的系统提示和参数配置:

```json
{
  "roles": {
    "default": {
      "prompt_path": "systemprompts/clink/default.txt",
      "role_args": []
    },
    "codereviewer": {
      "prompt_path": "systemprompts/clink/default_codereviewer.txt",
      "role_args": []
    }
  }
}
```

### 4.3 执行结果模型

```python
@dataclass
class AgentOutput:
    parsed: ParsedCLIResponse      # 解析后的响应
    sanitized_command: list[str]   # 执行的命令(脱敏)
    returncode: int                # 退出码
    stdout: str                    # 标准输出
    stderr: str                    # 标准错误
    duration_seconds: float        # 执行时长
    parser_name: str               # 使用的解析器
    output_file_content: str | None # 文件输出内容
```

## 5. 内部默认值系统

每个支持的 CLI 都有预定义的内部默认值:

```python
INTERNAL_DEFAULTS: dict[str, CLIInternalDefaults] = {
    "gemini": CLIInternalDefaults(
        parser="gemini_json",
        additional_args=["-o", "json"],  # 输出 JSON 格式
        default_role_prompt="systemprompts/clink/default.txt",
        runner="gemini",
    ),
    "codex": CLIInternalDefaults(
        parser="codex_jsonl",
        additional_args=["exec"],
        default_role_prompt="systemprompts/clink/default.txt",
        runner="codex",
    ),
    "claude": CLIInternalDefaults(
        parser="claude_json",
        additional_args=["--print", "--output-format", "json"],
        default_role_prompt="systemprompts/clink/default.txt",
        runner="claude",
    ),
}
```

## 6. 命令构建流程

完整的命令行参数组装顺序:

```
executable (来自 config)
    ↓
internal_args (来自 INTERNAL_DEFAULTS)
    ↓
config_args (来自用户配置)
    ↓
role_args (来自角色配置)
    ↓
[output_flag] (如果配置了 output_to_file)
```

示例 - Gemini CLI 最终命令:
```bash
gemini -o json --yolo [prompt via stdin]
```

示例 - Claude CLI 最终命令:
```bash
claude --print --output-format json --permission-mode acceptEdits --model sonnet --append-system-prompt "..." [prompt via stdin]
```

## 7. 关键代码路径分析

### 7.1 从 MCP 请求到 CLI 执行

```
MCP call_tool("clink", {...})
        ↓
CLinkTool.execute()
        ↓
registry.get_client(cli_name)
        ↓
client_config.get_role(role_name)
        ↓
create_agent(client_config)
        ↓
agent.run(role, prompt, ...)
        ↓
asyncio.create_subprocess_exec(...)
        ↓
parser.parse(stdout, stderr)
        ↓
AgentOutput 返回
```

### 7.2 错误恢复机制

每个 CLI Agent 可以覆盖 `_recover_from_error` 方法来处理非零退出码:

```python
def _recover_from_error(
    self,
    *,
    returncode: int,
    stdout: str,
    stderr: str,
    ...
) -> AgentOutput | None:
    """
    返回 AgentOutput: 将失败转换为成功响应
    返回 None: 继续正常的错误处理流程
    """
```

**Gemini 错误恢复策略**: 解析 JSON 错误块,提取错误码和消息
**Claude/Codex 错误恢复策略**: 尝试使用标准解析器解析输出

## 8. 扩展点

### 8.1 添加新的 CLI 支持

1. 创建配置文件: `conf/cli_clients/newcli.json`
2. 添加内部默认值到 `constants.py`
3. 实现解析器: `parsers/newcli.py`
4. 可选: 实现特化代理: `agents/newcli.py`
5. 在工厂注册表中注册

### 8.2 扩展解析器

```python
class NewCLIParser(BaseParser):
    name = "newcli_format"

    def parse(self, stdout: str, stderr: str) -> ParsedCLIResponse:
        # 实现特定格式的解析逻辑
        return ParsedCLIResponse(content=..., metadata=...)
```

## 9. 设计亮点

1. **配置与代码分离**: JSON 配置驱动,支持用户覆盖
2. **统一抽象**: 不同 CLI 通过统一接口调用
3. **可扩展架构**: 解析器和代理使用注册表模式
4. **错误恢复**: 每个 CLI 可以定义自己的错误处理策略
5. **跨平台支持**: 使用 `shutil.which` 解析可执行文件路径

## 10. Rust 迁移考虑

| 组件 | Python 实现 | Rust 对应 |
|------|------------|-----------|
| 配置解析 | Pydantic | serde + serde_json |
| 异步执行 | asyncio | tokio |
| 子进程管理 | subprocess | tokio::process |
| 注册表模式 | dict + class | HashMap + trait |
| 工厂模式 | 函数 + dict | enum + match |
| 路径解析 | pathlib | std::path |
