# Navidrome 配置热加载机制深度解析

本文档全面分析 Navidrome 项目中配置项从注册、保存、变更广播到前端消费的完整实现流程。

---

## 一、配置分层架构概述

Navidrome 的配置系统采用 **四层架构**，不同层级的配置具有不同的持久化策略和热加载能力：

| 层级 | 存储位置 | 支持热加载 | 典型配置项 |
|------|---------|-----------|-----------|
| **系统全局配置** | 配置文件 / 环境变量 (Viper) | ❌ 需重启 | 端口、MusicFolder、日志级别 |
| **系统属性** | `property` 数据库表 | ✅ | 上次扫描错误、密码加密标记 |
| **用户偏好设置** | `user_props` 数据库表 | ✅ | Last.fm SessionKey、主题、语言 |
| **运行时实体配置** | `plugin`/`library`/`user`/`transcoding` 等表 | ✅ | 插件配置、媒体库、用户信息 |

---

## 二、配置项的注册与默认值来源

### 2.1 核心配置结构体定义

所有全局配置项集中定义在 `configOptions` 结构体中：

[configuration.go:27-L153](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L27-L153)

```go
type configOptions struct {
    ConfigFile                      string
    Address                         string
    Port                            int
    MusicFolder                     string
    DataFolder                      Dir
    LogLevel                        string
    SessionTimeout                  time.Duration
    // ... 约 120+ 个配置字段，包括嵌套子结构体
    Search                          searchOptions
    Scanner                         scannerOptions
    LastFM                          lastfmOptions
    Plugins                         pluginsOptions
    // DevFlags 开发调试选项
    DevEnableProfiler               bool
    DevShowArtistPage               bool
}
```

全局单例访问入口为包级变量 `Server`：

[configuration.go:287-L290](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L287-L290)

```go
var (
    Server = &configOptions{}
    hooks  []func()
)
```

### 2.2 默认值注册机制：`setViperDefaults()`

使用 **Viper** 配置管理库，所有默认值在 `setViperDefaults()` 函数中统一注册，该函数通过 `init()` 在包加载时自动调用：

[configuration.go:722-L887](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L722-L887)

```go
func setViperDefaults() {
    viper.SetDefault("musicfolder", filepath.Join(".", "music"))
    viper.SetDefault("datafolder", ".")
    viper.SetDefault("loglevel", "info")
    viper.SetDefault("port", 4533)
    viper.SetDefault("sessiontimeout", consts.DefaultSessionTimeout)
    viper.SetDefault("enableartworkprecache", true)
    viper.SetDefault("lastfm.enabled", true)
    viper.SetDefault("plugins.enabled", true)
    // ... 共 160+ 条 SetDefault 调用
}

func init() {
    setViperDefaults()
}
```

**命名约定**：配置键使用小写+点号（如 `scanner.schedule`），对应结构体字段的 PascalCase（`Scanner.Schedule`）。

### 2.3 配置加载流程：`InitConfig()` → `Load()`

[configuration.go:889-L936](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L889-L936)

```go
func InitConfig(cfgFile string, loadEnvVars bool) {
    // 1. 注册 INI 编解码器
    codecRegistry := viper.NewCodecRegistry()
    _ = codecRegistry.RegisterCodec("ini", ini.Codec{...})

    // 2. 确定配置文件路径（命令行参数 → ND_CONFIGFILE 环境变量 → 当前目录 navidrome.*）
    cfgFile = getConfigFile(cfgFile)

    // 3. 环境变量绑定：ND_ 前缀 + 点号替换为下划线
    if loadEnvVars {
        viper.SetEnvPrefix("ND")
        replacer := strings.NewReplacer(".", "_")
        viper.SetEnvKeyReplacer(replacer)
        viper.AutomaticEnv()
    }

    // 4. 读取配置文件
    _ = viper.ReadInConfig()
}
```

配置优先级：**环境变量 > 配置文件 > 默认值**

