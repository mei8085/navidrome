# Navidrome 播放记录(Scrobble)提交链路分析报告

## 1. 系统架构概览

Navidrome 的播放记录系统采用**分层架构**，从播放进度检测到多服务并发上报，整个链路经过精心设计以确保可靠性和可扩展性。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              客户端播放事件                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PlayTracker (播放会话追踪器)                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────────┐  │
│  │ 进度检测    │→│ 触发条件判断 │→│ 本地播放计数 │→│ 分发到外部适配器   │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  BufferedScrobbler   │  │  BufferedScrobbler   │  │  BufferedScrobbler   │
│  (Last.fm 适配器)    │  │  (ListenBrainz)      │  │  (插件 Scrobbler)    │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
              │                       │                       │
              ▼                       ▼                       ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   ScrobbleBuffer     │  │   ScrobbleBuffer     │  │   ScrobbleBuffer     │
│   (数据库队列)       │  │   (数据库队列)       │  │   (数据库队列)       │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
              │                       │                       │
              ▼                       ▼                       ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   Last.fm API        │  │  ListenBrainz API    │  │  自定义插件 API      │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

---

## 2. 播放记录触发条件与进度检测

### 2.1 触发阈值逻辑

**核心代码位置**：`core/scrobbler/play_tracker.go:332-340`

```go
trackDurationMs := int64(mf.Duration * 1000)
threshold := min(trackDurationMs*50/100, 240_000)  // 50% 或 4 分钟，取较小值
if params.PositionMs >= threshold {
    err = p.incPlay(ctx, mf, now)
    p.dispatchScrobble(ctx, mf, now)
}
```

**触发条件**：
- 播放进度达到歌曲总时长的 **50%**
- 或者播放时长达到 **240 秒（4分钟）**
- 两者取较小值作为触发阈值

**触发时机**：
- 当播放器状态变为 `StateStopped`（停止）时检测
- 必须满足 `player.ScrobbleEnabled = true`
- 必须满足 `params.IgnoreScrobble = false`

### 2.2 播放会话状态管理

**状态定义**：
- `StateStarting` - 开始播放
- `StatePlaying` - 播放中
- `StatePaused` - 暂停
- `StateStopped` - 停止
- `StateExpired` - 会话过期（缓存超时）

**会话缓存 TTL 策略**：
- `StatePlaying` 状态：基于剩余播放时间计算（`remainingTTL` 函数）
- `StatePaused` 状态：固定 30 分钟
- 过期回调：自动触发 `PlaybackReport` 上报

---

## 3. 本地播放计数与历史记录

### 3.1 本地计数更新

**核心代码位置**：`core/scrobbler/play_tracker.go:434-455`

```go
func (p *playTracker) incPlay(ctx context.Context, track *model.MediaFile, timestamp time.Time) error {
    return p.ds.WithTx(func(tx model.DataStore) error {
        tx.MediaFile(ctx).IncPlayCount(track.ID, timestamp)    // 歌曲播放计数
        tx.Album(ctx).IncPlayCount(track.AlbumID, timestamp)   // 专辑播放计数
        for _, artist := range track.Participants[model.RoleArtist] {
            tx.Artist(ctx).IncPlayCount(artist.ID, timestamp)  // 艺术家播放计数
        }
        if conf.Server.EnableScrobbleHistory {
            tx.Scrobble(ctx).RecordScrobble(track.ID, timestamp)  // 本地历史记录
        }
        return nil
    })
}
```

**事务特性**：
- 所有计数更新在同一个数据库事务中执行
- 确保原子性：要么全部更新成功，要么全部失败

### 3.2 本地历史记录

通过 `EnableScrobbleHistory` 配置项控制是否在本地数据库中保存 scrobble 历史，表名为 `scrobble`。

---

## 4. 外部集成适配器架构

### 4.1 适配器注册机制

**核心代码位置**：`core/scrobbler/play_tracker.go:151-162`

```go
var constructors map[string]Constructor

func Register(name string, init Constructor) {
    if constructors == nil {
        constructors = make(map[string]Constructor)
    }
    constructors[name] = init
}
```

**内置适配器**：
1. **Last.fm** - `adapters/lastfm/agent.go`
2. **ListenBrainz** - `adapters/listenbrainz/agent.go`
3. **插件 Scrobbler** - `plugins/scrobbler_adapter.go`（动态加载）

### 4.2 Scrobbler 接口定义

**核心代码位置**：`core/scrobbler/interfaces.go:22-27`

```go
type Scrobbler interface {
    IsAuthorized(ctx context.Context, userId string) bool
    NowPlaying(ctx context.Context, userId string, track *model.MediaFile, position int) error
    Scrobble(ctx context.Context, userId string, s Scrobble) error
    PlaybackReport(ctx context.Context, info PlaybackSession) error
}
```

