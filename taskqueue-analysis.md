# Navidrome 任务队列系统代码分析报告

## 1. SQLite 持久化模块修正

### 1.1 事实错误修正

**❌ 错误说法**: 任务队列是唯一使用 SQLite 持久化的模块

**✅ 正确表述**: Navidrome 插件系统中有 **两个独立的 SQLite 持久化模块**：

| 模块 | 数据库文件 | 代码位置 | 用途 |
|------|-----------|----------|------|
| **任务队列** | `taskqueue.db` | `plugins/host_taskqueue.go:90-91` | 存储任务定义、状态、执行记录 |
| **键值存储** | `kvstore.db` | `plugins/host_kvstore.go:63-64` | 插件通用键值存储，支持过期时间 |

### 1.2 两个 SQLite 模块的对比

#### 任务队列 SQLite 配置
**文件位置**: `plugins/host_taskqueue.go:83-121`

```go
dbPath := filepath.Join(dataDir, "taskqueue.db")
db, err := sql.Open("sqlite3", dbPath+"?_busy_timeout=5000&_journal_mode=WAL&_foreign_keys=off")
db.SetMaxOpenConns(3)
db.SetMaxIdleConns(1)
```

**Schema**: `plugins/host_taskqueue.go:126-148`
- `queues` 表：队列配置（并发数、重试策略、保留时间等）
- `tasks` 表：任务实例（ID、队列名、负载、状态、尝试次数、下次运行时间等）
- `idx_tasks_dequeue` 索引：优化出队查询

#### 键值存储 SQLite 配置
**文件位置**: `plugins/host_kvstore.go:43-90`

```go
dbPath := filepath.Join(dataDir, "kvstore.db")
db, err := sql.Open("sqlite3", dbPath+"?_busy_timeout=5000&_journal_mode=WAL&_foreign_keys=off")
db.SetMaxOpenConns(3)
db.SetMaxIdleConns(1)
```

**Schema**: `plugins/host_kvstore.go:95-105`
- `kvstore` 表：键值对（key、value、size、created_at、expires_at）

### 1.3 数据库隔离设计

每个插件都有 **独立的数据库文件**，存储路径为：
```
<DataFolder>/plugins/<pluginName>/taskqueue.db
<DataFolder>/plugins/<pluginName>/kvstore.db
```

**设计意图**:
- 插件间数据完全隔离
- 单个插件数据库损坏不影响其他插件
- 便于独立备份和清理

---

## 2. 任务状态迁移图与详细路径

### 2.1 任务状态定义

**文件位置**: `plugins/host_taskqueue.go:36-40`

```go
const (
    taskStatusPending   = "pending"   // 待执行
    taskStatusRunning   = "running"   // 执行中
    taskStatusCompleted = "completed" // 已完成
    taskStatusFailed    = "failed"    // 失败（超过重试次数）
    taskStatusCancelled = "cancelled" // 已取消
)
```

### 2.2 完整状态迁移图

```
                      ┌─────────────────┐
                      │    pending      │◄──────────┐
                      └─────────────────┘           │
                                │                   │
                      dequeue   │                   │
                                ▼                   │
                      ┌─────────────────┐           │
                      │    running      │           │
                      └─────────────────┘           │
                                │                   │
          ┌───────────┬─────────┼─────────┬───────────┐
          │           │         │         │           │
          ▼           ▼         ▼         ▼           ▼
┌─────────────┐ ┌───────────┐ ┌───────┐ ┌──────┐ ┌──────────┐
│  completed  │ │  failed   │ │revert │ │close │ │ crash    │
│  (success)  │ │(retries>) │ │pending│ │pending│ │recovery │
└─────────────┘ └───────────┘ └───┬───┘ └───┬───┘ └───┬──────┘
                                  │         │         │
                                  └─────────┴─────────┘
                                            │
                                            ▼
                                    回到 pending 状态
```

### 2.3 三条"运行中回到待执行"路径详解

#### 路径 1：崩溃恢复 (Crash Recovery)

**触发时机**: 创建队列时（服务重启后）

**文件位置**: `plugins/host_taskqueue.go:234-241`

