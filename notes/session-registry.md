# 会话注册表：map、恢复、三层存储

thread 从哪来（resume 三路径）、map 什么时候写、历史存在哪一层。

## 1. 对象链

```
ThreadManager（薄句柄）
  └─ ThreadManagerState.threads: Arc<RwLock<HashMap<ThreadId, Arc<CodexThread>>>>   thread_manager.rs:374
       └─ CodexThread（session + io + rollout_path + session_source）
            └─ Session（active_turn: Mutex<Option<ActiveTurn>>、input_queue、services）
                 └─ SessionState.history: ContextManager
                      └─ items: Arc<Vec<ResponseItemEnvelope>>   history.rs:73
```

map 里的 value 是 `Arc<CodexThread>`，跟 Go 的 `map[ID]*Thread` 一一对应：存指针，多个持有者共享同一个堆上实例。

## 2. map 什么时候写

**只在线程创建 / resume 时写一次，turn 级别永远不写。**后续轮次直接路由到 map 里的活实例，map 键值对的指针从不换——Session 在 Arc 内部自己变（`Mutex<SessionState>`）。

插入点是 `codex-rs/core/src/thread_manager.rs:2196-2211`，用的不是「先查再插」而是 `Entry`：

```rust
{
    let mut threads = self.threads.write().await;
    if let std::collections::hash_map::Entry::Vacant(e) = threads.entry(thread_id) {
        let thread = Arc::new(CodexThread::new(
            session,
            io,
            session_configured.clone(),
            session_configured.rollout_path.clone(),
            session_source,
        ));
        e.insert(thread.clone());
        return Ok(NewThread {
            thread_id,
            thread,
            session_configured,
        });
    }
}

if let Err(err) = io.shutdown_and_wait().await {
    warn!("failed to shut down duplicate thread {thread_id}: {err}");
}
Err(CodexErr::InvalidRequest(format!(
    "thread {thread_id} is already running"
)))
```

锁内原子地「检查空位→落堆→插入」。输掉竞态的一方（两个连接同时 resume 同一个线程）要把自己刚建的重复实例关掉再报错（:2213-2222）。Go 的 `sync.Map.LoadOrStore` 就是这个语义。

## 3. 删除：只有显式事件，没有「对话结束就删」

删 map 条目的路径全是生命周期事件：archive/delete RPC、revert 重载、resume 替换死条目、延迟卸载、子代理驱逐、进程退出。

「延迟卸载」最讲究，`codex-rs/app-server/src/request_processors/thread_lifecycle.rs:56-59`：

```rust
fn unloading_target(&self) -> Option<Instant> {
    match (
        (self.no_subscribers, self.no_subscribers_since),
        (self.is_inactive, self.is_inactive_since),
    ) {
        ((false, has_no_subscribers_since), (false, is_inactive_since)) => {
            std::cmp::max(has_no_subscribers_since, is_inactive_since).checked_add(self.delay)
        }
        _ => None,
    }
}
```

无订阅 + 不活跃，两个条件取**较晚满足的那个**，再加 `thread_unload_delay`（:15）。空闲 ≠ 可卸载——还得没人看它，且过了一段缓冲期。

删除 ≠ 销毁：`Arc` 引用计数不归零，对象就还活着（`thread_manager.rs` 的 `remove_thread` 文档原话：*"Removes the thread from the manager's internal map, though the thread is stored as `Arc<CodexThread>`, it is possible that other references to it exist elsewhere"*）。和 Go 的 `delete(m, k)` 一样——删的只是索引，对象由 GC/引用计数管。

## 4. resume 三路径

`thread/resume` 进来时（`codex-rs/app-server/src/request_processors/thread_processor.rs:4180`），先查 map：

```rust
} else if let Ok(existing_thread_id) = ThreadId::from_string(&params.thread_id)
    && let Ok(existing_thread) = self.thread_manager.get_thread(existing_thread_id).await
{
    let source_thread = self
        .read_stored_thread_for_resume(
            &params.thread_id,
            /*path*/ None,
            /*include_history*/ false,
        )
        .await?;
    Some((existing_thread_id, existing_thread, source_thread))
}
```

（:4201-4213）map 命中 → 直接复用现有实例，连 `include_history` 都是 `false`——历史根本不用读，内存里就有。

| 路径 | 查 map | 查 SQLite | 读 JSONL | 落堆入 map |
|---|---|---|---|---|
| 热：线程活着（含空闲） | ✅ 命中 | ❌ | ❌ | ❌ |
| 冷：不在 map | ✅ 未命中 | ✅ 只拿元数据 | ✅ 历史本体 | ✅ |
| 新对话 `thread/start` | — | ❌ 无历史 | ❌ | ✅ 空历史 |

「活着」的定义是 `codex-rs/core/src/codex_thread.rs:689-691`：

```rust
pub(crate) fn is_running(&self) -> bool {
    !self.io.tx_sub.is_closed()
}
```

事件通道没关就算活着——空闲但已加载的线程命中热路径。只有 io 通道已关的死条目才会在 `thread_manager.rs:2013` 被 `threads.remove` 掉换个新的。

### 冷路径：SQLite 是索引，JSONL 才是历史

