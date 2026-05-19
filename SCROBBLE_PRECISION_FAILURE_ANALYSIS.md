# Navidrome 播放记录链路：时间精度与失败边界深度分析

## 1. 核心问题清单

| 问题 | 结论 |
|------|------|
| `play_time` 写库精度 | SQLite 以 `YYYY-MM-DD HH:MM:SS` 格式存储，**秒级精度** |
| 唯一约束何时去重 | 同一用户+服务+歌曲+**同一秒**内重复触发时去重 |
| 唯一约束何时不去重 | 同一首歌在**不同秒数**触发时产生多条记录 |
| Enqueue 失败后是否重试 | ❌ 不重试，只记录日志后结束 |
| Worker 阶段 ErrRetryLater | ✅ 5秒后重试，无限重试直到成功或不可恢复错误 |

---

## 2. play_time 时间精度深度分析

### 2.1 时间产生链路

```go
// play_tracker.go:267
func (p *playTracker) ReportPlayback(ctx context.Context, params ReportPlaybackParams) error {
    now := time.Now()  // ① Go 程序层面：纳秒精度
    
    switch params.State {
    case StateStopped:
        // ...
        if params.PositionMs >= threshold {
            err = p.incPlay(ctx, mf, now)
            p.dispatchScrobble(ctx, mf, now)  // ② 传递给分发层（Go time.Time，纳秒级）
        }
    }
}

// play_tracker.go:465
func (p *playTracker) dispatchScrobble(ctx context.Context, t *model.MediaFile, playTime time.Time) {
    scrobble := Scrobble{MediaFile: *t, TimeStamp: playTime}  // ③ 封装为 Scrobble
    // ...
    s.Scrobble(ctx, u.ID, scrobble)
}

// scrobble_buffer_repository.go:53-64
func (r *scrobbleBufferRepository) Enqueue(service, userId, mediaFileId string, playTime time.Time) error {
    ins := Insert(r.tableName).SetMap(map[string]any{
        // ...
        "play_time":     playTime,  // ④ 直接传入 time.Time 类型给 ORM
        "enqueue_time":  time.Now(),
    })
    _, err := r.executeSQL(ins)
    return err
}
```

### 2.2 SQLite 时间存储格式

SQLite 的 `DATETIME` 类型以**字符串**形式存储，格式为：
```
YYYY-MM-DD HH:MM:SS
```

**精度**：**秒级**（没有毫秒或微秒部分）

**证据**：
- 迁移文件中 `play_time datetime NOT NULL`
- SQLite 没有原生的时间类型，`DATETIME` 是 TEXT 的别名
- 当 Go 的 `time.Time`（纳秒精度）写入 SQLite 时，ORM 会自动格式化为 `YYYY-MM-DD HH:MM:SS`，**纳秒部分被截断**

### 2.3 时间精度验证

```
Go time.Now() 实际值: 2026-05-19 14:30:25.123456789 +0800 CST m=+123.456789
                    └───────────────────────┘
                            │
                          写入 SQLite 时被截断为：
                            ↓
                      2026-05-19 14:30:25
```

**结论**：`play_time` 字段的实际有效精度是**秒级**。

---

## 3. 唯一约束去重边界分析

### 3.1 唯一键构成

```sql
CONSTRAINT scrobble_buffer_pk UNIQUE (user_id, service, media_file_id, play_time)
```

| 字段 | 含义 | 精度 |
|------|------|------|
| `user_id` | 用户 ID | 精确匹配 |
| `service` | 服务名称 | 精确匹配 |
| `media_file_id` | 歌曲 ID | 精确匹配 |
| `play_time` | 播放时间 | **秒级匹配** |

### 3.2 去重判定逻辑

```
是否去重 = (user_id 相同) AND (service 相同) AND (media_file_id 相同) AND (play_time 相同到秒)
```

### 3.3 场景分析

#### 场景 1：同一播放事件被重复调用（同一秒内）

```
第 1 次调用 ReportPlayback(StateStopped)
  now = 2026-05-19 14:30:25.100
  → dispatchScrobble(now)
  → play_time 写入 SQLite: 2026-05-19 14:30:25
  → ✅ 插入成功

第 2 次调用 ReportPlayback(StateStopped)
  now = 2026-05-19 14:30:25.200  (同一秒内)
  → dispatchScrobble(now)
  → play_time 写入 SQLite: 2026-05-19 14:30:25
  → ❌ UNIQUE constraint failed
  → 记录错误日志，不重试

结果：✅ 去重成功，只有第 1 条记录
```