```go
// Reset stale running tasks from previous crash
now := time.Now().UnixMilli()
_, err = s.db.ExecContext(ctx, `
    UPDATE tasks SET status = ?, updated_at = ? WHERE queue_name = ? AND status = ?
`, taskStatusPending, now, name, taskStatusRunning)
```

**行为**:
- 将上一次崩溃时处于 `running` 状态的任务重置为 `pending`
- **不修改** `attempt` 计数（因为是外部中断，不是任务本身失败）
- 下次启动后这些任务会被重新调度

**重试计数影响**: ❌ 不回退

---

#### 路径 2：优雅关闭 (Graceful Shutdown)

**触发时机**: 调用 `Close()` 方法时

**文件位置**: `plugins/host_taskqueue.go:563-591`

```go
func (s *taskQueueServiceImpl) Close() error {
    // Cancel context to signal all goroutines
    s.cancel()
    
    // Wait for goroutines with timeout
    done := make(chan struct{})
    go func() {
        s.wg.Wait()
        close(done)
    }()
    
    select {
    case <-done:
    case <-time.After(shutdownTimeout):
        log.Warn("TaskQueue shutdown timed out", "plugin", s.pluginName)
    }
    
    // Mark running tasks as pending for recovery on next startup
    if s.db != nil {
        now := time.Now().UnixMilli()
        _, err := s.db.Exec(`UPDATE tasks SET status = ?, updated_at = ? WHERE status = ?`, 
            taskStatusPending, now, taskStatusRunning)
        // ...
        return s.db.Close()
    }
    return nil
}
```

**行为**:
1. 取消 context 通知所有 worker 停止
2. 等待 worker 完成当前任务（超时 10 秒）
3. 将仍处于 `running` 状态的任务重置为 `pending`
4. **不修改** `attempt` 计数

**重试计数影响**: ❌ 不回退

---

#### 路径 3：中断回滚 (Interrupt Rollback)

**触发时机**: 任务执行过程中 context 被取消（通常由优雅关闭触发）

有两个子场景：

##### 子场景 3a：限速等待时中断

**文件位置**: `plugins/host_taskqueue.go:418-426`

```go
// Enforce delay between task dispatches using a rate limiter.
if qs.limiter != nil {
    if err := qs.limiter.Wait(s.ctx); err != nil {
        // Context cancelled during wait — revert task to pending for recovery
        s.revertTaskToPending(taskID)
        return false
    }
}
```

##### 子场景 3b：回调执行后中断

**文件位置**: `plugins/host_taskqueue.go:428-436`

```go
// Invoke callback
message, callbackErr := s.invokeCallbackFn(s.ctx, queueName, taskID, payload, attempt)

// If context was cancelled (shutdown), revert task to pending for recovery
if s.ctx.Err() != nil {
    s.revertTaskToPending(taskID)
    return false
}
```

**核心回滚逻辑**: `plugins/host_taskqueue.go:490-498`

```go
// revertTaskToPending puts a running task back to pending status and decrements the attempt
// counter (used during shutdown to ensure the interrupted attempt doesn't count).
func (s *taskQueueServiceImpl) revertTaskToPending(taskID string) {
    now := time.Now().UnixMilli()
    _, err := s.db.Exec(`UPDATE tasks SET status = ?, attempt = MAX(attempt - 1, 0), updated_at = ? WHERE id = ? AND status = ?`, 
        taskStatusPending, now, taskID, taskStatusRunning)
    // ...
}
```

**关键行为**:
- 将任务状态从 `running` 改回 `pending`
- **递减 `attempt` 计数**（`MAX(attempt - 1, 0)`，确保不为负）
- 因为这次尝试是被外部中断的，不应该消耗重试次数

**重试计数影响**: ✅ 回退（-1）

---

### 2.4 其他状态迁移路径

#### 正常完成 (Completed)
**文件位置**: `plugins/host_taskqueue.go:446-452`
```go
func (s *taskQueueServiceImpl) completeTask(queueName, taskID, message string) {
    now := time.Now().UnixMilli()
    _, err := s.db.ExecContext(s.ctx, `UPDATE tasks SET status = ?, message = ?, updated_at = ? WHERE id = ?`, 
        taskStatusCompleted, message, now, taskID)
}
```
- 状态: `running` → `completed`
- 重试计数: 不变

