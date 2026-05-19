# Navidrome 播放记录去重机制与重试策略深度分析

## 1. 核心结论摘要

| 层面 | 去重机制 | 说明 |
|------|---------|------|
| **数据库层** | ✅ 已实现 | `UNIQUE (user_id, service, media_file_id, play_time)` 唯一约束 |
| **分发层** | ❌ 无显式去重 | 遍历所有适配器直接分发，依赖数据库层去重 |
| **队列层** | ✅ 被动去重 | 插入时违反唯一约束会报错，调用方忽略错误 |

**重试策略差异**：

| 适配器 | 重试条件 | 丢弃条件 | 重试间隔 |
|--------|---------|---------|---------|
| **Last.fm** | 网络错误 / 错误码 11, 16 | 其他业务错误 | 5秒（固定） |
| **ListenBrainz** | 网络错误 / HTTP 500, 503 | 其他 HTTP 错误 | 5秒（固定） |
| **插件** | 错误消息含 "retry later" | 其他错误消息 | 5秒（固定） |

---

## 2. 数据库约束分析

### 2.1 表结构演变

**初始创建**（`db/migrations/20210626213026_add_scrobble_buffer.go:14-35`）：

```sql
create table if not exists scrobble_buffer
(
    user_id varchar not null
        constraint scrobble_buffer_user_id_fk
            references user on update cascade on delete cascade,
    service varchar not null,
    media_file_id varchar not null
        constraint scrobble_buffer_media_file_id_fk
            references media_file on update cascade on delete cascade,
    play_time datetime not null,
    enqueue_time datetime not null default current_timestamp,
    constraint scrobble_buffer_pk
        unique (user_id, service, media_file_id, play_time, user_id)  -- 注意：user_id 重复了！
);
```

**Bug 修复**（`db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50`）：

```sql
-- 修复 scrobble_buffer 表：从唯一约束中移除重复的 user_id
-- 原始约束: UNIQUE (user_id, service, media_file_id, play_time, user_id)
-- 修复后约束: UNIQUE (user_id, service, media_file_id, play_time)
CREATE TABLE scrobble_buffer_new
(
    user_id varchar NOT NULL
        CONSTRAINT scrobble_buffer_user_id_fk
            REFERENCES user ON UPDATE CASCADE ON DELETE CASCADE,
    service varchar NOT NULL,
    media_file_id varchar NOT NULL
        CONSTRAINT scrobble_buffer_media_file_id_fk
            REFERENCES media_file ON UPDATE CASCADE ON DELETE CASCADE,
    play_time datetime NOT NULL,
    enqueue_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    id varchar NOT NULL DEFAULT '',
    CONSTRAINT scrobble_buffer_pk UNIQUE (user_id, service, media_file_id, play_time)
);
```

### 2.2 唯一键含义

`UNIQUE (user_id, service, media_file_id, play_time)` 表示：

- **同一用户** (`user_id`)
- **同一服务** (`service`)
- **同一首歌** (`media_file_id`)
- **同一播放时间** (`play_time`)

这四个维度的组合必须唯一，确保不会重复上报相同的播放记录。

### 2.3 时间精度问题

**注意**：`play_time` 是 `datetime` 类型，在 SQLite 中精度为秒级。

如果用户在**同一秒内**对同一首歌触发多次 scrobble（理论上可能但概率极低），数据库会自动去重。

---

## 3. 分发层去重分析

### 3.1 分发逻辑

**核心代码**：`core/scrobbler/play_tracker.go:457-477`

```go
func (p *playTracker) dispatchScrobble(ctx context.Context, t *model.MediaFile, playTime time.Time) {
    if t.Artist == consts.UnknownArtist {
        log.Debug(ctx, "Ignoring external Scrobble for track with unknown artist", "track", t.Title, "artist", t.Artist)
        return
    }

    allScrobblers := p.getActiveScrobblers()  // 获取所有已启用的适配器
    u, _ := request.UserFrom(ctx)
    scrobble := Scrobble{MediaFile: *t, TimeStamp: playTime}
    
    for name, s := range allScrobblers {
        if !s.IsAuthorized(ctx, u.ID) {  // 跳过未授权用户
            continue
        }
        log.Debug(ctx, "Buffering Scrobble", "scrobbler", name, "track", t.Title, "artist", t.Artist)
        err := s.Scrobble(ctx, u.ID, scrobble)  // 发送到各适配器
        if err != nil {
            log.Error(ctx, "Error sending Scrobble", "scrobbler", name, "track", t.Title, "artist", t.Artist, err)
            continue  // 只记录错误，不中断其他适配器
        }
    }
}
```