#### 场景 2：同一首歌在不同秒数触发

```
第 1 次点击暂停: 14:30:25.100
  now = 2026-05-19 14:30:25.100
  → play_time: 2026-05-19 14:30:25
  → ✅ 插入成功（第 1 条）

第 2 次点击继续，然后又暂停: 14:30:26.500
  now = 2026-05-19 14:30:26.500
  → play_time: 2026-05-19 14:30:26  (不同秒数！)
  → ✅ 插入成功（第 2 条）

第 3 次再次触发: 14:30:27.800
  now = 2026-05-19 14:30:27.800
  → play_time: 2026-05-19 14:30:27  (不同秒数！)
  → ✅ 插入成功（第 3 条）

结果：❌ 不去重，产生 3 条记录，都上报到外部服务
```

#### 场景 3：不同服务的同一播放记录

```
dispatchScrobble(now)
  ├─ Last.fm 适配器
  │   service = "lastfm"
  │   play_time = 2026-05-19 14:30:25
  │   → ✅ 插入成功
  │
  └─ ListenBrainz 适配器
      service = "listenbrainz"
      play_time = 2026-05-19 14:30:25
      → ✅ 插入成功（service 不同，不触发唯一约束）

结果：✅ 两个服务各有一条记录，互不影响
```

### 3.4 去重边界总结表

| 场景 | user_id | service | media_file_id | play_time | 是否去重 |
|------|---------|---------|--------------|-----------|---------|
| 同一秒内重复触发 | 相同 | 相同 | 相同 | 相同 | ✅ 去重 |
| 不同秒数触发 | 相同 | 相同 | 相同 | 不同 | ❌ 不去重 |
| 不同用户 | 不同 | 相同 | 相同 | 相同 | ❌ 不去重 |
| 不同服务 | 相同 | 不同 | 相同 | 相同 | ❌ 不去重 |
| 不同歌曲 | 相同 | 相同 | 不同 | 相同 | ❌ 不去重 |

---

## 4. 失败路径边界完整分析

### 4.1 失败路径全景图

```
ReportPlayback(StateStopped)
        │
        ▼
dispatchScrobble()
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              路径 1：Enqueue 阶段失败                   │
└─────────────────────────────────────────────────────────┘
        │
        ├─ 成功 → 唤醒 Worker → 路径 2
        │
        └─ 失败（如 UNIQUE 冲突）→ 记录错误日志 → ❌ 直接结束，不重试
                    ▲
                    │ （不进入 Worker 阶段）
        ┌───────────┴─────────────────────────────────────┐
        │              路径 2：Worker 阶段失败             │
        └─────────────────────────────────────────────────┘
                    │
                    ▼
        processUserQueue()
                    │
                    ├─ 成功 → Dequeue → 结束
                    │
                    ├─ ErrRetryLater → 保留在队列 → ⏰ 5 秒后重试（无限循环）
                    │
                    └─ ErrUnrecoverable / 其他错误 → Dequeue → ❌ 丢弃，不重试
```

### 4.2 路径 1：Enqueue 阶段失败

**触发点**：`BufferedScrobbler.Scrobble()` → `ScrobbleBufferRepository.Enqueue()`

**可能的失败原因**：
1. **唯一约束冲突**：同一秒内重复触发
2. **外键约束失败**：`media_file_id` 不存在
3. **数据库连接错误**（临时故障）
4. **磁盘满/权限问题**（持久化故障）

**处理逻辑**（`buffered_scrobbler.go:73-81`）：

```go
func (b *bufferedScrobbler) Scrobble(ctx context.Context, userId string, s Scrobble) error {
    err := b.ds.ScrobbleBuffer(ctx).Enqueue(b.service, userId, s.ID, s.TimeStamp)
    if err != nil {
        return err  // ⚠️ 直接返回错误，不重试！
    }
    b.sendWakeSignal()
    return nil
}
```

**上层处理**（`play_tracker.go:471-474`）：

```go
err := s.Scrobble(ctx, u.ID, scrobble)
if err != nil {
    // ⚠️ 只记录错误日志，然后 continue 处理下一个适配器
    log.Error(ctx, "Error sending Scrobble", "scrobbler", name, ..., err)
    continue
}
```

**结论**：
- ✅ **不重试**
- ✅ **不阻塞其他适配器**
- ✅ **只记录错误日志后结束**
- ⚠️ **不会进入 Worker 队列**，因此永远不会被重试

### 4.3 路径 2：Worker 阶段失败

**触发点**：`BufferedScrobbler.run()` → `processQueue()` → `processUserQueue()`