1. `read_stored_thread_for_resume`（`thread_processor.rs:4522`）→ `read_thread` → `read_sqlite_metadata`（`codex-rs/thread-store/src/local/read_thread.rs:309`）——`SELECT threads WHERE id=?`，只拿 rollout_path/cwd/model/archived_at 这些**元数据**
2. 历史本体从 rollout JSONL 读：Legacy 模式 `load_history_items`（`read_thread.rs:298`）全量逐行；Paginated 模式 `load_latest_model_context`（`codex-rs/thread-store/src/local/model_context.rs:37`）从文件尾**反向扫**，找到最近的 replacement-history checkpoint 就停（:29-33 文档注释）
3. 包成 `InitialHistory::Resumed`（`thread_processor.rs:4486`）
4. `spawn_thread` 写锁内**再查一次 map** 防竞态（`thread_manager.rs:1985-2015`），然后才落堆插入

为什么历史不存 SQLite：`codex-rs/thread-store/src/local/live_writer.rs:341-345` 的注释原话——

```rust
// SQLite is a rebuildable view. The flush barrier must win before projection starts so it
// can lag JSONL after failure, but can never get ahead of canonical history.
```

JSONL 是 canonical，SQLite 是可重建的投影，坏了可以从 JSONL 重放。恢复模型上下文永远走 JSONL；SQLite 里的 items 投影表只服务 UI 列表/搜索。

## 5. 启动时：什么都不加载

`ThreadManager::new` 里 map 生来是空的，`codex-rs/core/src/thread_manager.rs:514-516`：

```rust
Self {
    state: Arc::new(ThreadManagerState {
        threads: Arc::new(RwLock::new(HashMap::new())),
```

构造函数从上到下只装配周边服务（skills/plugins/MCP/models），零线程读取。UI 列表 `thread/list` 直接查 store 做展示，不碰 map。稳态是：SQLite 躺着几千个历史线程，map 里只有一两个在用的。

为什么不全量加载：map 条目不是「数据」是**活运行时**（Session + 事件通道 + rollout writer 占着文件句柄和写锁）。启动全量加载等于为几百个大概率不再打开的会话各开一套运行时。延迟卸载机制（§3）的存在反过来证明设计意图就是让 map 保持小。

## 6. 三层存储与写入节奏

| 层 | 角色 | 关键点 |
|---|---|---|
| 内存 `ContextManager` | 工作副本 | COW（`items: Arc<Vec<...>>`，`history.rs:73`）；`history_version` 在压缩/回滚时递增（:81） |
| rollout JSONL | **canonical** | 事实之源，事件粒度追加 |
| SQLite | 可重建投影 | `live_writer.rs:341-345`；threads 表元数据 + items 投影 |

写入发生在**内层循环**的 `record_prepared_conversation_items`（`codex-rs/core/src/session/mod.rs:3531`）——每个对话事件（用户消息、assistant 消息、工具调用、工具输出……）都是三连写：

```rust
{
    let mut state = self.state.lock().await;
    state
        .current_time_reminder
        .note_recorded_items(&response_items);
    state
        .history
        .record_annotated_items(&items, model_info.truncation_policy.into());
}
```

（:3564-3571，内存）然后：

```rust
let rollout_items: Vec<RolloutItem> =
    items.into_iter().map(RolloutItem::ResponseItem).collect();
self.persist_rollout_items(&rollout_items).await;
```

（:3578-3579，JSONL → 投影）。外层循环零写盘。

两个限流细节：

- `PersistContext`（`codex-rs/thread-store/src/store.rs:66`）：`TurnStart`/`SteeredUserInput` 允许后台异步落盘，`Standard` 同步——回合边界的批量写可以慢一点，回合中的写要稳
- threads 表的 `updated_at` 触碰节流 5 秒：`codex-rs/thread-store/src/thread_metadata_sync.rs:27` 的 `THREAD_UPDATED_AT_TOUCH_INTERVAL: Duration = Duration::from_secs(5)`——不然每个事件都 UPDATE 一次元数据表

## 7. Go 翻译

```go
// resume 的三分支
func (tm *ThreadManager) Resume(id ThreadID) (*Thread, error) {
	if t, ok := tm.threads.Load(id); ok && t.IOAlive() { // is_running = 事件通道没关
		return t.(*Thread), nil // 热路径：零 IO，map 不动
	}
	meta, err := tm.stateDB.GetThread(id) // 冷路径：SQLite 只当索引用
	if err != nil {
		return nil, err
	}
	items, err := LoadRolloutItems(meta.RolloutPath) // 历史本体在 JSONL
	if err != nil {
		return nil, err
	}
	t := NewThread(id, items, meta) // 落堆
	if _, loaded := tm.threads.LoadOrStore(id, t); loaded { // Entry::Vacant 的原子语义
		t.ShutdownAndWait() // 输掉竞态：丢弃新实例
		return nil, ErrAlreadyRunning
	}
	return t, nil
}
```

心智模型：不是「启动时把 DB 镜像进内存」，是 **SQLite/JSONL 冷的全量、map 热的按需**——类似 demand paging，resume 未加载的线程才「调页」。

## 8. 没验证的

- `thread/resume` 热路径里那次 `read_stored_thread_for_resume(include_history: false)` 读的是 SQLite 还是带缓存，没细查（推断是为了拿 history_mode 和校验 rollout path）。
- `load_history`（`thread-store/src/local/mod.rs`）开头有个 live-writer 优先分支，它与 app-server 层 map 检查的完整调用关系没展开。
