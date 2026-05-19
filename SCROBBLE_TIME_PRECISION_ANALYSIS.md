# Navidrome 播放记录链路：时间精度与失败边界深度分析

## 1. 核心结论速览

| 问题 | 结论 |
|------|------|
| `play_time` 写库精度 | SQLite 以 `YYYY-MM-DD HH:MM:SS` 格式存储，**秒级精度**，小数秒被截断 |
| 时区处理 | 以本地时区字符串存储，不保存时区信息 |
| 唯一约束去重窗口 | **同一秒内**的重复触发会去重；不同秒数不会去重 |
| Enqueue 失败后是否重试 | ❌ 不重试，只记录日志后结束，可能丢失数据 |
| Worker 阶段 ErrRetryLater | ✅ 5秒后重试，无限重试直到成功或不可恢复错误，数据已持久化 |

---

## 2. play_time 写库格式深度追踪

### 2.1 完整链路图示

```
Go 程序层                  ORM/驱动层                SQLite 存储层
─────────────────────────────────────────────────────────────────────────
time.Now()
    │
    ▼  纳秒精度 (e.g., 2026-05-19 14:30:25.123456789 +0800 CST)
time.Time
    │
    ▼  dbx + mattn/go-sqlite3 参数绑定
    │  自动转换为 SQLite TEXT 格式
    ▼
'2026-05-19 14:30:25'
    │
    ▼  写入 scrobble_buffer.play_time 列
YYYY-MM-DD HH:MM:SS
    └─────────────────────────────────────────────────┐
                                                      │
                                           ✅ 实际存储格式：秒级精度
                                           ❌ 小数秒被截断
                                           ❌ 时区信息不保存
```

### 2.2 代码证据链

#### 证据 1：迁移文件中的时间格式定义

**文件**：`db/migrations/20241026183640_support_new_scanner.go:121-123`

```sql
images_updated_at datetime default '0000-00-00 00:00:00' not null,
updated_at datetime default (datetime(current_timestamp, 'localtime')) not null,
created_at datetime default (datetime(current_timestamp, 'localtime')) not null
```

**分析**：
- `datetime(current_timestamp, 'localtime')` 是 SQLite 内置函数
- 返回格式固定为 `YYYY-MM-DD HH:MM:SS`
- **秒级精度**，没有小数秒部分
- 使用 `'localtime'` 修饰符转换为本地时区，但**不存储时区信息**

#### 证据 2：scrobble_buffer 表定义

**文件**：`db/migrations/20210626213026_add_scrobble_buffer.go:27-28`

```sql
play_time datetime not null,
enqueue_time datetime not null default current_timestamp,
```

**分析**：
- `play_time` 列类型为 `datetime`
- `enqueue_time` 使用 `current_timestamp` 作为默认值
- SQLite 的 `current_timestamp` 返回 `YYYY-MM-DD HH:MM:SS` 格式

#### 证据 3：dbx 库参数绑定

**文件**：`persistence/scrobble_buffer_repository.go:53-64`

```go
func (r *scrobbleBufferRepository) Enqueue(service, userId, mediaFileId string, playTime time.Time) error {
    ins := Insert(r.tableName).SetMap(map[string]any{
        "id":            id.NewRandom(),
        "user_id":       userId,
        "service":       service,
        "media_file_id": mediaFileId,
        "play_time":     playTime,  // 直接传入 time.Time 类型
        "enqueue_time":  time.Now(),
    })
    _, err := r.executeSQL(ins)
    return err
}
```

**分析**：
- 直接传入 Go 的 `time.Time` 类型给 dbx 库
- 由 `github.com/pocketbase/dbx` + `github.com/mattn/go-sqlite3` 驱动负责参数绑定
- SQLite 驱动会将 `time.Time` 格式化为 `YYYY-MM-DD HH:MM:SS` 字符串

#### 证据 4：测试中的时间断言

**文件**：`persistence/scrobble_buffer_repository_test.go:155-157`

```go
Expect(entry.EnqueueTime).To(BeTemporally("~", now, 100*time.Millisecond))
Expect(entry.MediaFileID).To(Equal(fileId))
Expect(entry.PlayTime).To(BeTemporally("==", playTime))
```

**分析**：
- `EnqueueTime` 使用 `"~"` 模糊匹配（允许 100ms 误差），因为它由数据库 `current_timestamp` 生成
- `PlayTime` 使用 `"=="` 精确匹配，因为它由 Go 代码传入并完整保留
- 测试用例中的 `playTime` 都是整点时间（`time.Date(2025, 01, 01, 00, 00, 00, 00, time.Local)`），没有小数秒，所以能精确匹配

### 2.3 时间精度实验

**假设场景**：