**可能的失败原因**：
1. **网络错误**（连接超时、DNS 失败等）
2. **外部服务不可用**（503、500 等）
3. **外部服务限流**（429）
4. **认证失败**（401、403）
5. **无效参数**（400）

**处理逻辑**（`buffered_scrobbler.go:131-168`）：

```go
func (b *bufferedScrobbler) processUserQueue(ctx context.Context, userId string) bool {
    buffer := b.ds.ScrobbleBuffer(ctx)
    for {
        entry, err := buffer.Next(b.service, userId)
        if err != nil {
            log.Error(ctx, "Error reading from scrobble buffer", ...)
            return false  // ⚠️ 读取失败也触发重试
        }
        if entry == nil {
            return true  // 队列空，处理成功
        }
        s, ok := b.loader()
        if !ok {
            log.Warn(ctx, "Scrobbler not available, will retry later", ...)
            return false  // ⚠️ 插件不可用，触发重试
        }
        
        err = s.Scrobble(ctx, entry.UserID, Scrobble{...})  // 调用实际适配器
        
        // ⚠️ 关键：错误类型决定处理方式
        if errors.Is(err, ErrRetryLater) {
            // ✅ 可重试错误：保留在队列中，5秒后重试
            log.Warn(ctx, "Could not send scrobble. Will be retried", ...)
            return false  // 返回 false，触发 5 秒后重试
        }
        if err != nil {
            // ❌ 不可恢复错误：直接丢弃
            log.Error(ctx, "Error sending scrobble to service. Discarding", ...)
        }
        
        // 成功或不可恢复错误都出队
        err = buffer.Dequeue(entry)
        if err != nil {
            log.Error(ctx, "Error removing entry from scrobble buffer", ...)
            return false  // Dequeue 失败也触发重试（条目未被删除）
        }
    }
}
```

**重试调度**（`buffered_scrobbler.go:99-113`）：

```go
func (b *bufferedScrobbler) run(ctx context.Context) {
    for {
        if !b.processQueue(ctx) {
            // ⏰ 5 秒后重试（固定间隔，无指数退避）
            time.AfterFunc(5*time.Second, func() {
                b.sendWakeSignal()
            })
        }
        select {
        case <-b.wakeSignal:
            continue
        case <-ctx.Done():
            return
        }
    }
}
```

**结论**：
- ✅ `ErrRetryLater`：保留在队列，**5 秒后重试**，无限循环
- ✅ 其他错误：从队列移除，**丢弃**，不重试
- ✅ 读取队列失败/插件不可用/Dequeue 失败：也触发 5 秒后重试
- ⚠️ **最大重试次数**：无限制，理论上可能无限重试同一条记录

### 4.4 两条路径对比表

| 维度 | Enqueue 阶段失败 | Worker 阶段失败（ErrRetryLater） | Worker 阶段失败（其他错误） |
|------|-----------------|--------------------------------|---------------------------|
| **触发位置** | 分发→入队时 | Worker 调用外部 API 时 | Worker 调用外部 API 时 |
| **是否入队** | ❌ 不入队 | ✅ 已入队，保留 | ✅ 已入队，然后出队 |
| **重试机制** | ❌ 不重试 | ✅ 5 秒后重试（无限） | ❌ 不重试，丢弃 |
| **错误日志** | ✅ 记录 | ✅ 记录（Warn 级别） | ✅ 记录（Error 级别） |
| **阻塞其他适配器** | ❌ 不阻塞 | ❌ 不阻塞 | ❌ 不阻塞 |
| **数据丢失风险** | ⚠️ 可能丢失（如临时数据库故障） | ✅ 不会丢失（持久化了） | ⚠️ 永久丢失 |
| **典型原因** | 唯一约束冲突、数据库连接错误 | 网络错误、503、500 | 400、401、403、无效参数 |

---

## 5. 各适配器重试判定的代码级差异

### 5.1 Last.fm 适配器

**代码位置**：`adapters/lastfm/agent.go:379-412`

