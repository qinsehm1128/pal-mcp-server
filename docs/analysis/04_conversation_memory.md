# 对话缓存系统深度分析

## 1. 架构概述

PAL MCP Server 的对话缓存系统解决了 MCP 协议无状态性的核心问题,实现了跨工具、跨请求的对话上下文持久化。

```
┌────────────────────────────────────────────────────────────────────┐
│                    MCP Protocol (Stateless)                         │
│  每个请求独立,无内置会话支持                                          │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                   Conversation Memory Layer                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     ThreadContext                             │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  thread_id: UUID                                        │ │  │
│  │  │  parent_thread_id: UUID | None (链式支持)                │ │  │
│  │  │  tool_name: str (创建工具)                               │ │  │
│  │  │  turns: list[ConversationTurn]                          │ │  │
│  │  │  initial_context: dict (原始请求)                        │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              InMemoryStorage (Singleton)                      │  │
│  │  _store: dict[str, tuple[str, float]]  # (value, expires_at) │  │
│  │  _lock: threading.Lock                  # 线程安全            │  │
│  │  _cleanup_thread: Thread                # 后台清理            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## 2. 核心数据模型

### 2.1 ConversationTurn

```python
class ConversationTurn(BaseModel):
    """对话中的单个轮次"""

    role: str                           # "user" (Agent) 或 "assistant" (Model)
    content: str                        # 消息内容
    timestamp: str                      # ISO 时间戳
    files: Optional[list[str]] = None   # 该轮次引用的文件
    images: Optional[list[str]] = None  # 该轮次引用的图片
    tool_name: Optional[str] = None     # 生成此轮次的工具
    model_provider: Optional[str] = None # 模型提供商 (google, openai, etc)
    model_name: Optional[str] = None    # 具体模型名称
    model_metadata: Optional[dict] = None # 附加模型信息
```

### 2.2 ThreadContext

```python
class ThreadContext(BaseModel):
    """完整的对话上下文"""

    thread_id: str                      # UUID 标识符
    parent_thread_id: Optional[str]     # 父线程 (用于链式对话)
    created_at: str                     # 创建时间
    last_updated_at: str                # 最后更新时间
    tool_name: str                      # 创建此线程的工具
    turns: list[ConversationTurn]       # 所有对话轮次
    initial_context: dict[str, Any]     # 原始请求参数