| Go time.Now() 实际值 | SQLite 存储值 | 精度变化 |
|---------------------|--------------|---------|
| `2026-05-19 14:30:25.123456789` | `'2026-05-19 14:30:25'` | 纳秒 → 秒，小数秒丢失 |
| `2026-05-19 14:30:25.999999999` | `'2026-05-19 14:30:25'` | 同秒内，相同存储值 |
| `2026-05-19 14:30:26.000000001` | `'2026-05-19 14:30:26'` | 跨秒，不同存储值 |

**结论**：
- ✅ 实际存储精度：**秒级**
- ✅ 小数秒：被截断，不存储
- ✅ 时区：以本地时区字符串存储，不保存时区信息
- ✅ 去重窗口：**同一秒内**的时间值在数据库中相同

---

## 3. 唯一约束去重边界重算

### 3.1 唯一键构成

**文件**：`db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50`

```sql
CONSTRAINT scrobble_buffer_pk UNIQUE (user_id, service, media_file_id, play_time)
```

### 3.2 去重判定公式

```
是否去重 = (user_id 相同) AND (service 相同) AND (media_file_id 相同) AND (play_time 存储值相同)
```

由于 `play_time` 存储值是**秒级精度**：
- 同一秒内的两个不同时间 → 存储值相同 → **去重**
- 不同秒的两个时间 → 存储值不同 → **不去重**

### 3.3 场景详细分析

#### 场景 1：同一秒内重复触发（去重成功）

```
时间轴: 14:30:25.100 ─────────────────────────────── 14:30:25.900
            │                                            │
            ▼                                            ▼
第 1 次 ReportPlayback                         第 2 次 ReportPlayback
  now = 14:30:25.100                            now = 14:30:25.900
  play_time 存储: '2026-05-19 14:30:25'         play_time 存储: '2026-05-19 14:30:25'
  ✅ 插入成功                                   ❌ UNIQUE constraint failed
                                                → 记录错误日志，不重试

结果：✅ 去重成功，只有第 1 条记录被保留
```

#### 场景 2：跨秒边界触发（去重失败）

```
时间轴: 14:30:25.900 ────────────────┼─────────────── 14:30:26.100
            │                        │                       │
            ▼                        ▼                       ▼
第 1 次触发                     秒边界                    第 2 次触发
  now = 14:30:25.900                                      now = 14:30:26.100
  play_time 存储: '2026-05-19 14:30:25'                   play_time 存储: '2026-05-19 14:30:26'
  ✅ 插入成功（第 1 条）                                   ✅ 插入成功（第 2 条）

结果：❌ 不去重，产生 2 条记录，都会上报到外部服务
```

#### 场景 3：不同服务的同一播放时间

```
dispatchScrobble(now = 14:30:25.500)
    │
    ├─ Last.fm 适配器
    │   service = "lastfm"
    │   play_time 存储: '2026-05-19 14:30:25'
    │   → ✅ 插入成功
    │
    └─ ListenBrainz 适配器
        service = "listenbrainz"
        play_time 存储: '2026-05-19 14:30:25'
        → ✅ 插入成功（service 不同，不触发唯一约束）

结果：✅ 两个服务各有一条记录，互不影响
```

### 3.4 去重边界总结表

| 场景 | user_id | service | media_file_id | play_time (存储值) | 是否去重 |
|------|---------|---------|--------------|-------------------|---------|
| 同一秒内重复触发 | 相同 | 相同 | 相同 | 相同 | ✅ 去重 |
| 跨秒边界触发 | 相同 | 相同 | 相同 | 不同 | ❌ 不去重 |
| 间隔 2 秒触发 | 相同 | 相同 | 相同 | 不同 | ❌ 不去重 |
| 不同用户 | 不同 | 相同 | 相同 | 相同 | ❌ 不去重 |
| 不同服务 | 相同 | 不同 | 相同 | 相同 | ❌ 不去重 |
| 不同歌曲 | 相同 | 相同 | 不同 | 相同 | ❌ 不去重 |

---

## 4. 失败路径边界完整分析

### 4.1 两条失败路径全景图

```
ReportPlayback(StateStopped)
        │
        ▼
dispatchScrobble()
        │
        ├─────────────────────────────────────────────────────────┐
        │                                                         │
        ▼                                                         ▼
┌──────────────────────────────┐                   ┌──────────────────────────────┐
│    路径 1：Enqueue 阶段      │                   │    路径 2：Worker 阶段       │
│    （入队前失败）            │                   │    （入队后失败）            │
└──────────────────────────────┘                   └──────────────────────────────┘
        │                                                         │
        ├─ 原因：                                                ├─ 原因：
        │  • 唯一约束冲突（同一秒重复）                           │  • 网络错误
        │  • 数据库连接错误                                      │  • 外部服务不可用（503/500）
        │  • 外键约束失败                                        │  • 认证失败（401/403）
        │  • 磁盘满/权限问题                                     │  • 无效参数（400）
        │                                                         │
        ├─ 处理：                                                ├─ 处理：
        │  • ❌ 不入库                                           │  • ✅ 已入库（持久化）
        │  • ❌ 不重试                                           │  • ErrRetryLater → 5秒后重试
        │  • ✅ 记录错误日志                                     │  • 其他错误 → 出队丢弃
        │  • ✅ 不阻塞其他适配器                                 │  • ✅ 不阻塞其他适配器
        │                                                         │
        └─ 可靠性：⚠️ 可能丢失数据                                └─ 可靠性：✅ 数据不丢失（已持久化）
```

