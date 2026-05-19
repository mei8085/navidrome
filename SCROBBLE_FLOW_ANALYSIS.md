# Navidrome 播放记录上报流程完整分析（修正版）

## 1. 架构总览

```
                          ┌─────────────────────────────────┐
                          │        客户端播放事件           │
                          │   (StateStopped / Submit)      │
                          └─────────────────────────────────┘
                                          │
                                          ▼
                          ┌─────────────────────────────────┐
                          │   播放进度阈值检测 (332-340)    │
                          │  >= 50% 或 >= 240 秒           │
                          └─────────────────────────────────┘
                                          │
                      ┌───────────────────┴───────────────────┐
                      ▼                                       ▼
          ┌───────────────────────┐             ┌───────────────────────┐
          │  本地播放计数更新       │             │   dispatchScrobble    │
          │  (incPlay, 事务)       │             │   (分发层, 串行)      │
          └───────────────────────┘             └───────────────────────┘
                                                                  │
                                                  ┌───────────────┼───────────────┐
                                                  ▼               ▼               ▼
                                      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                                      │  Last.fm     │ │ ListenBrainz │ │  插件        │
                                      │  Buffered   │ │  Buffered    │ │  Buffered    │
                                      └──────────────┘ └──────────────┘ └──────────────┘
                                                  │               │               │
                                                  └───────────────┼───────────────┘
                                                                  ▼
                                                  ┌───────────────────────────────┐
                                                  │    写入 scrobble_buffer 表    │
                                                  │  UNIQUE(user_id, service,     │
                                                  │         media_file_id, play_time) │
                                                  └───────────────────────────────┘
                                                                  │
                                                                  ▼
                                                  ┌───────────────────────────────┐
                                                  │    发送唤醒信号给 Worker       │
                                                  └───────────────────────────────┘
                                                                  │
                                                                  ▼
                                                  ┌───────────────────────────────┐
                                                  │  后台 Worker 异步处理队列      │
                                                  │  (processUserQueue)           │
                                                  └───────────────────────────────┘
                                                                  │
                                                  ┌───────────────┼───────────────┐
                                                  ▼               ▼               ▼
                                      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                                      │  Last.fm API │ │ ListenBrainz │ │  插件 API    │
                                      │   外部调用   │ │   API 调用   │ │   调用       │
                                      └──────────────┘ └──────────────┘ └──────────────┘
                                                  │               │               │
                                                  └───────────────┼───────────────┘
                                                                  ▼
                                                  ┌───────────────────────────────┐
                                                  │   错误分类 & 重试判定          │
                                                  │  ┌─ 成功: Dequeue              │
                                                  │  ├─ ErrRetryLater: 5秒后重试   │
                                                  │  └─ 其他错误: Dequeue (丢弃)   │
                                                  └───────────────────────────────┘
```

---

## 2. 三层边界说明

### 2.1 分发层 (Dispatch Layer)

**边界**：`PlayTracker.dispatchScrobble()` → 各 `BufferedScrobbler.Scrobble()`

**职责**：
- 获取所有已激活的 scrobbler 适配器
- 检查用户是否已授权
- 串行调用每个适配器的 `Scrobble()` 方法
- 记录错误但不中断其他适配器

**关键代码**（`core/scrobbler/play_tracker.go:457-477`）：

```go
func (p *playTracker) dispatchScrobble(ctx context.Context, t *model.MediaFile, playTime time.Time) {
    if t.Artist == consts.UnknownArtist {
        log.Debug(ctx, "Ignoring external Scrobble for track with unknown artist", ...)
        return
    }

    allScrobblers := p.getActiveScrobblers()  // 获取所有已启用的适配器
    u, _ := request.UserFrom(ctx)
    scrobble := Scrobble{MediaFile: *t, TimeStamp: playTime}
    
    // ⚠️ 串行调用，不是并发！
    for name, s := range allScrobblers {
        if !s.IsAuthorized(ctx, u.ID) {  // 跳过未授权用户
            continue
        }
        log.Debug(ctx, "Buffering Scrobble", "scrobbler", name, "track", t.Title, ...)
        err := s.Scrobble(ctx, u.ID, scrobble)  // 逐个调用
        if err != nil {
            log.Error(ctx, "Error sending Scrobble", "scrobbler", name, ..., err)
            continue  // 只记录错误，不中断其他适配器
        }
    }
}
```