### 3.2 分发层特点

| 特点 | 说明 |
|------|------|
| **无显式去重** | 没有内存级别的去重缓存或幂等检查 |
| **并发分发** | 遍历所有适配器，逐个调用 `Scrobble` 方法 |
| **错误隔离** | 单个适配器失败不影响其他适配器 |
| **依赖下层** | 去重完全依赖 `BufferedScrobbler` 和数据库层 |

### 3.3 重复触发场景分析

**场景 1：同一首歌快速暂停/播放多次**

如果用户在短时间内对同一首歌多次触发 `StateStopped` 且都满足阈值条件：

1. 第一次调用 `dispatchScrobble`，所有适配器收到 scrobble
2. 第二次调用 `dispatchScrobble`，所有适配器再次收到 scrobble
3. 由于 `playTime` 不同（`time.Now()` 每次都不同），数据库不会去重
4. **结果**：会产生多条 scrobble 记录

**场景 2：同一播放事件被多个 API 端点触发**

如果同一播放事件同时触发了 `ReportPlayback` 和 `Submit`（Subsonic API）：

1. `ReportPlayback` 调用 `dispatchScrobble`
2. `Submit` 也调用 `dispatchScrobble`
3. 如果 `playTime` 相同，数据库会去重
4. 如果 `playTime` 不同，会产生多条记录

---

## 4. 队列层去重分析

### 4.1 入队逻辑

**核心代码**：`core/scrobbler/buffered_scrobbler.go:73-81`

```go
func (b *bufferedScrobbler) Scrobble(ctx context.Context, userId string, s Scrobble) error {
    err := b.ds.ScrobbleBuffer(ctx).Enqueue(b.service, userId, s.ID, s.TimeStamp)
    if err != nil {
        return err  // 直接返回错误，不做特殊处理
    }
    b.sendWakeSignal()
    return nil
}
```

### 4.2 数据库插入逻辑

**核心代码**：`persistence/scrobble_buffer_repository.go:53-64`

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
    _, err := r.executeSQL(ins)
    return err  // 违反唯一约束时返回原始数据库错误
}
```

### 4.3 错误处理链

```
dispatchScrobble()
    ↓
BufferedScrobbler.Scrobble()
    ↓
ScrobbleBufferRepository.Enqueue()
    ↓
sqlRepository.executeSQL()
    ↓
数据库执行 INSERT
    ↓
如果违反 UNIQUE 约束 → 返回 SQLite 错误（如 "UNIQUE constraint failed"）
    ↓
BufferedScrobbler.Scrobble() 直接返回错误
    ↓