---

## 5. 凭据来源与授权机制

### 5.1 服务端配置凭据

| 服务 | 配置项 | 存储位置 |
|------|--------|----------|
| Last.fm | `LastFM.ApiKey`, `LastFM.Secret` | 配置文件/环境变量 |
| ListenBrainz | 无（用户级 Token） | 无服务端凭据 |
| 插件 | 自定义配置 | 插件配置系统 |

**Last.fm 配置结构**：`conf/configuration.go:187-196`
```go
type lastfmOptions struct {
    Enabled                 bool
    ApiKey                  string  // 应用级 API Key
    Secret                  string  // 应用级 Secret
    Language                string
    ScrobbleFirstArtistOnly bool
    Languages               []string
}
```

### 5.2 用户级会话密钥

**存储机制**：`core/agents/session_keys.go`

```go
type SessionKeys struct {
    model.DataStore
    KeyName string  // e.g., "LastFMSessionKey", "ListenBrainzSessionKey"
}

func (sk *SessionKeys) Get(ctx context.Context, userId string) (string, error) {
    return sk.DataStore.UserProps(ctx).Get(userId, sk.KeyName)
}
```

**存储位置**：`user_props` 数据库表，按用户存储

**授权检查**：
```go
func (l *lastfmAgent) IsAuthorized(ctx context.Context, userId string) bool {
    sk, err := l.sessionKeys.Get(ctx, userId)
    return err == nil && sk != ""
}
```

### 5.3 插件级授权

**核心代码位置**：`plugins/scrobbler_adapter.go:63-81`

插件 Scrobbler 采用**双重授权检查**：
1. **服务端授权**：检查用户是否在插件的允许列表中
2. **插件授权**：调用插件的 `nd_scrobbler_is_authorized` 函数

---

## 6. 多服务并发提交与去重策略

### 6.1 分发机制

**核心代码位置**：`core/scrobbler/play_tracker.go:457-477`

```go
func (p *playTracker) dispatchScrobble(ctx context.Context, t *model.MediaFile, playTime time.Time) {
    allScrobblers := p.getActiveScrobblers()  // 获取所有已启用的适配器
    u, _ := request.UserFrom(ctx)
    scrobble := Scrobble{MediaFile: *t, TimeStamp: playTime}
    
    for name, s := range allScrobblers {
        if !s.IsAuthorized(ctx, u.ID) {  // 跳过未授权用户
            continue
        }
        err := s.Scrobble(ctx, u.ID, scrobble)  // 并发发送到各适配器
        // ...
    }
}
```

**并发特性**：
- 遍历所有已启用的适配器
- 每个适配器独立处理
- 单个适配器失败不影响其他适配器

### 6.2 去重策略