加载时执行的额外处理：
- **废弃选项映射**：如 `ReverseProxyWhitelist` → `ExtAuth.TrustedSources`
- **INI 格式适配**：将 `[default]` 节合并到根级别
- **计算字段填充**：`BaseURL` 解析为 `BasePath/BaseHost/BaseScheme`
- **Hook 调用**：通过 `AddHook()` 注册的初始化回调

---

## 三、前端配置注入机制

全局配置通过 `serve_index.go` 在渲染 `index.html` 时以 JSON 形式注入前端：

[serve_index.go:42-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go#L42-L83)

```go
appConfig := map[string]any{
    "version":                   consts.Version,
    "baseURL":                   conf.Server.BasePath,
    "defaultTheme":              conf.Server.DefaultTheme,
    "enableDownloads":           conf.Server.EnableDownloads,
    "lastFMEnabled":             conf.Server.LastFM.Enabled,
    "listenBrainzEnabled":       conf.Server.ListenBrainz.Enabled,
    "enableReplayGain":          conf.Server.EnableReplayGain,
    "pluginsEnabled":            conf.Server.Plugins.Enabled,
    // ... 约 35 个 UI 相关配置项
}
```

注入方式：将 JSON 序列化为字符串，通过 Go Template 嵌入 `window.__APP_CONFIG__`：

[serve_index.go:103-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go#L103-L114)

前端在 [config.js:50-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/config.js#L50-L58) 解析：

```javascript
const appConfig = JSON.parse(window.__APP_CONFIG__)
config = { ...defaultConfig, ...appConfig }
```

> **关键限制**：此注入发生在页面加载时，**全局配置修改后必须刷新页面才能生效**，实际上等同于需要重启（因为后端 `conf.Server` 本身不支持热修改）。

---

## 四、配置保存接口与持久层写入

### 4.1 系统属性表：`property`

**表结构**（2020 年初始化 Schema）：

[20200130083147_create_schema.go:142-L147](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/db/migrations/20200130083147_create_schema.go#L142-L147)

```sql
create table if not exists property (
    id    varchar(255) not null primary key,
    value varchar(255) default '' not null
);
```

**接口定义**：

[properties.go:3-L8](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/model/properties.go#L3-L8)

```go
type PropertyRepository interface {
    Put(id string, value string) error
    Get(id string) (string, error)
    Delete(id string) error
    DefaultGet(id string, defaultValue string) (string, error)
}
```

**Upsert 实现**：先尝试 UPDATE，影响行数为 0 时再 INSERT：

[property_repository.go:24-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/property_repository.go#L24-L36)

```go
func (r propertyRepository) Put(id string, value string) error {
    update := Update(r.tableName).Set("value", value).Where(Eq{"id": id})
    count, err := r.executeSQL(update)
    if count > 0 { return nil }
    insert := Insert(r.tableName).Columns("id", "value").Values(id, value)
    _, err = r.executeSQL(insert)
    return err
}
```

**典型使用场景**：
- 密码加密校验和标记（`consts.PasswordsEncryptedKey`）
- 扫描状态与错误信息（`consts.LastScanErrorKey`）

### 4.2 用户偏好表：`user_props`

**接口定义**：

[user_props.go:3-L8](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/model/user_props.go#L3-L8)

```go
type UserPropsRepository interface {
    Put(userId, key string, value string) error
    Get(userId, key string) (string, error)
    Delete(userId, key string) error
    DefaultGet(userId, key string, defaultValue string) (string, error)
}
```

**持久化实现**：复合主键 `(user_id, key)`：

[user_props_repository.go:24-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/user_props_repository.go#L24-L36)

```go
func (r userPropsRepository) Put(userId, key string, value string) error {
    update := Update(r.tableName).Set("value", value).
        Where(And{Eq{"user_id": userId}, Eq{"key": key}})
    // 同 property 表的 upsert 逻辑
}
```

**典型使用者——SessionKeys 封装**：

[session_keys.go:9-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/agents/session_keys.go#L9-L25)

```go
type SessionKeys struct {
    model.DataStore
    KeyName string  // 如 "LastFMSessionKey"、"ListenBrainzSessionKey"
}
```

**Last.fm 链接/取消链接 API**：

[auth_router.go:51-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/adapters/lastfm/auth_router.go#L51-L91)

| HTTP 方法 | 路径 | 操作 |
|-----------|------|------|
| `GET` | `/api/lastfm/link` | 获取当前用户的链接状态 + ApiKey |
| `DELETE` | `/api/lastfm/link` | 删除 SessionKey，取消授权 |
| `GET` | `/api/lastfm/link/callback` | OAuth 回调，保存 SessionKey |

### 4.3 媒体库（Library）配置

**REST API 路由注册**：

[native_api.go:88-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go#L88-L94)

```go
r.With(adminOnlyMiddleware).Group(func(r chi.Router) {
    api.RX(r, "/library", api.libs.NewRepository, true)  // GET/POST/PUT/DELETE
})
```

**保存逻辑（含热加载动作）**：

[library.go:162-L192](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L162-L192)

```go
func (r *libraryRepositoryWrapper) Save(entity any) (string, error) {
    // 1. 校验 + 持久化
    if err := r.validateLibrary(lib); err != nil { ... }
    err := r.LibraryRepository.Put(lib)

    // 2. 启动文件系统监听器（热加载）
    if r.watcher != nil {
        _ = r.watcher.Watch(r.ctx, lib)
    }

    // 3. 异步触发扫描
    if r.scanner != nil {
        go r.triggerScan(lib, "new")
    }

    // 4. 广播刷新事件
    event := &events.RefreshResource{}
    r.broker.SendBroadcastMessage(r.ctx, event.With("library", strconv.Itoa(lib.ID)))
}
```

**Update 操作的差异化处理**：路径变更时才重启监听器和扫描：

[library.go:194-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L194-L240)

### 4.4 用户-库关联配置

**专用 API**：

[library.go:15-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/library.go#L15-L22)

```go
r.Route("/user/{id}/library", func(r chi.Router) {
    r.Get("/",  getUserLibraries(api.libs))   // 获取用户已分配库
    r.Put("/", setUserLibraries(api.libs))    // 批量分配库
})
```

**业务逻辑**：

[library.go:69-L105](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L69-L105)

```go
func (s *libraryService) SetUserLibraries(ctx, userID string, libraryIDs []int) error {
    // 管理员不允许手动分配（自动拥有全部）
    if user.IsAdmin { return error }
    // 至少一个库
    if len(libraryIDs) == 0 { return error }
    // 写入 user_library 关联表
    _ = s.ds.User(ctx).SetUserLibraries(userID, libraryIDs)
    // 广播刷新事件
    s.broker.SendBroadcastMessage(ctx, event.With("user", userID).With("library", libIDs...))
}
```

### 4.5 插件配置（Plugin）

**路由端点**：

[plugin.go:17-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/plugin.go#L17-L31)

```go
r.Route("/plugin", func(r chi.Router) {
    r.Get("/", rest.GetAll(constructor))
    r.Post("/rescan", api.rescanPlugins)
    r.Route("/{id}", func(r chi.Router) {
        r.Get("/", rest.Get(constructor))
        r.Put("/", api.updatePlugin)  // 统一更新入口
    })
})
```

**统一更新函数 `updatePluginSettings()`**：支持配置、用户、库权限的增量更新 + 自动热重载：

[manager.go:440-L510](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L440-L510)

```go
func (m *Manager) updatePluginSettings(ctx, id string, updateFn func(*model.Plugin)) error {
    // 1. 读取当前插件 → 应用更新函数 → 写回 DB
    plugin, _ := repo.Get(id)
    updateFn(plugin)
    plugin.UpdatedAt = time.Now()

    // 2. 权限不足时自动禁用插件
    if manifest.Permissions.Users != nil && !hasValidUsersConfig(...) {
        _ = m.unloadPlugin(id)
        plugin.Enabled = false
        _ = repo.Put(plugin)
        m.sendPluginRefreshEvent(ctx, id)
        return nil
    }

    // 3. 已启用插件：卸载 → 用新配置重载
    if wasEnabled {
        _ = m.unloadPlugin(id)
        _ = m.loadPluginWithConfig(plugin)  // 热重载！
    }
    _ = repo.Put(plugin)
    m.sendPluginRefreshEvent(ctx, id)  // 通知前端刷新
}
```

**具体配置保存方法**：

```go
func (m *Manager) UpdatePluginConfig(ctx, id, configJSON string) error {
    return m.updatePluginSettings(ctx, id, func(p *model.Plugin) {
        p.Config = configJSON  // 保存 JSON 序列化后的配置
    })
}

func (m *Manager) UpdatePluginUsers(ctx, id, usersJSON string, allUsers bool) error { ... }
func (m *Manager) UpdatePluginLibraries(ctx, id, librariesJSON string, ...) error { ... }
```

### 4.6 Dev/调试只读配置接口

仅在 `DevUIShowConfig=true` 时启用的 GET 接口（**只读**，无 PUT/POST）：

[native_api.go:234-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go#L234-L238)

```go
func (api *Router) addConfigRoute(r chi.Router) {
    if conf.Server.DevUIShowConfig {
        r.Get("/config/*", getConfig)  // 序列化 conf.Server 并脱敏返回
    }
}
```

敏感字段脱敏处理（[config.go:20-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/config.go#L20-L94)）：
- **完全掩码**：`DevAutoCreateAdminPassword`、`PasswordEncryptionKey`、`Prometheus.Password`
- **部分掩码**：`LastFM.ApiKey`、`LastFM.Secret`（首尾字符可见）

---

## 五、配置变更事件广播机制（SSE Broker）

### 5.1 核心数据结构

基于 **Server-Sent Events (SSE)** 协议实现服务端→客户端的实时推送。

**事件接口定义**：

[events.go:21-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go#L21-L36)

```go
type Event interface {
    Name(Event) string   // 事件名：首字母小写的类型名（如 "refreshResource"）
    Data(Event) string   // JSON 序列化后的事件数据
}

type baseEvent struct{}  // 提供默认反射实现
```

**主要事件类型**：

| 事件类型 | 结构 | 用途 |
|---------|------|------|
| `refreshResource` | `RefreshResource` | 资源变更通知（配置/数据修改） |
| `scanStatus` | `ScanStatus` | 扫描进度 |
| `serverStart` | `ServerStart` | 服务启动/重连 |
| `keepAlive` | `KeepAlive` | 每 15s 心跳 |
| `nowPlayingCount` | `NowPlayingCount` | 当前播放人数 |

### 5.2 Broker 单例与消息管道

[sse.go:56-L80](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L56-L80)

```go
type broker struct {
    publish       messageChan     // 待发布事件（buffered=2）
    subscribing   clientsChan     // 新客户端连接
    unsubscribing clientsChan     // 客户端断开
}

func GetBroker() Broker {
    return singleton.GetInstance(func() *broker {
        broker := &broker{
            publish:       make(messageChan, 2),
            subscribing:   make(clientsChan, 1),
            unsubscribing: make(clientsChan, 1),
        }
        go broker.listen()  // 启动事件循环协程
        return broker
    })
}
```

### 5.3 发送 API：定向 vs 广播

[sse.go:82-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L82-L99)

```go
// 广播：所有连接的客户端（通过 context 标记）
func (b *broker) SendBroadcastMessage(ctx context.Context, evt Event) {
    ctx = broadcastToAll(ctx)
    b.SendMessage(ctx, evt)
}

// 定向：默认发给同用户的其他客户端（排除触发者自身）
func (b *broker) SendMessage(ctx context.Context, evt Event) {
    b.publish <- b.prepareMessage(ctx, evt)
}
```

**分发判定逻辑 `shouldSend()`**：

[sse.go:196-L211](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L196-L211)

```go
func (b *broker) shouldSend(msg message, c client) bool {
    // 1. broadcastToAll=true → 所有人（配置变更场景）
    if broadcastToAll { return true }
    // 2. 非客户端触发 → 所有人（系统事件）
    if !originatedFromClient { return true }
    // 3. 同一 clientUniqueId → 不发（避免回显）
    if c.clientUniqueId == clientUniqueId { return false }
    // 4. 同用户 → 发送（多端同步）
    if username == c.username { return true }
    return true
}
```

### 5.4 核心事件：`RefreshResource`

用于细粒度的资源刷新通知，支持 `*` 通配符：

[events.go:61-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go#L61-L89)

```go
type RefreshResource struct {
    baseEvent
    resources map[string][]string  // resourceType → [ids...]
}

func (rr *RefreshResource) With(resource string, ids ...string) *RefreshResource {
    if len(ids) == 0 {
        rr.resources[resource] = append(rr.resources[resource], Any) // Any = "*"
    }
    rr.resources[resource] = append(rr.resources[resource], ids...)
}
```

**使用示例（广播所有音乐资源变更）**：

[controller.go:234-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/controller.go#L234-L237)

```go
if s.changesDetected {
    s.broker.SendBroadcastMessage(ctx, &events.RefreshResource{})  // 全量刷新
}
```

**使用示例（细粒度插件刷新）**：

[manager.go:86-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L86-L92)

```go
func (m *Manager) sendPluginRefreshEvent(ctx context.Context, pluginIDs ...string) {
    event := (&events.RefreshResource{}).With("plugin", pluginIDs...)
    m.broker.SendBroadcastMessage(ctx, event)
}
```

### 5.5 listen 事件循环

[sse.go:213-L270](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L213-L270)

```go
func (b *broker) listen() {
    clients := map[client]struct{}{}
    for {
        select {
        case c := <-b.subscribing:
            clients[c] = struct{}{}
            sendOrDrop(c, serverStartEvent)  // 新连接立即发送启动事件

        case c := <-b.unsubscribing:
            close(c.msgC); delete(clients, c)

        case msg := <-b.publish:
            msg.id = getNextEventId()
            for c := range clients {
                if b.shouldSend(msg, c) {
                    sendOrDrop(c, msg)  // 非阻塞写入：满则丢弃
                }
            }

        case ts := <-keepAlive.C:
            // 每 15s 向所有客户端发送心跳
        }
    }
}
```

### 5.6 HTTP 处理器（SSE 握手）

[sse.go:133-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L133-L166)

关键响应头：
```
Content-Type: text/event-stream
Cache-Control: no-cache, no-transform
Connection: keep-alive
X-Accel-Buffering: no   // 禁用 Nginx 缓冲
```

---

## 六、前端事件消费链路

### 6.1 SSE 连接建立

[eventStream.js:7-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/eventStream.js#L7-L58)

```javascript
const newEventStream = async () => {
    let url = baseUrl(`${REST_URL}/events?jwt=${token}`)
    return new EventSource(url)  // 原生 EventSource API
}
```

**监听事件类型**：
```javascript
stream.addEventListener('serverStart', handler)
stream.addEventListener('scanStatus', throttledHandler)  // 100ms 节流
stream.addEventListener('refreshResource', handler)
stream.addEventListener('nowPlayingCount', handler)
stream.addEventListener('keepAlive', handler)  // 仅丢弃，不 dispatch
```

**自动重连**：出错后 5 秒重试，重连成功后 dispatch `streamReconnected`。

### 6.2 Redux 状态层：activityReducer

[activityReducer.js:25-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/reducers/activityReducer.js#L25-L61)

```javascript
case EVENT_REFRESH_RESOURCE:
    return {
        ...previousState,
        refresh: {
            lastReceived: Date.now(),  // 时间戳用于去重/顺序判定
            resources: data,           // { library: ['1','2'], user: ['uid'] }
        },
    }
```

### 6.3 组件消费 Hooks

**Hook 1：`useResourceRefresh` — React-Admin 资源刷新**

[useResourceRefresh.jsx:66-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useResourceRefresh.jsx#L66-L97)

```javascript
export const useResourceRefresh = (...visibleResources) => {
    const refreshData = useSelector(state => state.activity.refresh)

    // 收到 '*' 通配符 → 全页面 refresh()
    if (resources['*'] === '*' || anyResourceContainsWildcard) {
        refresh()
        return
    }
    // 指定资源 + 指定 ID → 调用 dataProvider.getMany() 精准刷新
    Object.keys(resources).forEach(r => {
        if (visibleResources.includes(r)) {
            dataProvider.getMany(r, { ids: resources[r] })
        }
    })
}
```

**Hook 2：`useRefreshOnEvents` — 自定义回调**

[useRefreshOnEvents.jsx:69-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useRefreshOnEvents.jsx#L69-L109)

```javascript
export const useRefreshOnEvents = ({ events, onRefresh }) => {
    // 监听指定事件类型，异步执行自定义 onRefresh 回调
    const shouldRefresh = events.some(eventType => resources[eventType])
    if (shouldRefresh) { onRefresh() }
}
```

---

## 七、需要重启才生效的配置提示

### 7.1 转码配置（Transcoding）安全提示

**配置项**：`EnableTranscodingConfig`

**提示文案定义**（英文 i18n 资源）：

[en.json:569-L570](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/i18n/en.json#L569-L570)

```json
{
  "message": {
    "transcodingDisabled":
      "Changing the transcoding configuration through the web interface is disabled for security reasons. If you would like to change (edit or add) transcoding options, restart the server with the %{config} configuration option.",
    "transcodingEnabled":
      "Navidrome is currently running with %{config}, making it possible to run system commands from the transcoding settings using the web interface. We recommend to disable it for security reasons and only enable it when configuring Transcoding options."
  }
}
```

**生效机制**：此开关通过 `conf.Server.EnableTranscodingConfig` 控制，**修改配置文件后必须重启服务**（`EnableTranscodingConfig` 无运行时热修改 API）。

### 7.2 其他需要重启的配置项（隐含约束）

根据代码架构分析，以下全局配置修改后 **必须重启 Navidrome**：

| 配置类别 | 示例配置项 | 原因 |
|---------|-----------|------|
| **网络/监听** | `Address`, `Port`, `TLSCert`, `TLSKey`, `UnixSocketPerm` | 服务启动时创建 Listener，无运行时重建逻辑 |
| **存储路径** | `MusicFolder`, `DataFolder`, `CacheFolder`, `DbPath` | 启动时初始化 DB 连接、文件系统监听器，运行时不可改 |
| **进程控制** | `EnforceNonRootUser`, `PID` | 仅在进程启动时校验/读取 |
| **功能开关（系统级）** | `Scanner.Enabled`, `Prometheus.Enabled`, `Jukebox.Enabled` | 启动时决定是否初始化对应子系统 |
| **认证/安全** | `PasswordEncryptionKey`, `AuthRequestLimit` | 启动时初始化加密器和限流组件 |
| **日志系统** | `LogLevel`, `LogFile`, `EnableLogRedacting`, `DevLogLevels` | 启动时配置 logrus 输出目标和级别 |
| **集成开关** | `EnableExternalServices` | 启动时决定是否注册 Last.fm/Deezer/ListenBrainz 代理 |
| **插件系统** | `Plugins.Enabled`, `Plugins.Folder`, `Plugins.CacheSize` | Manager.Start() 仅运行一次 |

**注意**：`MusicFolder` 在引入多库支持后已非推荐使用方式（改为 `library` 表管理，可热加载）。

### 7.3 配置文件与环境变量：无运行时重载机制

全局配置（`conf.Server`）的读取流程为：
1. **服务启动** → `InitConfig()` → `Load()` → 填充 `conf.Server`
2. **运行期间** → `conf.Server` 结构体字段被各处直接读取（指针值语义）
3. **无 Hot-Reload** → 未提供 `WatchConfig` / `OnConfigChange` 等 Viper 回调注册

因此所有 `conf/configuration.go` 中的配置项，修改配置文件或环境变量后都**需要重启进程**才能生效。

---

## 八、热加载能力总览矩阵

| 配置类型 | 修改方式 | 持久化目标 | 变更广播 | 前端即时生效 | 后端即时生效 |
|---------|---------|-----------|---------|------------|------------|
| **主题/语言（用户级）** | UI 选择器 → localStorage + Redux | 无（客户端存储） | ❌ | ✅ 无需后端 | ❌ N/A |
| **Last.fm 授权** | OAuth 回调 → SessionKeys.Put() | `user_props` 表 | ❌ | ✅ 前端自行轮询 | ✅ 下次 Scrobble 请求读取 |
| **媒体库路径/名称** | 管理员 → `/api/library` | `library` 表 | ✅ `RefreshResource{library}` | ✅ Hook 自动刷新 | ✅ 重启监听器+触发扫描 |
| **用户-库关联** | 管理员 → `/api/user/{id}/library` | `user_library` 表 | ✅ `RefreshResource{user,library}` | ✅ Hook 自动刷新 | ✅ 下次请求校验权限 |
| **插件启停** | 管理员 → `/api/plugin/{id}` PUT | `plugin` 表 | ✅ `RefreshResource{plugin}` | ✅ Hook 自动刷新 | ✅ `unloadPlugin` → `loadPluginWithConfig` |
| **插件配置** | 管理员 → SchemaEditor → PUT config | `plugin.config` 字段 | ✅ 同上 | ✅ 同上 | ✅ 同上（卸载+重载） |
| **扫描计划** | 通过配置文件 | `conf.Server.Scanner.Schedule` | ❌ | ❌ 需刷新 | ❌ 需重启 Scheduler |
| **端口/日志级别** | 通过配置文件/环境变量 | `conf.Server.*` | ❌ | ❌ 刷新也没用 | ❌ 必须重启进程 |

---

## 九、关键文件索引

| 文件路径 | 核心职责 |
|---------|---------|
| [conf/configuration.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go) | 全局配置结构体、默认值、加载逻辑 |
| [server/serve_index.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go) | 配置注入前端（`__APP_CONFIG__`） |
| [server/events/sse.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go) | SSE Broker、事件循环、客户端管理 |
| [server/events/events.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go) | 事件类型定义（RefreshResource 等） |
| [server/nativeapi/native_api.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go) | Native API 路由注册（adminOnly 配置入口） |
| [server/nativeapi/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/config.go) | Dev 模式配置只读接口 + 脱敏 |
| [server/nativeapi/library.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/library.go) | 用户-库关联 API |
| [server/nativeapi/plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/plugin.go) | 插件配置更新 API |
| [core/library.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go) | Library 保存/更新/删除 + 事件广播 |
| [plugins/manager.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go) | 插件生命周期、配置热重载 |
| [persistence/property_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/property_repository.go) | 系统属性 Upsert 实现 |
| [persistence/user_props_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/user_props_repository.go) | 用户偏好 Upsert 实现 |
| [core/agents/session_keys.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/agents/session_keys.go) | UserProps 封装（Last.fm/ListenBrainz Session） |
| [adapters/lastfm/auth_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/adapters/lastfm/auth_router.go) | Last.fm 授权 API |
| [ui/src/eventStream.js](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/eventStream.js) | 前端 SSE 连接 + 事件分发 |
| [ui/src/reducers/activityReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/reducers/activityReducer.js) | Redux 事件状态管理 |
| [ui/src/common/useResourceRefresh.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useResourceRefresh.jsx) | react-admin 资源自动刷新 Hook |
| [ui/src/common/useRefreshOnEvents.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useRefreshOnEvents.jsx) | 自定义回调刷新 Hook |
| [ui/src/i18n/en.json](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/i18n/en.json) | 重启提示文案定义 |