#### 失败重试 (Retry)
**文件位置**: `plugins/host_taskqueue.go:472-488`
```go
// Exponential backoff: backoffMs * 2^(attempt-1)
backoff := qs.config.BackoffMs << (attempt - 1)
nextRunAt := now + backoff
_, err := s.db.ExecContext(s.ctx, `
    UPDATE tasks SET status = ?, next_run_at = ?, updated_at = ? WHERE id = ?
`, taskStatusPending, nextRunAt, now, taskID)
```
- 状态: `running` → `pending`（等待下次重试）
- 重试计数: 不变（`attempt` 已在出队时递增）
- 设置 `next_run_at` 实现指数退避

#### 失败终止 (Failed)
**文件位置**: `plugins/host_taskqueue.go:463-470`
```go
if attempt > maxRetries {
    _, err := s.db.ExecContext(s.ctx, `UPDATE tasks SET status = ?, message = ?, updated_at = ? WHERE id = ?`, 
        taskStatusFailed, message, now, taskID)
}
```
- 状态: `running` → `failed`
- 重试计数: 不变（已用完所有重试）

#### 取消 (Cancelled)
**文件位置**: `plugins/host_taskqueue.go:307-336`
```go
result, err := s.db.ExecContext(ctx, `
    UPDATE tasks SET status = ?, updated_at = ? WHERE id = ? AND status = ?
`, taskStatusCancelled, now, taskID, taskStatusPending)
```
- 状态: `pending` → `cancelled`
- 只能取消处于 `pending` 状态的任务

---

### 2.5 重试计数行为汇总表

| 路径 | 触发场景 | 状态迁移 | attempt 变化 |
|------|---------|----------|-------------|
| **崩溃恢复** | 服务重启 | `running` → `pending` | 不变 |
| **优雅关闭** | `Close()` 调用 | `running` → `pending` | 不变 |
| **中断回滚** | 执行中 context 取消 | `running` → `pending` | -1（回退） |
| **正常完成** | 回调成功 | `running` → `completed` | 不变 |
| **失败重试** | 回调失败但未超限 | `running` → `pending` | 不变（已递增） |
| **失败终止** | 回调失败且超限 | `running` → `failed` | 不变 |

**设计意图**:
- 崩溃和关闭是外部事件，不是任务本身的问题，所以不消耗重试次数
- 中断回滚发生在任务已经出队（`attempt` 已 +1）但尚未真正执行时，所以需要回退
- 任务本身失败时正常消耗重试次数

---

## 3. 完整决策链总结

### 3.1 任务出队与执行流程

```
pending 任务
    ↓
UPDATE tasks SET status = 'running', attempt = attempt + 1
WHERE id = (SELECT ... LIMIT 1)
RETURNING id, payload, attempt, max_retries
    ↓ (已原子性递增 attempt)
running 状态
    ↓
┌─ 限速等待 (如果配置了 delay_ms)
│   ├─ 上下文取消 → revertTaskToPending (attempt -1) → pending
│   └─ 成功 → 继续
    ↓
调用插件回调
    ↓
├─ 上下文取消 → revertTaskToPending (attempt -1) → pending
├─ 成功 → completeTask → completed
└─ 失败 →
    ├─ attempt > maxRetries → failed
    └─ 否则 → 设置 next_run_at → pending (等待重试)
```

### 3.2 关键设计决策

| 决策 | 说明 |
|------|------|
| **独立 SQLite 数据库** | 每个插件有独立的 taskqueue.db 和 kvstore.db，实现隔离 |
| **原子出队操作** | 使用 UPDATE ... RETURNING 原子性地标记任务为 running 并递增 attempt |
| **指数退避重试** | `backoffMs * 2^(attempt-1)`，最大 1 小时 |
| **中断回退计数** | 优雅关闭时的中断不消耗重试次数，attempt -1 |
| **崩溃不消耗重试** | 崩溃恢复时仅重置状态，不修改 attempt |