dispatchScrobble() 记录错误日志，继续处理下一个适配器
```

### 4.4 去重效果

| 情况 | 结果 |
|------|------|
| user_id + service + media_file_id + play_time 都相同 | ✅ 数据库拒绝插入，返回错误 |
| 任何一个字段不同 | ✅ 成功插入 |

**重要**：数据库层面的去重是**被动的**，不是主动检查后跳过，而是直接尝试插入，利用数据库约束来拒绝重复。

---

## 5. 重试机制深度分析

### 5.1 重试调度器

**核心代码**：`core/scrobbler/buffered_scrobbler.go:99-113`

```go
func (b *bufferedScrobbler) run(ctx context.Context) {
    for {
        if !b.processQueue(ctx) {
            // 处理失败（遇到 ErrRetryLater），5秒后重试
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
- 固定 **5秒** 间隔重试
- 无限重试直到成功或遇到不可恢复错误
- 没有指数退避，没有最大重试次数限制

### 5.2 队列处理逻辑

**核心代码**：`core/scrobbler/buffered_scrobbler.go:131-168`

```go
func (b *bufferedScrobbler) processUserQueue(ctx context.Context, userId string) bool {
    buffer := b.ds.ScrobbleBuffer(ctx)
    for {
        entry, err := buffer.Next(b.service, userId)
        if err != nil {
            log.Error(ctx, "Error reading from scrobble buffer", "scrobbler", b.service, err)
            return false
        }
        if entry == nil {
            return true  // 队列空，处理成功
        }
        s, ok := b.loader()
        if !ok {
            log.Warn(ctx, "Scrobbler not available, will retry later", "scrobbler", b.service)
            return false  // 插件不可用，稍后重试
        }
        log.Debug(ctx, "Sending scrobble", "scrobbler", b.service, "track", entry.Title, "artist", entry.Artist)
        err = s.Scrobble(ctx, entry.UserID, Scrobble{
            MediaFile: entry.MediaFile,
            TimeStamp: entry.PlayTime,
        })
        if errors.Is(err, ErrRetryLater) {
            log.Warn(ctx, "Could not send scrobble. Will be retried", "userId", entry.UserID,
                "track", entry.Title, "artist", entry.Artist, "scrobbler", b.service, err)
            return false  // 可重试错误，保留在队列中，5秒后重试
        }
        if err != nil {
            log.Error(ctx, "Error sending scrobble to service. Discarding", "scrobbler", b.service,
                "userId", entry.UserID, "artist", entry.Artist, "track", entry.Title, err)
            // 不可恢复错误，继续执行 Dequeue 丢弃
        }
        err = buffer.Dequeue(entry)  // 成功或不可恢复错误都出队
        if err != nil {
            log.Error(ctx, "Error removing entry from scrobble buffer", "userId", entry.UserID,
                "track", entry.Title, "artist", entry.Artist, "scrobbler", b.service, err)
            return false
        }
    }
}
```

### 5.3 错误类型定义

**核心代码**：`core/scrobbler/interfaces.go:16-19`

```go
var (
    ErrNotAuthorized = errors.New("not authorized")   // 用户未授权，跳过（不入队）
    ErrRetryLater    = errors.New("retry later")      // 可重试错误，保留在队列中
    ErrUnrecoverable = errors.New("unrecoverable")    // 不可恢复错误，丢弃
)
```

---

## 6. 各适配器重试判定差异

### 6.1 Last.fm 适配器

**核心代码**：`adapters/lastfm/agent.go:379-412`

```go
func (l *lastfmAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)
    }

    if s.Duration <= 30 {
        log.Debug(ctx, "Skipping Last.fm scrobble for short song", "track", s.Title, "duration", s.Duration)
        return nil
    }
    err = l.client.scrobble(ctx, sk, ScrobbleInfo{...})
    if err == nil {
        return nil
    }
    var lfErr *lastFMError
    isLastFMError := errors.As(err, &lfErr)
    
    // 非 Last.fm 特定错误（如网络错误）→ 重试
    if !isLastFMError {
        log.Warn(ctx, "Last.fm client.scrobble returned error", "track", s.Title, err)
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // Last.fm 特定错误码判断
    // 11 = 服务暂时不可用, 16 = 临时错误
    if lfErr.Code == 11 || lfErr.Code == 16 {
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 其他 Last.fm 错误（如 6 = 验证失败, 7 = 无效会话等）→ 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**Last.fm 错误码参考**：

| 错误码 | 含义 | 处理方式 |
|--------|------|---------|
| 1 | 无效参数 | 丢弃 |
| 2 | 无效服务 | 丢弃 |
| 3 | 无效方法 | 丢弃 |
| 4 | 认证失败 | 丢弃 |
| 5 | 会话无效 | 丢弃 |
| 6 | 无效参数 | 丢弃 |
| 7 | 无效资源 | 丢弃 |
| 8 | 操作失败 | 丢弃 |
| 9 | 会话过期 | 丢弃 |
| 10 | 无效 API Key | 丢弃 |
| 11 | 服务暂时不可用 | ✅ 重试 |
| 12 | 订阅者限制 | 丢弃 |
| 13 | 签名无效 | 丢弃 |
| 14 | 令牌过期 | 丢弃 |
| 15 | 令牌未授权 | 丢弃 |
| 16 | 临时错误 | ✅ 重试 |
| 17 | 登录要求 | 丢弃 |
| 18 | 试用过期 | 丢弃 |
| 20 | 未授权 | 丢弃 |
| 21 | 未注册 | 丢弃 |
| 22 | API 挂起 | 丢弃 |
| 23 | 未授权 | 丢弃 |
| 24 | 已过期 | 丢弃 |
| 25 | 速率限制 | 丢弃 |

### 6.2 ListenBrainz 适配器

**核心代码**：`adapters/listenbrainz/agent.go:91-114`

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
    
    // 非 ListenBrainz 特定错误（如网络错误）→ 重试
    if !isListenBrainzError {
        log.Warn(ctx, "ListenBrainz Scrobble returned HTTP error", "track", s.Title, err)
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // HTTP 5xx 服务端错误 → 重试
    if lbErr.Code == 500 || lbErr.Code == 503 {
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    
    // 其他错误（如 4xx 客户端错误）→ 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

**ListenBrainz HTTP 状态码处理**：

| 状态码 | 含义 | 处理方式 |
|--------|------|---------|
| 200 | 成功 | 成功 |
| 400 | 无效请求 | 丢弃 |
| 401 | 未授权 | 丢弃 |
| 403 | 禁止访问 | 丢弃 |
| 404 | 未找到 | 丢弃 |
| 429 | 请求过多 | 丢弃 |
| 500 | 服务器内部错误 | ✅ 重试 |
| 503 | 服务不可用 | ✅ 重试 |

### 6.3 插件适配器

**核心代码**：`plugins/scrobbler_adapter.go:109-119, 169-186`

```go
func (s *ScrobblerPlugin) Scrobble(ctx context.Context, userId string, sc scrobbler.Scrobble) error {
    username := getUsernameFromContext(ctx)
    input := capabilities.ScrobbleRequest{
        Username:  username,
        Track:     mediaFileToTrackInfo(s.plugin, &sc.MediaFile),
        Timestamp: sc.TimeStamp.Unix(),
    }

    err := callPluginFunctionNoOutput(ctx, s.plugin, FuncScrobblerScrobble, input)
    return mapScrobblerError(err)
}

// 通过错误消息字符串匹配进行映射
func mapScrobblerError(err error) error {
    if err == nil {
        return nil
    }
    errMsg := err.Error()
    switch {
    case strings.Contains(errMsg, capabilities.ScrobblerErrorNotAuthorized.Error()):
        return scrobbler.ErrNotAuthorized
    case strings.Contains(errMsg, capabilities.ScrobblerErrorRetryLater.Error()):
        return scrobbler.ErrRetryLater
    case strings.Contains(errMsg, capabilities.ScrobblerErrorUnrecoverable.Error()):
        return scrobbler.ErrUnrecoverable
    default:
        return scrobbler.ErrUnrecoverable  // 默认不可恢复
    }
}
```

**插件错误约定**：

插件通过返回的错误消息字符串来指示处理方式：

| 错误消息包含 | 映射结果 |
|-------------|---------|
| `"not authorized"` | `ErrNotAuthorized` |
| `"retry later"` | `ErrRetryLater` |
| `"unrecoverable"` | `ErrUnrecoverable` |
| 其他任何错误 | `ErrUnrecoverable` |

---

## 7. 重试机制对比总结

| 维度 | Last.fm | ListenBrainz | 插件 |
|------|---------|-------------|------|
| **重试判定依据** | Last.fm 错误码 (11, 16) | HTTP 状态码 (500, 503) | 错误消息字符串匹配 |
| **网络错误** | ✅ 重试 | ✅ 重试 | ⚠️ 取决于插件实现 |
| **服务不可用** | ✅ 重试 | ✅ 重试 | ⚠️ 取决于插件实现 |
| **认证失败** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **无效参数** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **重试间隔** | 5秒（固定） | 5秒（固定） | 5秒（固定） |
| **最大重试次数** | 无限制 | 无限制 | 无限制 |
| **指数退避** | 无 | 无 | 无 |

---

## 8. 潜在问题与改进建议

### 8.1 现有问题

1. **分发层无去重**：如果同一首歌在不同时间被多次触发，会产生多条记录
2. **固定重试间隔**：5秒固定间隔可能导致在服务恢复时产生请求风暴
3. **无死信队列**：永久失败的记录会被直接丢弃，无法事后排查
4. **插件错误映射脆弱**：基于字符串匹配的错误映射容易出错
5. **无重试次数限制**：理论上可能无限重试同一条记录

### 8.2 改进建议

**建议 1：分发层增加时间窗口去重**

```go
// 在 PlayTracker 中增加最近 scrobble 缓存
type playTracker struct {
    // ...
    recentScrobbles cache.SimpleCache[string, time.Time]  // key: userId+service+mediaFileId
}

func (p *playTracker) dispatchScrobble(ctx context.Context, t *model.MediaFile, playTime time.Time) {
    // 检查 30 秒内是否已上报过
    key := fmt.Sprintf("%s:%s", u.ID, t.ID)
    if lastTime, ok := p.recentScrobbles.Get(key); ok {
        if playTime.Sub(lastTime) < 30*time.Second {
            log.Debug(ctx, "Skipping duplicate scrobble within time window", "track", t.Title)
            return
        }
    }
    p.recentScrobbles.AddWithTTL(key, playTime, 30*time.Second)
    // ... 继续分发
}
```

**建议 2：实现指数退避重试**

```go
func (b *bufferedScrobbler) run(ctx context.Context) {
    retryCount := 0
    for {
        if !b.processQueue(ctx) {
            retryCount++
            // 指数退避：5s, 10s, 20s, 40s, 最大 60s
            backoff := time.Duration(math.Min(5*math.Pow(2, float64(retryCount-1)), 60)) * time.Second
            time.AfterFunc(backoff, func() {
                b.sendWakeSignal()
            })
        } else {
            retryCount = 0  // 重置计数器
        }
        // ...
    }
}
```

**建议 3：增加死信队列**

```go
// 在 ScrobbleEntry 中增加重试次数字段
type ScrobbleEntry struct {
    ID          string
    Service     string
    UserID      string
    PlayTime    time.Time
    EnqueueTime time.Time
    MediaFileID string
    RetryCount  int  // 新增
    MediaFile
}

// 超过最大重试次数后移入死信队列
const MaxRetries = 10
if entry.RetryCount >= MaxRetries {
    log.Error(ctx, "Max retries exceeded, moving to dead letter queue", ...)
    b.ds.DeadLetterQueue(ctx).Enqueue(entry)
    buffer.Dequeue(entry)
    continue
}
```

**建议 4：插件错误使用类型化错误**

```go
// 在插件能力定义中使用类型化错误码
type ScrobblerError int32
const (
    ScrobblerErrorNone ScrobblerError = 0
    ScrobblerErrorNotAuthorized ScrobblerError = 1
    ScrobblerErrorRetryLater ScrobblerError = 2
    ScrobblerErrorUnrecoverable ScrobblerError = 3
)
```

---

## 9. 数据流完整图示

```
客户端播放事件 (StateStopped)
        │
        ▼
ReportPlayback() 被调用
        │
        ▼
检查播放进度 >= 阈值 (50% 或 4分钟)
        │
        ├─ 否 → 结束
        │
        └─ 是 → 本地事务更新播放计数
                  │
                  ▼
          dispatchScrobble()
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
Last.fm      ListenBrainz    插件 Scrobbler
    │             │             │
    └─────────────┼─────────────┘
                  │
                  ▼
      BufferedScrobbler.Scrobble()
                  │
                  ▼
      写入 scrobble_buffer 表
                  │
                  ├─ 违反 UNIQUE 约束 → 返回错误 → 记录日志 → 结束
                  │
                  └─ 成功 → 唤醒后台 Worker
                            │
                            ▼
                Worker 循环处理队列
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
        调用实际 Scrobbler        读取下一条记录
                │                       ▲
                ├─ 成功 → Dequeue →─────┘
                │
                ├─ ErrRetryLater → 保留在队列 → 5秒后重试
                │
                └─ ErrUnrecoverable → Dequeue (丢弃) → 下一条
```

---

## 10. 关键代码文件索引

| 文件 | 职责 |
|------|------|
| `core/scrobbler/play_tracker.go` | 播放会话管理、分发逻辑 |
| `core/scrobbler/buffered_scrobbler.go` | 缓冲队列、重试调度 |
| `core/scrobbler/interfaces.go` | Scrobbler 接口、错误类型定义 |
| `persistence/scrobble_buffer_repository.go` | 数据库队列操作 |
| `adapters/lastfm/agent.go` | Last.fm 适配器 |
| `adapters/listenbrainz/agent.go` | ListenBrainz 适配器 |
| `plugins/scrobbler_adapter.go` | 插件 Scrobbler 适配器 |
| `db/migrations/20210626213026_add_scrobble_buffer.go` | 初始表结构（含 bug） |
| `db/migrations/20260405124200_fix_schema_inconsistencies.sql` | 修复唯一约束 |