```go
func (l *lastfmAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    // 检查 Session Key
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)  // → 不入队
    }

    // 跳过短歌曲（<30 秒）
    if s.Duration <= 30 {
        log.Debug(ctx, "Skipping Last.fm scrobble for short song", ...)
        return nil  // → 成功但不上报
    }
    
    // 调用 Last.fm API
    err = l.client.scrobble(ctx, sk, ScrobbleInfo{...})
    if err == nil {
        return nil  // → 成功
    }
    
    var lfErr *lastFMError
    isLastFMError := errors.As(err, &lfErr)
    
    // 判定逻辑
    if !isLastFMError {
        // 1. 非 Last.fm 错误（网络错误等）→ 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    if lfErr.Code == 11 || lfErr.Code == 16 {
        // 2. Last.fm 错误码 11（服务不可用）或 16（临时错误）→ 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    // 3. 其他 Last.fm 错误 → 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**Last.fm 重试判定表**：

| 错误类型 | 错误码 | 处理 |
|---------|--------|------|
| 网络错误、超时、DNS 失败 | N/A | ✅ 重试 |
| 服务暂时不可用 | 11 | ✅ 重试 |
| 临时错误 | 16 | ✅ 重试 |
| 无效参数 | 1, 2, 3, 6, 7, 8 | ❌ 丢弃 |
| 认证失败 | 4, 5, 9, 14, 15 | ❌ 丢弃 |
| 速率限制 | 25 | ❌ 丢弃 |
| 其他错误 | 10, 12, 13, 17-24 | ❌ 丢弃 |

### 5.2 ListenBrainz 适配器

**代码位置**：`adapters/listenbrainz/agent.go:91-114`

```go
func (l *listenBrainzAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    // 检查 Session Key
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)  // → 不入队
    }

    // 调用 ListenBrainz API
    li := l.formatListen(&s.MediaFile)
    li.ListenedAt = int(s.TimeStamp.Unix())
    err = l.client.scrobble(ctx, sk, li)

    if err == nil {
        return nil  // → 成功
    }
    
    var lbErr *listenBrainzError
    isListenBrainzError := errors.As(err, &lbErr)
    
    // 判定逻辑
    if !isListenBrainzError {
        // 1. 非 ListenBrainz 错误（网络错误等）→ 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    if lbErr.Code == 500 || lbErr.Code == 503 {
        // 2. HTTP 5xx 服务端错误 → 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    // 3. 其他错误（4xx 客户端错误等）→ 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**ListenBrainz 重试判定表**：

| 错误类型 | HTTP 状态码 | 处理 |
|---------|------------|------|
| 网络错误、超时、DNS 失败 | N/A | ✅ 重试 |
| 服务器内部错误 | 500 | ✅ 重试 |
| 服务不可用 | 503 | ✅ 重试 |
| 无效请求 | 400 | ❌ 丢弃 |
| 未授权 | 401 | ❌ 丢弃 |
| 禁止访问 | 403 | ❌ 丢弃 |
| 未找到 | 404 | ❌ 丢弃 |
| 请求过多 | 429 | ❌ 丢弃 |

### 5.3 插件适配器

**错误字面值定义**（`plugins/capabilities/scrobbler.go:123-136`）：

```go
type ScrobblerError string

const (
    ScrobblerErrorNotAuthorized ScrobblerError = "scrobbler(not_authorized)"
    ScrobblerErrorRetryLater    ScrobblerError = "scrobbler(retry_later)"
    ScrobblerErrorUnrecoverable ScrobblerError = "scrobbler(unrecoverable)"
)

func (e ScrobblerError) Error() string { return string(e) }
```

**错误映射逻辑**（`plugins/scrobbler_adapter.go:169-186`）：

```go
func mapScrobblerError(err error) error {
    if err == nil {
        return nil
    }
    errMsg := err.Error()
    switch {
    // 使用 strings.Contains 检查（包含即可，不是精确匹配！）
    case strings.Contains(errMsg, "scrobbler(not_authorized)"):
        return scrobbler.ErrNotAuthorized
    case strings.Contains(errMsg, "scrobbler(retry_later)"):
        return scrobbler.ErrRetryLater
    case strings.Contains(errMsg, "scrobbler(unrecoverable)"):
        return scrobbler.ErrUnrecoverable
    default:
        return scrobbler.ErrUnrecoverable  // ⚠️ 默认不可恢复
    }
}
```

**插件重试判定表**：

| 插件返回的错误消息包含 | 映射结果 | 处理 |
|-------------------|---------|------|
| `"scrobbler(not_authorized)"` | `ErrNotAuthorized` | 不入队 |
| `"scrobbler(retry_later)"` | `ErrRetryLater` | 5 秒后重试（无限） |
| `"scrobbler(unrecoverable)"` | `ErrUnrecoverable` | 丢弃 |
| 其他任何错误消息 | `ErrUnrecoverable` | 丢弃 |

> **重要注意**：
> 1. 使用 `strings.Contains`，不是精确匹配！只要错误消息**包含**上述字面值即可
> 2. 插件开发者必须确保返回的错误消息包含精确的字面值
> 3. 没有包含特殊字面值的错误默认映射为 `ErrUnrecoverable`（丢弃）

### 5.4 三个适配器对比总结

| 维度 | Last.fm | ListenBrainz | 插件 |
|------|---------|-------------|------|
| **判定依据** | 错误码 11, 16 | HTTP 500, 503 | 错误消息字符串匹配 |
| **网络错误** | ✅ 重试 | ✅ 重试 | ⚠️ 取决于插件实现 |
| **服务不可用** | ✅ 重试（码11） | ✅ 重试（码503） | ⚠️ 取决于插件实现 |
| **认证失败** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **无效参数** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **默认行为** | 不明确定义的错误 → 丢弃 | 不明确定义的错误 → 丢弃 | 不明确定义的错误 → 丢弃 |
| **匹配方式** | 精确错误码匹配 | 精确状态码匹配 | 字符串包含匹配 |

---

## 6. 完整失败处理流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ReportPlayback(StateStopped)                         │
│                              now = time.Now()                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                        检查 position >= 阈值 (50% 或 240秒)
                                      │
                                      ├─ 否 → 结束
                                      │
                                      └─ 是 → incPlay() + dispatchScrobble()
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           dispatchScrobble()                               │
│  for name, s := range allScrobblers {  // ⚠️ 串行，不是并发！               │
│      if !s.IsAuthorized(ctx, u.ID) { continue }                            │
│      err := s.Scrobble(ctx, u.ID, scrobble)                                 │
│      if err != nil { log.Error(...); continue }  // ⚠️ 失败不重试！          │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┴───────────────────────────┐
          ▼                                                       ▼
┌──────────────────────┐                              ┌──────────────────────┐
│  适配器 1：Last.fm   │                              │  适配器 N：插件      │
│  BufferedScrobbler   │       ...                    │  BufferedScrobbler   │
└──────────────────────┘                              └──────────────────────┘
          │                                                       │
          └───────────────────────────┬───────────────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BufferedScrobbler.Scrobble()                         │
│  err := Enqueue(...)  // 写入 scrobble_buffer 表                            │
│  if err != nil { return err }  // ⚠️ 路径 1：Enqueue 失败                    │
│  sendWakeSignal()  // 唤醒后台 Worker                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┴───────────────────────────┐
          ▼                                                       ▼
┌──────────────────────┐                              ┌──────────────────────┐
│  Enqueue 成功        │                              │  Enqueue 失败        │
│  → 进入路径 2        │                              │  → 路径 1 结束       │
│  → 唤醒 Worker       │                              │  → 记录错误日志      │
│                      │                              │  → ❌ 不重试！       │
└──────────────────────┘                              └──────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           后台 Worker 循环                                  │
│  for {                                                                      │
│      if !processQueue(ctx) {                                                │
│          time.AfterFunc(5*time.Second, sendWakeSignal)  // ⏰ 5秒后重试     │
│      }                                                                      │
│      select { case <-wakeSignal: continue; case <-ctx.Done(): return }       │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                        调用实际适配器的 Scrobble() 方法
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  成功                │  │  ErrRetryLater       │  │  其他错误            │
│  → Dequeue()         │  │  → return false      │  │  → Dequeue()         │
│  → 处理下一条        │  │  → ⏰ 5秒后重试      │  │  → ❌ 丢弃           │
│                      │  │  → 无限循环          │  │                      │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

---

## 7. 关键代码索引

| 文件 | 说明 |
|------|------|
| `core/scrobbler/play_tracker.go:267` | `now = time.Now()` 时间产生点 |
| `core/scrobbler/play_tracker.go:457-477` | `dispatchScrobble` 分发逻辑 |
| `core/scrobbler/buffered_scrobbler.go:73-81` | `Scrobble` 入队逻辑 |
| `core/scrobbler/buffered_scrobbler.go:99-113` | 重试调度器 `run` |
| `core/scrobbler/buffered_scrobbler.go:131-168` | `processUserQueue` 错误处理 |
| `persistence/scrobble_buffer_repository.go:53-64` | `Enqueue` 数据库插入 |
| `adapters/lastfm/agent.go:379-412` | Last.fm 重试判定 |
| `adapters/listenbrainz/agent.go:91-114` | ListenBrainz 重试判定 |
| `plugins/capabilities/scrobbler.go:123-136` | 插件错误字面值定义 |
| `plugins/scrobbler_adapter.go:169-186` | 插件错误映射 `mapScrobblerError` |
| `db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50` | 数据库唯一约束定义 |