**分发方式**：✅ **串行调用**，使用普通 `for` 循环遍历 `map[string]Scrobbler`，无 goroutine、无 WaitGroup。

### 2.2 队列层 (Queue Layer)

**边界**：`BufferedScrobbler.Scrobble()` → 数据库 `scrobble_buffer` 表 → Worker 异步处理

**职责**：
- 将 scrobble 写入数据库缓冲表
- 发送唤醒信号给后台 worker
- 由后台 worker 异步处理实际的外部 API 调用

**关键代码**（`core/scrobbler/buffered_scrobbler.go:73-81`）：

```go
func (b *bufferedScrobbler) Scrobble(ctx context.Context, userId string, s Scrobble) error {
    // 直接写入数据库，无去重检查
    err := b.ds.ScrobbleBuffer(ctx).Enqueue(b.service, userId, s.ID, s.TimeStamp)
    if err != nil {
        return err  // 直接返回错误（如唯一约束冲突）
    }
    b.sendWakeSignal()  // 唤醒后台 worker
    return nil
}
```

### 2.3 数据库层 (Database Layer)

**边界**：`ScrobbleBufferRepository.Enqueue()` → SQLite INSERT 操作

**职责**：
- 执行数据库插入操作
- 依赖唯一约束实现去重
- 返回原始数据库错误

**关键代码**（`persistence/scrobble_buffer_repository.go:53-64`）：

```go
func (r *scrobbleBufferRepository) Enqueue(service, userId, mediaFileId string, playTime time.Time) error {
    ins := Insert(r.tableName).SetMap(map[string]any{
        "id":            id.NewRandom(),
        "user_id":       userId,
        "service":       service,
        "media_file_id": mediaFileId,
        "play_time":     playTime,
        "enqueue_time":  time.Now(),
    })
    _, err := r.executeSQL(ins)  // 执行 INSERT，违反约束时返回 SQLite 错误
    return err
}
```

---

## 3. 去重机制分析

### 3.1 数据库约束

**表结构**（`db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50`）：

```sql
CREATE TABLE scrobble_buffer_new
(
    user_id varchar NOT NULL,
    service varchar NOT NULL,
    media_file_id varchar NOT NULL,
    play_time datetime NOT NULL,
    enqueue_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    id varchar NOT NULL DEFAULT '',
    -- ✅ 唯一约束：四个维度的组合必须唯一
    CONSTRAINT scrobble_buffer_pk UNIQUE (user_id, service, media_file_id, play_time)
);
```

**唯一键含义**：

| 字段 | 说明 |
|------|------|
| `user_id` | 同一用户 |
| `service` | 同一服务（lastfm/listenbrainz/plugin-xxx） |
| `media_file_id` | 同一首歌 |
| `play_time` | 同一播放时间（秒级精度） |

### 3.2 各层去重行为

| 层级 | 去重机制 | 说明 |
|------|---------|------|
| **分发层** | ❌ 无 | 直接遍历分发，无任何去重检查 |
| **队列层** | ❌ 无 | 直接调用 `Enqueue`，不检查重复 |
| **数据库层** | ✅ 被动去重 | 依赖 `UNIQUE` 约束拒绝重复插入，返回原始错误 |

**去重流程**：