```

## 3. 存储后端

### 3.1 InMemoryStorage 实现

```python
class InMemoryStorage:
    """线程安全的内存存储"""

    def __init__(self):
        self._store: dict[str, tuple[str, float]] = {}  # key -> (value, expires_at)
        self._lock = threading.Lock()

        # 后台清理线程
        timeout_hours = int(get_env("CONVERSATION_TIMEOUT_HOURS", "3"))
        self._cleanup_interval = max(300, (timeout_hours * 3600) // 10)
        self._cleanup_thread = threading.Thread(target=self._cleanup_worker, daemon=True)
        self._cleanup_thread.start()

    def setex(self, key: str, ttl_seconds: int, value: str) -> None:
        """带过期时间的存储 (Redis 兼容接口)"""
        with self._lock:
            expires_at = time.time() + ttl_seconds
            self._store[key] = (value, expires_at)

    def get(self, key: str) -> Optional[str]:
        """获取值 (自动过滤过期条目)"""
        with self._lock:
            if key in self._store:
                value, expires_at = self._store[key]
                if time.time() < expires_at:
                    return value
                else:
                    del self._store[key]  # 惰性删除
        return None
```

### 3.2 单例模式

```python
_storage_instance = None
_storage_lock = threading.Lock()

def get_storage_backend() -> InMemoryStorage:
    """全局单例获取"""
    global _storage_instance
    if _storage_instance is None:
        with _storage_lock:
            if _storage_instance is None:
                _storage_instance = InMemoryStorage()
    return _storage_instance
```

### 3.3 关键限制

**进程隔离问题**:
```
⚠️ 内存存储仅在单个 Python 进程内有效
   - ✓ Claude Desktop 持久 MCP 服务器进程
   - ✗ 子进程调用 (每个子进程独立内存空间)
```

## 4. 核心 API

### 4.1 线程创建

```python
def create_thread(
    tool_name: str,
    initial_request: dict[str, Any],
    parent_thread_id: Optional[str] = None
) -> str:
    """创建新对话线程"""

    thread_id = str(uuid.uuid4())
    now = datetime.now(timezone.utc).isoformat()

    # 过滤不可序列化的参数
    filtered_context = {
        k: v for k, v in initial_request.items()
        if k not in ["temperature", "thinking_mode", "model", "continuation_id"]
    }

    context = ThreadContext(
        thread_id=thread_id,
        parent_thread_id=parent_thread_id,
        created_at=now,
        last_updated_at=now,
        tool_name=tool_name,
        turns=[],
        initial_context=filtered_context,
    )

    storage = get_storage()
    key = f"thread:{thread_id}"
    storage.setex(key, CONVERSATION_TIMEOUT_SECONDS, context.model_dump_json())

    return thread_id
```

### 4.2 轮次添加

```python
def add_turn(
    thread_id: str,
    role: str,
    content: str,
    files: Optional[list[str]] = None,
    images: Optional[list[str]] = None,
    tool_name: Optional[str] = None,
    model_provider: Optional[str] = None,
    model_name: Optional[str] = None,
    model_metadata: Optional[dict] = None,
) -> bool:
    """向线程添加新轮次"""

    context = get_thread(thread_id)
    if not context:
        return False

    # 检查轮次限制
    if len(context.turns) >= MAX_CONVERSATION_TURNS:
        return False

    turn = ConversationTurn(
        role=role,
        content=content,
        timestamp=datetime.now(timezone.utc).isoformat(),
        files=files,
        images=images,
        tool_name=tool_name,
        model_provider=model_provider,
        model_name=model_name,
        model_metadata=model_metadata,
    )

    context.turns.append(turn)
    context.last_updated_at = datetime.now(timezone.utc).isoformat()

    # 保存并刷新 TTL
    storage = get_storage()
    storage.setex(
        f"thread:{thread_id}",
        CONVERSATION_TIMEOUT_SECONDS,
        context.model_dump_json()
    )
    return True
```

### 4.3 线程链遍历

```python
def get_thread_chain(thread_id: str, max_depth: int = 20) -> list[ThreadContext]:
    """获取从当前线程到根线程的完整链"""

    chain = []
    current_id = thread_id
    seen_ids = set()

    while current_id and len(chain) < max_depth:
        # 防止循环引用
        if current_id in seen_ids:
            break
        seen_ids.add(current_id)

        context = get_thread(current_id)
        if not context:
            break

        chain.append(context)
        current_id = context.parent_thread_id

    # 反转为时间顺序 (最老在前)
    chain.reverse()
    return chain
```

## 5. 文件优先级算法

### 5.1 Newest-First 文件收集

核心思想: **相同文件在多个轮次出现时,保留最新轮次的引用**

```python
def get_conversation_file_list(context: ThreadContext) -> list[str]:
    """提取唯一文件列表,最新优先"""

    if not context.turns:
        return []

    seen_files = set()
    file_list = []

    # 关键: 从后向前遍历 (newest first)
    for i in range(len(context.turns) - 1, -1, -1):
        turn = context.turns[i]
        if turn.files:
            for file_path in turn.files:
                if file_path not in seen_files:
                    # 首次遇到 = 最新引用
                    seen_files.add(file_path)
                    file_list.append(file_path)
                # 已见过 = 更新的轮次已包含,跳过

    return file_list
```

### 5.2 算法示例

```
Turn 1: files = ["main.py", "utils.py"]
Turn 2: files = ["test.py"]
Turn 3: files = ["main.py", "config.py"]

遍历顺序: Turn 3 → Turn 2 → Turn 1

处理 Turn 3:
  - main.py: 未见过 → 添加
  - config.py: 未见过 → 添加
  file_list = ["main.py", "config.py"]

处理 Turn 2:
  - test.py: 未见过 → 添加
  file_list = ["main.py", "config.py", "test.py"]

处理 Turn 1:
  - main.py: 已见过 (Turn 3) → 跳过
  - utils.py: 未见过 → 添加
  file_list = ["main.py", "config.py", "test.py", "utils.py"]

最终结果: main.py 来自 Turn 3 (最新), 而非 Turn 1
```

## 6. 对话历史构建

### 6.1 双阶段策略

**阶段 1: 收集 (Newest-First for Token Budget)**
```
优先保留最近的轮次,当 token 预算不足时排除较老的轮次
```

**阶段 2: 呈现 (Chronological for LLM Understanding)**
```
将收集的轮次反转为时间顺序,让 LLM 看到自然的对话流程
```

```python
def build_conversation_history(context: ThreadContext, model_context=None) -> tuple[str, int]:
    # 获取 token 分配
    token_allocation = model_context.calculate_token_allocation()
    max_history_tokens = token_allocation.history_tokens

    # === 阶段 1: 收集 (Newest-First) ===
    turn_entries = []
    total_turn_tokens = 0

    # 从最新到最老遍历
    for idx in range(len(all_turns) - 1, -1, -1):
        turn = all_turns[idx]
        turn_content = format_turn(turn, idx)
        turn_tokens = estimate_tokens(turn_content)

        # 检查 token 预算
        if total_turn_tokens + turn_tokens > max_history_tokens:
            break  # 停止添加,较老的轮次被排除

        turn_entries.append((idx, turn_content))
        total_turn_tokens += turn_tokens

    # === 阶段 2: 呈现 (Chronological) ===
    turn_entries.reverse()  # 恢复时间顺序

    # 构建最终输出
    for _, turn_content in turn_entries:
        history_parts.append(turn_content)
```

### 6.2 输出格式

```
=== CONVERSATION HISTORY (CONTINUATION) ===
Thread: <uuid>
Tool: analyze
Turn 5/50
You are continuing this conversation thread from where it left off.

=== FILES REFERENCED IN THIS CONVERSATION ===
The following files have been shared and analyzed during our conversation.
[NOTE: 3 files omitted (size constraints, missing files, or access issues)]

[嵌入的文件内容,带行号]

=== END REFERENCED FILES ===

Previous conversation turns:

--- Turn 1 (Agent using analyze) ---
Files used in this turn: main.py

[用户请求内容]

--- Turn 2 (gemini-2.5-flash using analyze via google) ---
[模型响应内容]

--- Turn 3 (Agent using codereview) ---
[跨工具继续的请求]

=== END CONVERSATION HISTORY ===

IMPORTANT: You are continuing an existing conversation thread...
This is turn 6 of the conversation...
```

## 7. Token 预算管理

### 7.1 Token 分配

```python
# utils/model_context.py
class TokenAllocation:
    total_tokens: int      # 模型上下文窗口
    content_tokens: int    # 60-80% 用于内容
    response_tokens: int   # 20-40% 保留给响应
    file_tokens: int       # 内容的 30-40% 用于文件
    history_tokens: int    # 内容的 40-50% 用于对话历史
```

### 7.2 文件包含规划

```python
def _plan_file_inclusion_by_size(
    all_files: list[str],
    max_file_tokens: int
) -> tuple[list[str], list[str], int]:
    """基于大小约束规划文件包含"""

    files_to_include = []
    files_to_skip = []
    total_tokens = 0

    for file_path in all_files:  # all_files 已按 newest-first 排序
        if os.path.exists(file_path) and os.path.isfile(file_path):
            estimated_tokens = estimate_file_tokens(file_path)

            if total_tokens + estimated_tokens <= max_file_tokens:
                files_to_include.append(file_path)
                total_tokens += estimated_tokens
            else:
                files_to_skip.append(file_path)  # 较老的文件被跳过
        else:
            files_to_skip.append(file_path)

    return files_to_include, files_to_skip, total_tokens
```

## 8. 跨工具继续

### 8.1 工作流程

```
Tool A (analyze) 创建线程
        │
        │ create_thread("analyze", request)
        │ → 返回 UUID
        ▼
Tool A 添加响应
        │
        │ add_turn(UUID, "assistant", response, tool_name="analyze")
        ▼
用户使用 continuation_id 调用 Tool B (codereview)
        │
        │ get_thread(UUID)
        │ → 获取完整上下文
        ▼
Tool B 看到完整历史
        │
        │ build_conversation_history(context)
        │ → 包含 Tool A 的所有轮次和文件
        ▼
Tool B 添加响应
        │
        │ add_turn(UUID, "assistant", response, tool_name="codereview")
        ▼
对话继续...
```

### 8.2 上下文重建 (server.py)

```python
async def reconstruct_thread_context(
    continuation_id: str,
    arguments: dict
) -> dict:
    """从存储重建对话上下文"""

    context = get_thread(continuation_id)
    if not context:
        return arguments  # 无效 ID,返回原始参数

    # 构建对话历史
    history, token_count = build_conversation_history(
        context,
        model_context
    )

    # 注入到提示中
    if history:
        arguments["_conversation_history"] = history
        arguments["_history_token_count"] = token_count

    # 收集跨轮次的文件
    conversation_files = get_conversation_file_list(context)
    arguments["_conversation_files"] = conversation_files

    return arguments
```

## 9. 配置参数

| 参数 | 环境变量 | 默认值 | 说明 |
|------|----------|--------|------|
| 最大轮次 | `MAX_CONVERSATION_TURNS` | 50 | 单线程最大对话轮次 |
| 超时时间 | `CONVERSATION_TIMEOUT_HOURS` | 3 | 线程 TTL (小时) |
| 清理间隔 | (计算) | timeout/10 | 后台清理周期 |

## 10. 安全考虑

### 10.1 UUID 验证

```python
def _is_valid_uuid(val: str) -> bool:
    """防止注入攻击的 UUID 格式验证"""
    try:
        uuid.UUID(val)
        return True
    except ValueError:
        return False

def get_thread(thread_id: str) -> Optional[ThreadContext]:
    if not thread_id or not _is_valid_uuid(thread_id):
        return None  # 无效 ID 返回 None,不抛异常
    # ...
```

### 10.2 存储密钥命名

```python
key = f"thread:{thread_id}"  # 前缀隔离
```

### 10.3 异常处理

```python
try:
    storage = get_storage()
    data = storage.get(key)
    if data:
        return ThreadContext.model_validate_json(data)
except Exception:
    # 静默处理,避免泄露存储细节
    return None
```

## 11. Rust 迁移考虑

### 11.1 数据结构

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConversationTurn {
    pub role: String,
    pub content: String,
    pub timestamp: DateTime<Utc>,
    pub files: Option<Vec<String>>,
    pub images: Option<Vec<String>>,
    pub tool_name: Option<String>,
    pub model_provider: Option<String>,
    pub model_name: Option<String>,
    pub model_metadata: Option<serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThreadContext {
    pub thread_id: Uuid,
    pub parent_thread_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub last_updated_at: DateTime<Utc>,
    pub tool_name: String,
    pub turns: Vec<ConversationTurn>,
    pub initial_context: serde_json::Value,
}
```

### 11.2 存储后端

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::time::{Duration, Instant};
use tokio::time::interval;

struct StorageEntry {
    value: String,
    expires_at: Instant,
}

pub struct InMemoryStorage {
    store: RwLock<HashMap<String, StorageEntry>>,
}

impl InMemoryStorage {
    pub fn new(cleanup_interval: Duration) -> Arc<Self> {
        let storage = Arc::new(Self {
            store: RwLock::new(HashMap::new()),
        });

        // 启动后台清理任务
        let storage_clone = Arc::clone(&storage);
        tokio::spawn(async move {
            let mut interval = interval(cleanup_interval);
            loop {
                interval.tick().await;
                storage_clone.cleanup_expired();
            }
        });

        storage
    }

    pub fn setex(&self, key: String, ttl: Duration, value: String) {
        let mut store = self.store.write().unwrap();
        store.insert(key, StorageEntry {
            value,
            expires_at: Instant::now() + ttl,
        });
    }

    pub fn get(&self, key: &str) -> Option<String> {
        let store = self.store.read().unwrap();
        store.get(key).and_then(|entry| {
            if Instant::now() < entry.expires_at {
                Some(entry.value.clone())
            } else {
                None
            }
        })
    }

    fn cleanup_expired(&self) {
        let mut store = self.store.write().unwrap();
        let now = Instant::now();
        store.retain(|_, entry| entry.expires_at > now);
    }
}
```

### 11.3 文件收集算法

```rust
impl ThreadContext {
    pub fn get_file_list(&self) -> Vec<String> {
        let mut seen = std::collections::HashSet::new();
        let mut files = Vec::new();

        // Rust 的 rev() 自然支持反向迭代
        for turn in self.turns.iter().rev() {
            if let Some(ref turn_files) = turn.files {
                for file in turn_files {
                    if seen.insert(file.clone()) {
                        files.push(file.clone());
                    }
                }
            }
        }

        files
    }
}
```

### 11.4 依赖推荐

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
```