### 4.2 路径 1：Enqueue 阶段失败详解

**触发位置**：`core/scrobbler/buffered_scrobbler.go:73-81`

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

**上层处理**：`core/scrobbler/play_tracker.go:471-474`

```go
err := s.Scrobble(ctx, u.ID, scrobble)
if err != nil {
    // ⚠️ 只记录错误日志，然后 continue 处理下一个适配器
    log.Error(ctx, "Error sending Scrobble", "scrobbler", name, ..., err)
    continue
}
```

**失败原因与处理**：

| 失败原因 | 是否入队 | 是否重试 | 数据可靠性 |
|---------|---------|---------|-----------|
| 唯一约束冲突（同一秒重复） | ❌ | ❌ | ✅ 预期行为，不丢失 |
| 数据库连接临时故障 | ❌ | ❌ | ⚠️ 可能丢失 |
| 外键约束失败（歌曲不存在） | ❌ | ❌ | ⚠️ 可能丢失 |
| 磁盘满/权限问题 | ❌ | ❌ | ⚠️ 可能丢失 |

### 4.3 路径 2：Worker 阶段失败详解

**触发位置**：`core/scrobbler/buffered_scrobbler.go:131-168`

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
            return false  // ⚠️ Dequeue 失败也触发重试（条目未被删除）
        }
    }
}
```

**重试调度**：`core/scrobbler/buffered_scrobbler.go:99-113`

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

**失败原因与处理**：

| 失败原因 | 错误类型 | 是否保留队列 | 是否重试 | 数据可靠性 |
|---------|---------|-------------|---------|-----------|
| 网络错误/超时 | `ErrRetryLater` | ✅ | ✅ 5秒后 | ✅ 安全 |
| 外部服务 503 | `ErrRetryLater` | ✅ | ✅ 5秒后 | ✅ 安全 |
| 外部服务 500 | `ErrRetryLater` | ✅ | ✅ 5秒后 | ✅ 安全 |
| 插件不可用 | N/A | ✅ | ✅ 5秒后 | ✅ 安全 |
| 读取队列失败 | N/A | ✅ | ✅ 5秒后 | ✅ 安全 |
| Dequeue 失败 | N/A | ✅ | ✅ 5秒后 | ✅ 安全 |
| 认证失败 401 | `ErrUnrecoverable` | ❌ | ❌ | ⚠️ 丢弃 |
| 无效参数 400 | `ErrUnrecoverable` | ❌ | ❌ | ⚠️ 丢弃 |
| 禁止访问 403 | `ErrUnrecoverable` | ❌ | ❌ | ⚠️ 丢弃 |
| 请求过多 429 | `ErrUnrecoverable` | ❌ | ❌ | ⚠️ 丢弃 |

### 4.4 两条路径可靠性对比

| 维度 | Enqueue 阶段失败 | Worker 阶段失败（ErrRetryLater） | Worker 阶段失败（其他错误） |
|------|-----------------|--------------------------------|---------------------------|
| **触发位置** | 分发→入队时 | Worker 调用外部 API 时 | Worker 调用外部 API 时 |
| **是否持久化** | ❌ 未持久化 | ✅ 已持久化到数据库 | ✅ 已持久化，然后出队 |
| **重试机制** | ❌ 不重试 | ✅ 5秒后重试（无限） | ❌ 不重试，丢弃 |
| **数据丢失风险** | ⚠️ 高（如临时数据库故障） | ✅ 无（已持久化） | ⚠️ 高（永久丢失） |
| **数据恢复可能性** | ❌ 无法恢复 | ✅ 可恢复（服务恢复后自动重试） | ❌ 无法恢复 |
| **典型场景** | 唯一约束冲突、数据库连接错误 | 网络波动、外部服务重启 | 认证失效、参数错误 |
| **日志级别** | Error | Warn | Error |

---

## 5. 各适配器重试判定差异

### 5.1 Last.fm 适配器

**文件**：`adapters/lastfm/agent.go:379-412`

```go
func (l *lastfmAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)  // → 不入队
    }

    if s.Duration <= 30 {
        return nil  // → 成功但不上报
    }
    
    err = l.client.scrobble(ctx, sk, ScrobbleInfo{...})
    if err == nil {
        return nil
    }
    
    var lfErr *lastFMError
    isLastFMError := errors.As(err, &lfErr)
    
    if !isLastFMError {
        // 网络错误等 → 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    if lfErr.Code == 11 || lfErr.Code == 16 {
        // 11=服务不可用, 16=临时错误 → 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    // 其他错误 → 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

### 5.2 ListenBrainz 适配器

**文件**：`adapters/listenbrainz/agent.go:91-114`

```go
func (l *listenBrainzAgent) Scrobble(ctx context.Context, userId string, s scrobbler.Scrobble) error {
    sk, err := l.sessionKeys.Get(ctx, userId)
    if err != nil || sk == "" {
        return errors.Join(err, scrobbler.ErrNotAuthorized)  // → 不入队
    }

    li := l.formatListen(&s.MediaFile)
    li.ListenedAt = int(s.TimeStamp.Unix())
    err = l.client.scrobble(ctx, sk, li)

    if err == nil {
        return nil
    }
    
    var lbErr *listenBrainzError
    isListenBrainzError := errors.As(err, &lbErr)
    
    if !isListenBrainzError {
        // 网络错误等 → 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    if lbErr.Code == 500 || lbErr.Code == 503 {
        // 5xx 服务端错误 → 重试
        return errors.Join(err, scrobbler.ErrRetryLater)
    }
    // 其他错误 → 丢弃
    return errors.Join(err, scrobbler.ErrUnrecoverable)
}
```

### 5.3 插件适配器

**错误字面值定义**：`plugins/capabilities/scrobbler.go:123-136`

```go
type ScrobblerError string

const (
    ScrobblerErrorNotAuthorized ScrobblerError = "scrobbler(not_authorized)"
    ScrobblerErrorRetryLater    ScrobblerError = "scrobbler(retry_later)"
    ScrobblerErrorUnrecoverable ScrobblerError = "scrobbler(unrecoverable)"
)
```

**错误映射逻辑**：`plugins/scrobbler_adapter.go:169-186`

```go
func mapScrobblerError(err error) error {
    if err == nil {
        return nil
    }
    errMsg := err.Error()
    switch {
    // ⚠️ 使用 strings.Contains，不是精确匹配！
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

### 5.4 三适配器对比总结

| 维度 | Last.fm | ListenBrainz | 插件 |
|------|---------|-------------|------|
| **判定依据** | Last.fm 错误码 (11, 16) | HTTP 状态码 (500, 503) | 错误消息字符串包含匹配 |
| **网络错误** | ✅ 重试 | ✅ 重试 | ⚠️ 取决于插件是否返回正确字面值 |
| **服务不可用** | ✅ 重试（码11） | ✅ 重试（码503） | ⚠️ 取决于插件实现 |
| **认证失败** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **无效参数** | ❌ 丢弃 | ❌ 丢弃 | ⚠️ 取决于插件实现 |
| **匹配方式** | 精确错误码匹配 | 精确状态码匹配 | `strings.Contains` 包含匹配 |
| **默认行为** | 未明确错误码 → 丢弃 | 未明确状态码 → 丢弃 | 未明确字面值 → 丢弃 |

---

## 6. 关键代码索引

| 文件 | 说明 |
|------|------|
| `core/scrobbler/play_tracker.go:267` | `now = time.Now()` 时间产生点 |
| `core/scrobbler/play_tracker.go:457-477` | `dispatchScrobble` 分发逻辑（串行） |
| `core/scrobbler/buffered_scrobbler.go:73-81` | `Scrobble` 入队逻辑（路径 1 失败点） |
| `core/scrobbler/buffered_scrobbler.go:99-113` | 重试调度器 `run`（5秒固定间隔） |
| `core/scrobbler/buffered_scrobbler.go:131-168` | `processUserQueue` 错误处理（路径 2 失败点） |
| `persistence/scrobble_buffer_repository.go:53-64` | `Enqueue` 数据库插入 |
| `adapters/lastfm/agent.go:379-412` | Last.fm 重试判定 |
| `adapters/listenbrainz/agent.go:91-114` | ListenBrainz 重试判定 |
| `plugins/capabilities/scrobbler.go:123-136` | 插件错误字面值定义 |
| `plugins/scrobbler_adapter.go:169-186` | 插件错误映射 `mapScrobblerError` |
| `db/migrations/20210626213026_add_scrobble_buffer.go:27-28` | 初始表结构 |
| `db/migrations/20260405124200_fix_schema_inconsistencies.sql:27-50` | 唯一约束定义 |
| `db/migrations/20241026183640_support_new_scanner.go:121-123` | 时间格式证据 |