```
dispatchScrobble()
    ↓
[分发层] 无去重，直接调用 s.Scrobble()
    ↓
[队列层] BufferedScrobbler.Scrobble()
    ↓
[数据库层] 执行 INSERT
    ├─ 成功 → 返回 nil → 唤醒 Worker
    └─ 失败（唯一约束冲突）→ 返回 SQLite 错误
        ↓
[队列层] 直接返回错误
    ↓
[分发层] 记录错误日志 → continue 处理下一个适配器
```

### 3.3 重复场景分析

**场景 1：同一播放事件被多次触发**

如果 `playTime` 相同（例如同一秒内多次调用）：
- 第一次：成功插入
- 第二次：数据库拒绝，返回 "UNIQUE constraint failed" 错误
- 结果：✅ 去重成功

**场景 2：同一首歌在不同时间多次触发**

如果 `playTime` 不同（例如间隔几秒）：
- 第一次：成功插入（play_time = t1）
- 第二次：成功插入（play_time = t2）
- 结果：❌ 产生两条记录，都上报到外部服务

---

## 4. 重试机制分析

### 4.1 重试调度

**关键代码**（`core/scrobbler/buffered_scrobbler.go:99-113`）：

```go
func (b *bufferedScrobbler) run(ctx context.Context) {
    for {
        if !b.processQueue(ctx) {
            // 遇到 ErrRetryLater，5秒后重试
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

**重试策略**：
- 固定 **5秒** 间隔
- 无限重试直到成功或遇到不可恢复错误
- 无指数退避，无最大重试次数限制

### 4.2 错误处理逻辑

**关键代码**（`core/scrobbler/buffered_scrobbler.go:131-168`）：

```go
func (b *bufferedScrobbler) processUserQueue(ctx context.Context, userId string) bool {
    buffer := b.ds.ScrobbleBuffer(ctx)
    for {
        entry, err := buffer.Next(b.service, userId)
        if err != nil {
            log.Error(ctx, "Error reading from scrobble buffer", ...)
            return false
        }
        if entry == nil {
            return true  // 队列空，处理成功
        }
        s, ok := b.loader()
        if !ok {
            log.Warn(ctx, "Scrobbler not available, will retry later", ...)
            return false  // 插件不可用，稍后重试
        }
        
        err = s.Scrobble(ctx, entry.UserID, Scrobble{...})
        
        if errors.Is(err, ErrRetryLater) {
            // ✅ 可重试错误：保留在队列中，5秒后重试
            log.Warn(ctx, "Could not send scrobble. Will be retried", ...)
            return false
        }
        if err != nil {
            // ❌ 不可恢复错误：直接丢弃
            log.Error(ctx, "Error sending scrobble to service. Discarding", ...)
        }
        
        // 成功或不可恢复错误都出队
        err = buffer.Dequeue(entry)
        if err != nil {
            log.Error(ctx, "Error removing entry from scrobble buffer", ...)
            return false
        }
    }
}
```

**错误类型定义**（`core/scrobbler/interfaces.go:16-19`）：

```go
var (
    ErrNotAuthorized = errors.New("not authorized")   // 用户未授权，不入队
    ErrRetryLater    = errors.New("retry later")      // 可重试错误，保留队列
    ErrUnrecoverable = errors.New("unrecoverable")    // 不可恢复错误，丢弃
)
```

---

## 5. 各适配器重试判定差异

### 5.1 Last.fm 适配器

**关键代码**（`adapters/lastfm/agent.go:379-412`）：

```go
func (l *lastfmAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)
    }

    if s.Duration <= 30 {
        log.Debug(ctx, "Skipping Last.fm scrobble for short song", ...)
        return nil
    }
    
    err = l.client.scrobble(ctx, sk, ScrobbleInfo{...})
    if err == nil {
        return nil
    }
    
    var lfErr *lastFMError
    isLastFMError := errors.As(err, &lfErr)
    
    // 1. 非 Last.fm 特定错误（如网络错误）→ 重试
    if !isLastFMError {
        log.Warn(ctx, "Last.fm client.scrobble returned error", ...)
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 2. Last.fm 错误码 11（服务暂时不可用）或 16（临时错误）→ 重试
    if lfErr.Code == 11 || lfErr.Code == 16 {
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 3. 其他 Last.fm 错误 → 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**Last.fm 错误处理表**：

| 情况 | 错误码示例 | 处理方式 |
|------|-----------|---------|
| 网络错误、超时 | N/A | ✅ 重试 |
| 服务暂时不可用 | 11 | ✅ 重试 |
| 临时错误 | 16 | ✅ 重试 |
| 认证失败 | 4, 5, 6, 9, 14, 15 | ❌ 丢弃 |
| 无效参数 | 1, 2, 3, 6, 7, 8 | ❌ 丢弃 |
| 速率限制 | 25 | ❌ 丢弃 |
| 其他错误 | 10, 12, 13, 17-24 | ❌ 丢弃 |

### 5.2 ListenBrainz 适配器

**关键代码**（`adapters/listenbrainz/agent.go:91-114`）：

```go
func (l *listenBrainzAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)
    }

    li := l.formatListen(&s.MediaFile)
    li.ListenedAt = int(s.TimeStamp.Unix())
    err = l.client.scrobble(ctx, sk, li)

    if err == nil {
        return nil
    }
    
    var lbErr *listenBrainzError
    isListenBrainzError := errors.As(err, &lbErr)
    
    // 1. 非 ListenBrainz 特定错误（如网络错误）→ 重试
    if !isListenBrainzError {
        log.Warn(ctx, "ListenBrainz Scrobble returned HTTP error", ...)
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 2. HTTP 5xx 服务端错误 → 重试
    if lbErr.Code == 500 || lbErr.Code == 503 {
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 3. 其他错误 → 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**ListenBrainz 错误处理表**：

| 情况 | HTTP 状态码 | 处理方式 |
|------|------------|---------|
| 网络错误、超时 | N/A | ✅ 重试 |
| 服务器内部错误 | 500 | ✅ 重试 |
| 服务不可用 | 503 | ✅ 重试 |
| 成功 | 200 | 成功 |
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
    // ScrobblerErrorNotAuthorized indicates the user is not authorized.
    ScrobblerErrorNotAuthorized ScrobblerError = "scrobbler(not_authorized)"
    // ScrobblerErrorRetryLater indicates the operation should be retried later.
    ScrobblerErrorRetryLater    ScrobblerError = "scrobbler(retry_later)"
    // ScrobblerErrorUnrecoverable indicates an unrecoverable error.
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
    // 使用 strings.Contains 检查错误消息是否包含特定字面值
    case strings.Contains(errMsg, capabilities.ScrobblerErrorNotAuthorized.Error()):
        return scrobbler.ErrNotAuthorized
    case strings.Contains(errMsg, capabilities.ScrobblerErrorRetryLater.Error()):
        return scrobbler.ErrRetryLater
    case strings.Contains(errMsg, capabilities.ScrobblerErrorUnrecoverable.Error()):
        return scrobbler.ErrUnrecoverable
    default:
        return scrobbler.ErrUnrecoverable  // ⚠️ 默认不可恢复
    }
}
```

**插件错误处理表**：

| 插件返回错误消息包含 | 映射结果 |
|-------------------|---------|
| `"scrobbler(not_authorized)"` | `ErrNotAuthorized`（不入队） |
| `"scrobbler(retry_later)"` | `ErrRetryLater`（保留队列，5秒后重试） |
| `"scrobbler(unrecoverable)"` | `ErrUnrecoverable`（丢弃） |
| 其他任何错误消息 | `ErrUnrecoverable`（丢弃） |

> **注意**：插件错误映射基于 `strings.Contains` 字符串匹配，不是精确匹配。插件开发者必须确保返回的错误消息包含上述精确字面值。

---

## 6. 重试机制对比总结

| 维度 | Last.fm | ListenBrainz | 插件 |
|------|---------|-------------|------|
| **判定依据** | Last.fm 错误码 (11, 16) | HTTP 状态码 (500, 503) | 错误消息字符串匹配 |
| **网络错误** | ✅ 重试 | ✅ 重试 | ⚠️ 取决于插件是否返回 `scrobbler(retry_later)` |
| **服务不可用** | ✅ 重试（错误码11） | ✅ 重试（HTTP 503） | ⚠️ 取决于插件实现 |
| **认证失败** | ❌ 丢弃（错误码4,5,6,9） | ❌ 丢弃（HTTP 401） | ⚠️ 取决于插件是否返回 `scrobbler(not_authorized)` |
| **无效参数** | ❌ 丢弃（错误码1,2,3,6,7,8） | ❌ 丢弃（HTTP 400） | ⚠️ 取决于插件实现 |
| **重试间隔** | 5秒（固定） | 5秒（固定） | 5秒（固定） |
| **最大重试次数** | 无限制 | 无限制 | 无限制 |
| **指数退避** | 无 | 无 | 无 |
| **死信队列** | 无 | 无 | 无 |

---

## 7. 完整数据流转

```
1. 客户端 ReportPlayback(StateStopped)
   │
   ▼
2. 播放进度检测：position >= min(50%, 240秒) ?
   ├─ 否 → 结束
   └─ 是 → incPlay() 更新本地计数（事务）
   │
   ▼
3. dispatchScrobble()
   │
   ├─ getActiveScrobblers() 获取 [lastfm, listenbrainz, plugin1, plugin2, ...]
   │
   └─ for name, s := range allScrobblers {  // ⚠️ 串行，不是并发！
          if !s.IsAuthorized(ctx, u.ID) { continue }
          s.Scrobble(ctx, u.ID, scrobble)
      }
   │
   ▼
4. BufferedScrobbler.Scrobble()
   │
   ├─ Enqueue() → INSERT INTO scrobble_buffer
   │  ├─ 成功 → sendWakeSignal()
   │  └─ 失败（UNIQUE 冲突）→ 返回错误
   │     └─ dispatchScrobble() 记录错误，continue
   │
   ▼
5. 后台 Worker（每个 BufferedScrobbler 一个 goroutine）
   │
   └─ processQueue()
      │
      └─ processUserQueue(userId)
         │
         ├─ Next() 读取队列条目
         │
         ├─ s.Scrobble() 调用实际适配器
         │  │
         │  ├─ 成功 → Dequeue() 移除条目
         │  │
         │  ├─ ErrRetryLater → return false → 5秒后重试
         │  │
         │  └─ 其他错误 → Dequeue() 丢弃条目
         │
         └─ 循环直到队列空或遇到 ErrRetryLater
```

---

## 8. 关键代码索引

| 文件 | 说明 |
|------|------|
| `core/scrobbler/play_tracker.go:457-477` | 分发层入口 `dispatchScrobble` |
| `core/scrobbler/buffered_scrobbler.go:73-81` | 队列层 `BufferedScrobbler.Scrobble` |
| `core/scrobbler/buffered_scrobbler.go:99-113` | 重试调度器 `run` |
| `core/scrobbler/buffered_scrobbler.go:131-168` | 队列处理 `processUserQueue` |
| `persistence/scrobble_buffer_repository.go:53-64` | 数据库插入 `Enqueue` |
| `adapters/lastfm/agent.go:379-412` | Last.fm 重试判定 |
| `adapters/listenbrainz/agent.go:91-114` | ListenBrainz 重试判定 |
| `plugins/capabilities/scrobbler.go:123-136` | 插件错误字面值定义 |
| `plugins/scrobbler_adapter.go:169-186` | 插件错误映射 `mapScrobblerError` |
| `db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50` | 数据库唯一约束 |