**数据库级去重**：`persistence/scrobble_buffer_repository.go:53-64`

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
    return err
}
```

**去重机制分析**：
- **按服务隔离**：每个服务（`service` 字段）有独立的队列
- **按用户隔离**：每个用户（`user_id` 字段）有独立的队列
- **FIFO 顺序**：按 `play_time` 和 `rowid` 排序保证播放顺序
- **无内置去重**：数据库层面没有唯一键约束，依赖调用方保证不重复入队

> **注意**：当前实现没有严格的去重机制。如果同一首歌被多次触发 scrobble，可能会重复入队。这是设计上的权衡，因为：
> 1. 播放触发条件（50%/4分钟）已经减少了重复概率
> 2. 外部服务（Last.fm/ListenBrainz）通常有自己的去重逻辑
> 3. 简化了实现复杂度

### 6.3 插件动态刷新

**核心代码位置**：`core/scrobbler/play_tracker.go:188-238`

```go
func (p *playTracker) refreshPluginScrobblers() {
    pluginNames := p.pluginLoader.PluginNames("Scrobbler")
    if pluginNamesMatchScrobblers(pluginNames, p.pluginScrobblers) {
        return  // 无变化时快速返回
    }
    // 添加新插件 / 移除已卸载插件
}
```

每次分发前都会检查插件列表，确保动态加载的插件能及时参与 scrobble 分发。

---

## 7. 缓冲队列与重试机制

### 7.1 BufferedScrobbler 包装器

**核心代码位置**：`core/scrobbler/buffered_scrobbler.go`

```go
type bufferedScrobbler struct {
    ds         model.DataStore
    loader     Loader        // 动态加载实际的 Scrobbler
    service    string        // 服务名称
    wakeSignal chan struct{} // 唤醒信号
    ctx        context.Context
    cancel     context.CancelFunc
}
```

**设计模式**：装饰器模式，为每个 Scrobbler 添加缓冲能力

### 7.2 入队流程

```go
func (b *bufferedScrobbler) Scrobble(ctx context.Context, userId string, s Scrobble) error {
    err := b.ds.ScrobbleBuffer(ctx).Enqueue(b.service, userId, s.ID, s.TimeStamp)
    if err != nil {
        return err
    }
    b.sendWakeSignal()  // 唤醒后台 worker
    return nil
}
```

**持久化保证**：Scrobble 先写入数据库 `scrobble_buffer` 表，再通知 worker 处理，确保即使服务重启也不会丢失。

### 7.3 后台 Worker 处理

**核心代码位置**：`core/scrobbler/buffered_scrobbler.go:99-113`

```go
func (b *bufferedScrobbler) run(ctx context.Context) {
    for {
        if !b.processQueue(ctx) {
            time.AfterFunc(5*time.Second, func() {  // 失败后 5 秒重试
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

### 7.4 错误分类与重试策略

**错误类型定义**：`core/scrobbler/interfaces.go:16-19`

```go
var (
    ErrNotAuthorized = errors.New("not authorized")   // 用户未授权，跳过
    ErrRetryLater    = errors.New("retry later")      // 可重试错误，保留在队列中
    ErrUnrecoverable = errors.New("unrecoverable")    // 不可恢复错误，丢弃
)
```

**处理逻辑**：`core/scrobbler/buffered_scrobbler.go:131-168`

```go
func (b *bufferedScrobbler) processUserQueue(ctx context.Context, userId string) bool {
    for {
        entry, err := buffer.Next(b.service, userId)
        // ...
        err = s.Scrobble(ctx, entry.UserID, Scrobble{...})
        
        if errors.Is(err, ErrRetryLater) {
            log.Warn("Could not send scrobble. Will be retried", ...)
            return false  // 返回 false，触发 5 秒后重试
        }
        if err != nil {
            log.Error("Error sending scrobble to service. Discarding", ...)
            // 不可恢复错误，继续执行 Dequeue 丢弃
        }
        err = buffer.Dequeue(entry)  // 成功或不可恢复错误都出队
    }
}
```

**重试策略总结**：

| 错误类型 | 处理方式 | 重试行为 |
|---------|---------|---------|
| `ErrRetryLater` | 保留在队列中 | 5 秒后自动重试，无限重试直到成功 |
| `ErrUnrecoverable` | 从队列中删除 | 不重试，永久丢弃 |
| `ErrNotAuthorized` | 跳过（不入队） | 不重试 |
| 其他错误 | 从队列中删除 | 不重试 |

### 7.5 各适配器的错误映射

**Last.fm 错误映射**：`adapters/lastfm/agent.go:399-412`

```go
var lfErr *lastFMError
isLastFMError := errors.As(err, &lfErr)
if !isLastFMError {
    return errors.Join(err, scrobbler.ErrRetryLater)  // 网络错误等，重试
}
if lfErr.Code == 11 || lfErr.Code == 16 {
    return errors.Join(err, scrobbler.ErrRetryLater)  // 服务繁忙/临时错误，重试
}
return errors.Join(err, scrobbler.ErrUnrecoverable)    // 其他错误，丢弃
```

**ListenBrainz 错误映射**：`adapters/listenbrainz/agent.go:101-114`

```go
var lbErr *listenBrainzError
isListenBrainzError := errors.As(err, &lbErr)
if !isListenBrainzError {
    return errors.Join(err, scrobbler.ErrRetryLater)  // 网络错误等，重试
}
if lbErr.Code == 500 || lbErr.Code == 503 {
    return errors.Join(err, scrobbler.ErrRetryLater)  // 服务端错误，重试
}
return errors.Join(err, scrobbler.ErrUnrecoverable)    // 其他错误，丢弃
```

**插件错误映射**：`plugins/scrobbler_adapter.go:169-186`

通过错误消息字符串匹配进行映射：
- `"not authorized"` → `ErrNotAuthorized`
- `"retry later"` → `ErrRetryLater`
- `"unrecoverable"` → `ErrUnrecoverable`

---

## 8. NowPlaying 实时上报

### 8.1 上报触发

**核心代码位置**：`core/scrobbler/play_tracker.go:374-379`

```go
if !params.IgnoreScrobble && player.ScrobbleEnabled &&
    (params.State == StateStarting || params.State == StatePlaying) {
    if info, err := p.playMap.Get(clientId); err == nil {
        p.enqueueNowPlaying(ctx, clientId, user.ID, &info.MediaFile, int(params.PositionMs/1000))
    }
}
```

### 8.2 批量合并机制

**核心代码位置**：`core/scrobbler/nowplaying_worker.go:33-58`

```go
func (p *playTracker) nowPlayingWorker() {
    for {
        select {
        case <-time.After(time.Second):  // 每秒合并一次
        case <-p.npSignal:
        }
        p.npMu.Lock()
        entries := p.npQueue
        p.npQueue = make(map[string]nowPlayingEntry)  // 按 clientId 去重，只保留最新
        p.npMu.Unlock()
        
        for _, entry := range entries {
            p.dispatchNowPlaying(entry.ctx, entry.userId, entry.track, entry.position)
        }
    }
}
```

**去重特性**：
- 使用 `map[string]nowPlayingEntry` 按 `clientId` 存储
- 同一客户端的多次上报会被合并，只保留最后一次
- 每秒批量处理一次，减少 API 调用频率

---

## 9. PlaybackReport 播放状态上报

### 9.1 上报时机

- `StateStarting` - 播放开始时
- `StatePlaying` - 播放进度更新时
- `StatePaused` - 暂停时
- `StateStopped` - 停止时
- `StateExpired` - 会话过期时（缓存自动清理触发）

### 9.2 后台处理

**核心代码位置**：`core/scrobbler/playbackreport_worker.go:27-49`

与 NowPlaying 类似，使用独立的 worker 和队列，支持批量合并处理。

---

## 10. 数据流总结

### 10.1 完整 Scrobble 提交流程

```
1. 客户端调用 ReportPlayback(StateStopped)
   ↓
2. 检测播放进度是否达到阈值（50% 或 4分钟）
   ↓
3. 本地事务更新播放计数（歌曲/专辑/艺术家）
   ↓
4. dispatchScrobble() 遍历所有 Scrobbler
   ├─ 检查用户是否授权（IsAuthorized）
   └─ 调用 BufferedScrobbler.Scrobble()
      ↓
5. 写入 scrobble_buffer 数据库表
   ↓
6. 唤醒后台 worker
   ↓
7. worker 从数据库按顺序取出条目
   ├─ 调用实际 Scrobbler 的 Scrobble() 方法
   ├─ 成功：Dequeue 删除条目
   ├─ ErrRetryLater：保留条目，5秒后重试
   └─ ErrUnrecoverable：Dequeue 删除条目（丢弃）
```

### 10.2 关键组件交互表

| 组件 | 职责 | 关键文件 |
|------|------|---------|
| `PlayTracker` | 播放会话管理、触发检测、分发 | `core/scrobbler/play_tracker.go` |
| `BufferedScrobbler` | 队列缓冲、重试调度 | `core/scrobbler/buffered_scrobbler.go` |
| `LastFMAgent` | Last.fm API 适配 | `adapters/lastfm/agent.go` |
| `ListenBrainzAgent` | ListenBrainz API 适配 | `adapters/listenbrainz/agent.go` |
| `ScrobblerPlugin` | 插件 Scrobbler 适配 | `plugins/scrobbler_adapter.go` |
| `ScrobbleBufferRepository` | 队列数据库操作 | `persistence/scrobble_buffer_repository.go` |
| `SessionKeys` | 用户会话密钥管理 | `core/agents/session_keys.go` |

---

## 11. 设计特点与权衡

### 11.1 优点

1. **高可靠性**：Scrobble 先持久化到数据库，再异步发送
2. **可扩展性**：通过注册机制支持多种适配器，包括动态插件
3. **错误隔离**：单个适配器失败不影响其他适配器和本地计数
4. **智能重试**：区分可重试和不可恢复错误，避免无限重试无效请求
5. **流量控制**：NowPlaying 每秒合并上报，减少 API 调用频率

### 11.2 潜在改进点

1. **去重机制**：当前依赖外部服务去重，可考虑在 `scrobble_buffer` 表增加唯一索引
2. **重试退避**：当前固定 5 秒重试，可考虑指数退避策略
3. **死信队列**：多次重试失败的条目可移入死信队列，避免永久阻塞
4. **批量上报**：Last.fm API 支持批量 scrobble，当前是逐条发送

---

## 12. 配置参考

### 12.1 Last.fm 配置

```toml
[LastFM]
Enabled = true
ApiKey = "your_api_key"
Secret = "your_secret"
Language = "en"
ScrobbleFirstArtistOnly = false
```

### 12.2 ListenBrainz 配置

```toml
[ListenBrainz]
Enabled = true
BaseURL = "https://api.listenbrainz.org/1/"
```

### 12.3 通用配置

```toml
[Server]
EnableScrobbleHistory = true  # 启用本地 scrobble 历史记录
EnableNowPlaying = true       # 启用 NowPlaying 广播
```
