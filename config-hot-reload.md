# Navidrome 配置热加载机制梳理

## 一、配置体系概览

Navidrome 的"设置"实际分为 **两个完全不同的层次**，它们的热加载路径截然不同：

| 层次 | 存储 | 可在管理端运行时修改 | 热加载能力 |
|------|------|---------------------|-----------|
| 服务器级配置 `conf.Server` | 配置文件 / 环境变量 | ❌ 只读展示 | ❌ 需重启 |
| 运行时数据（Library / Plugin / User 等） | SQLite 数据库 | ✅ REST API 增删改 | ✅ 实时生效 |

---

## 二、服务器级配置 `conf.Server` — 不支持热加载

### 2.1 配置加载入口

`conf/configuration.go` 定义全局单例：

```go
var Server = &configOptions{}
```

启动时通过 `conf.Load()` 一次性加载，流程为：

1. `conf.InitConfig()` — 初始化 Viper，绑定环境变量前缀 `ND_`
2. `conf.Load()` — 读取配置文件 → 反序列化到 `conf.Server` → 运行校验 → 执行注册的 Hook

关键代码位于 `conf/configuration.go` 的 `Load()` 函数：

```go
func Load(noConfigDump bool) {
    parseIniFileConfiguration()
    remapEnvVarKeysFromConfig()
    // ...
    viper.Unmarshal(&Server, ...)  // 一次性反序列化
    // 校验 + 后处理
    for _, hook := range hooks {
        hook()  // 执行初始化钩子
    }
}
```

### 2.2 管理端配置查看接口

`server/nativeapi/native_api.go` 的 `addConfigRoute()` 注册了只读的配置端点：

```go
func (api *Router) addConfigRoute(r chi.Router) {
    if conf.Server.DevUIShowConfig {
        r.Get("/config/*", getConfig)
    }
}
```

`server/nativeapi/config.go` 中 `getConfig` 仅做 GET 读取，序列化 `conf.Server` 并对敏感字段脱敏后返回。**没有 PUT/POST handler，不支持运行时修改**。

### 2.3 配置在运行时的消费方式

大部分代码在 **每次请求时** 直接读取 `conf.Server.*` 字段（如 `conf.Server.EnableDownloads`、`conf.Server.EnableSharing`），属于"引用式"消费，如果 `conf.Server` 被修改，理论上会立即生效。但问题在于 **没有任何运行时机制去修改它**：

- 配置 API 是只读的
- 没有对配置文件的 `fsnotify` 监听
- `SIGHUP` 信号在 `cmd/root.go` 中被注册，但仅用于触发 context 取消（关闭进程），不会重载配置
- `SIGUSR1` 信号（`cmd/signaller_unix.go`）只触发音乐库扫描，不涉及配置

### 2.4 部分配置在启动时缓存

一些子系统在启动时读取配置并缓存，即使 `conf.Server` 变化也无法生效：

| 子系统 | 缓存的配置项 | 代码位置 |
|--------|-------------|---------|
| 定时扫描调度 | `Scanner.Schedule` | `cmd/root.go` |
| 定时备份调度 | `Backup.Schedule` | `cmd/root.go` |
| 文件监控防抖间隔 | `Scanner.WatcherWait` | `scanner/watcher.go` |
| Scanner 控制器 | `DevExternalScanner` | `scanner/controller.go` |
| 插件文件监控 | `Plugins.AutoReload` | `plugins/manager.go` |

### 2.5 `conf.AddHook` 机制

`conf/configuration.go` 提供了钩子注册：

```go
func AddHook(hook func()) {
    hooks = append(hooks, hook)
}
```

Hook 仅在 `conf.Load()` 末尾执行一次，用于初始化派生配置：

| 注册者 | 作用 |
|--------|------|
| `model/tag_mappings.go` | 初始化标签映射规则 |
| `core/artwork/reader_resized.go` | 初始化封面图质量参数 |
| `conf/mime/mime_types.go` | 初始化 MIME 类型 |
| `adapters/lastfm/agent.go` | 初始化 LastFM 语言列表 |
| `adapters/deezer/deezer.go` | 初始化 Deezer 语言列表 |
| `adapters/gotaglib/gotaglib.go` | 初始化标签库配置 |
| `adapters/listenbrainz/agent.go` | 初始化 ListenBrainz 配置 |

---

## 三、前端发起配置变更到接口调用的完整链路

Navidrome 前端基于 **react-admin** 框架构建，通过标准化的 dataProvider 抽象层与后端 API 通信。

### 3.1 前端架构概览

```
┌──────────────────────────────────────────────────────────────────┐
│                        UI 组件层 (JSX)                            │
│  LibraryEdit / LibraryCreate / PluginShow / UserEdit / ...        │
│  - 使用 useMutation / useUpdate / useDeleteWithConfirmController  │
│  - 自定义 save() callback                                         │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                  react-admin DataProvider 抽象层                   │
│  wrapperDataProvider (ui/src/dataProvider/wrapperDataProvider.js) │
│  - 包装 ra-data-json-server                                       │
│  - 特殊资源路由映射 (playlistTrack, user 库关联)                   │
│  - 注入 library 过滤参数                                          │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                     HTTP 客户端层                                  │
│  httpClient (ui/src/dataProvider/httpClient.js)                   │
│  - 添加 X-ND-Authorization (Bearer JWT) 头                        │
│  - 添加 X-ND-Client-Unique-Id (客户端 UUID)                       │
│  - 拦截响应头刷新 token                                            │
│  - base URL: /api (定义于 ui/src/consts.js → REST_URL)            │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                  后端 Go Chi Router                               │
│  server/nativeapi/native_api.go: Router.routes()                  │
│  - 权限中间件 adminOnlyMiddleware (adminOnly=true 的资源)          │
│  - 映射 REST 动词到 Repository 包装层方法                          │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 音乐库（Library）变更完整链路

#### 3.2.1 编辑音乐库（PUT /api/library/{id}）

**前端入口**：`ui/src/library/LibraryEdit.jsx`

```
用户点击"Save"按钮
  │
  ▼
LibraryEdit.save()  [LibraryEdit.jsx#L60-L82]
  │
  ├─ useMutation() 提交
  │   type: 'update'
  │   resource: 'library'
  │   payload: { id, data: { name, path, defaultNewUsers } }
  │
  ▼
wrapperDataProvider.update('library', params)
  └─ mapResource() → 无特殊映射，直接透传
  │
  ▼
ra-data-json-server → JSON REST API 适配
  └─ 生成 PUT /api/library/{id}
     Body: { "name": "...", "path": "...", "defaultNewUsers": true }
  │
  ▼
httpClient(url, { method: 'PUT', body, headers })
  ├─ 注入 X-ND-Authorization: Bearer <token>
  ├─ 注入 X-ND-Client-Unique-Id: <uuid>
  └─ Accept: application/json
  │
  ▼
Go 后端 Chi Router
  │
  ├─ 中间件链：
  │   ├─ JWTMiddleware → 解析 JWT，设置 ctx 用户身份
  │   ├─ adminOnlyMiddleware → 校验 role == "admin"
  │   └─ URLParamParserMiddleware → 解析 /{id} 参数
  │
  ▼
Router.routes() 中注册的 PUT handler
  └─ libraryRepositoryWrapper.Update(ctx, id, entity)
     [core/library.go#L194-L240]
     │
     ├─ 1) 持久化：LibraryRepository.Put(lib) 写入 SQLite
     │
     ├─ 2) 路径变更检测：
     │   ├─ if lib.Path != oldPath:
     │   │   ├─ watcher.Watch(ctx, lib)  ← 重启该库的文件监控
     │   │   │   (关闭旧的 watcher，新建 fsnotify watcher)
     │   │   └─ scanner.ScanAll(ctx)    ← 异步触发全库扫描
     │   └─ else: 跳过
     │
     ├─ 3) 事件广播：
     │   └─ broker.SendBroadcastMessage(ctx,
     │        &RefreshResource{}.With("library", string(id)))
     │      ↓
     │      server/events/sse.go Broker
     │      - 遍历所有已连接 SSE 客户端
     │      - shouldSend() 过滤
     │      - sendOrDrop() 写入每个客户端的 channel
     │
     └─ 4) 返回序列化后的 library JSON
  │
  ▼
httpClient 处理响应
  ├─ 检查响应头 X-ND-Authorization，若有则更新 localStorage token
  └─ 返回 Promise resolve
  │
  ▼
LibraryEdit.save() 收到成功响应
  ├─ notify('resources.library.notifications.updated')
  └─ redirect('/library')  ← 跳转到库列表页
  │
  ▼
SSE 事件到达所有在线客户端
  │
  ▼
eventStream.js EventSource 监听 refreshResource 事件
  ├─ dispatch(processEvent('refreshResource', data))
  │   ↓
  │   reducers/activityReducer.js
  │   - 更新 state.activity.refresh:
  │     { lastReceived: Date.now(), resources: {"library":["id"]} }
  │
  ▼
订阅了该事件的组件触发重新渲染
  │
  ├─ LibraryList.jsx 使用 useResourceRefresh('library')
  │   [common/useResourceRefresh.jsx]
  │   - 读取 state.activity.refresh
  │   - 检测到 library 资源变更
  │   - 调用 dataProvider.getMany('library', {ids: ["id"]})
  │   - react-admin 缓存更新 → UI 自动重渲染
  │
  └─ LibrarySelector.jsx 使用 useRefreshOnEvents({events:['library',...]})
      - 检测到 library 事件
      - 触发自定义 onRefresh() 重新加载用户可用库列表
```

#### 3.2.2 新建音乐库（POST /api/library）

**前端入口**：`ui/src/library/LibraryCreate.jsx#L25-L71`

与编辑类似，`useMutation()` type 为 `'create'`，后端走 `libraryRepositoryWrapper.Create()`，
成功后同样触发 `watcher.Watch()` + `scanner.ScanAll()` + SSE 广播。

#### 3.2.3 删除音乐库（DELETE /api/library/{id}）

**前端入口**：`ui/src/library/DeleteLibraryButton.jsx`

使用 react-admin 的 `useDeleteWithConfirmController`，弹出确认对话框后发起 DELETE 请求。
后端删除后额外调用 `pluginManager.UnloadDisabledPlugins()` 清理因权限丢失而被自动禁用的插件。

#### 3.2.4 触发扫描（GET /rest/startScan）

**前端入口**：`ui/src/library/LibraryScanButton.jsx#L23-L49`

扫描按钮不通过 dataProvider，而是直接调用 `subsonic/index.js` 的 `startScan()`：

```
LibraryScanButton.handleClick()
  │
  ├─ subsonic.startScan({ fullScan, target: ["1:", "2:"] })
  │   ↓
  │   subsonic/url() 构建 Subsonic 风格 URL:
  │   /rest/startScan?u=<user>&t=<token>&s=<salt>&f=json&v=1.8.0&c=NavidromeUI
  │   &target=1:&target=2:  (每个 libraryID: 表示扫描整个库)
  │
  ▼
httpClient 发送 GET 请求
  │
  ▼
后端 Subsonic API handler startScan()
  ├─ scanner.Rescan(ctx, mediaFolderIds, fullScan)
  └─ 扫描完成后在 scanner/controller.go 中广播全局 RefreshResource
```

### 3.3 插件（Plugin）变更完整链路

插件管理使用 react-admin 的 **Show** 视图（而非 Edit 视图），因为一个插件有多个独立的可修改区域（配置、用户权限、库权限、启用状态），需要分开发送更新请求。

#### 3.3.1 插件主容器

**前端入口**：`ui/src/plugin/PluginShow.jsx`

`PluginShowLayout` 组件通过 `useShowController` 获取插件数据后，将其拆分为多个独立的 state：

```
PluginShow.jsx 初始化
  │
  ├─ state.configData       ← 来自 record.config JSON 字符串
  ├─ state.selectedUsers + state.allUsers   ← 来自 record.users + record.allUsers
  ├─ state.selectedLibraries + state.allLibraries + state.allowWriteAccess
  │                         ← 来自 record.libraries + record.allLibraries + ...
  └─ state.isDirty          ← 是否有未保存变更
  │
  ├─ useUpdate() hook 预配置：
  │   resource: 'plugin', id: record.id, undoable: false
  │   onSuccess: refresh() + setIsDirty(false) + notify()
  │
  └─ handleSaveConfig()  [PluginShow.jsx#L200-L234]
      │
      ├─ 按 manifest 权限门控组装 payload：
      │   {
      │     config: JSON.stringify(configData),      // 仅当有 config.schema
      │     users: JSON.stringify(selectedUsers),    // 仅当有 permissions.users
      │     allUsers: allUsers,
      │     libraries: JSON.stringify(selectedLibraries), // 仅当有 permissions.library
      │     allLibraries: allLibraries,
      │     allowWriteAccess: allowWriteAccess,
      │   }
      │
      └─ updatePlugin('plugin', record.id, data, record)
          ↓
          dataProvider.update('plugin', { id, data })
          ↓
          PUT /api/plugin/{id}
```

#### 3.3.2 启用/禁用插件开关

**前端入口**：`ui/src/plugin/ToggleEnabledSwitch.jsx#L57-L81`

这是一个独立的 `useUpdate()` 调用，在列表页和详情页都可用：

```
ToggleEnabledSwitch.handleClick()
  │
  ├─ useUpdate('plugin', id, { enabled: !record.enabled }, record)
  │   ↓
  │   PUT /api/plugin/{id}
  │   Body: { "enabled": false }
  │
  ├─ onSuccess: refresh() + notify()
  └─ onFailure: refresh() + notify(error)
```

#### 3.3.3 后端插件更新处理

`server/nativeapi/plugin.go` 的 `updatePlugin()` 根据请求 body 的字段分发到不同逻辑：

```
PUT /api/plugin/{id}
  │
  ▼
PluginUpdateRequest 绑定 JSON body
  │
  ▼
updatePlugin(ctx, id, req)  [plugin.go#L68-L167]
  │
  ├─ 如果 req.Config 字段存在：
  │   ValidatePluginConfig(id, req.Config)  ← AJV schema 校验
  │   UpdatePluginConfig(id, req.Config)
  │     │
  │     ▼ manager.go updatePluginSettings()
  │       ├─ repo.Get(id) → 从 DB 读 plugin
  │       ├─ plugin.Config = req.Config  ← 修改内存对象
  │       ├─ repo.Put(plugin)  ← 写回 DB
  │       ├─ unloadPlugin(id)
  │       │   ├─ delete(Manager.plugins, id)  ← 从内存 map 移除
  │       │   ├─ plugin.Close()               ← 调用插件清理函数
  │       │   └─ compiledPlugin.Close(ctx)    ← 释放 wazero 编译缓存
  │       └─ loadPluginWithConfig(plugin)
  │           ├─ manager.loadPlugin()  ← 读取 .ndp 包
  │           ├─ wazero.CompileModule()  ← 重新编译 WASM
  │           ├─ instantiateModule()    ← 实例化 WASM
  │           ├─ 初始化沙箱环境 (fs, host functions)
  │           └─ Manager.plugins[id] = loaded
  │
  ├─ 如果 req.Users / req.AllUsers 字段存在：
  │   UpdatePluginUsers(id, req.Users, req.AllUsers)
  │     ↓ 同样走 updatePluginSettings() + unload+reload
  │     + 权限门控校验：若 users 为空且 !allUsers → DisablePlugin()
  │
  ├─ 如果 req.Libraries / req.AllLibraries / req.AllowWriteAccess 存在：
  │   UpdatePluginLibraries(...)
  │     ↓ 同上，附带写权限门控校验
  │
  └─ 如果 req.Enabled 字段存在：
      ├─ true  → EnablePlugin(id)
      │           loadPluginWithConfig() + repo.Put + sendPluginRefreshEvent()
      └─ false → DisablePlugin(id)
                  unloadPlugin() + repo.Put + sendPluginRefreshEvent()
  │
  ▼
sendPluginRefreshEvent() → SendBroadcastMessage(RefreshResource{plugin: [id]})
  │
  ▼
所有在线客户端 SSE 收到 {"plugin":["id"]}
  │
  ▼
PluginShow.jsx 通过 useResourceRefresh('plugin')  [PluginShow.jsx#L34]
  → dataProvider.getMany('plugin', {ids: [id]})
  → react-admin 重新获取数据 → useShowController 更新 record
  → useEffect 检测到 record 变化 → 重新初始化本地 state
```

#### 3.3.4 插件文件自动重载（.ndp 文件变更）

当 `Plugins.AutoReload = true` 时，`plugins/manager_watcher.go` 监控 `.ndp` 文件变化。这是文件系统级别的变更，不经过 REST API。

**完整处理流程**（`manager_watcher.go` L135-L217 `processPluginEvent`）：

```
文件系统事件 (CREATE/WRITE/REMOVE/RENAME)
  │
  ▼
handleWatcherEvent(event)  [L84-L109]
  │
  ├─ 过滤非 .ndp 文件
  ├─ 2 秒防抖（取消旧 timer，新建 timer）
  └─ 防抖到期后调用 processPluginEvent(pluginName)
  │
  ▼
processPluginEvent(pluginName)  [L135-L217]
  │
  ├─ 根据文件存在性判断 action：
  │   ├─ os.Stat(path) 成功 → actionUpdate（文件存在）
  │   └─ os.Stat(path) 失败 → actionRemove（文件已删除）
  │
  ├─ actionUpdate（文件存在/变更）：
  │   │
  │   ├─ 1) SHA256 比对  [L161-L184]
  │   │   ├─ 计算新文件 SHA256
  │   │   ├─ 与 DB 中 dbPlugin.SHA256 比较
  │   │   └─ 相同 → return（无实际变更，跳过）
  │   │
  │   ├─ 2) 提取 manifest  [L186-L199]
  │   │   │
  │   │   ├─ 提取失败：
  │   │   │   ├─ 设置 dbPlugin.LastError = err.Error()
  │   │   │   ├─ dbPlugin.UpdatedAt = time.Now()
  │   │   │   ├─ **如果 dbPlugin.Enabled == true**：
  │   │   │   │   ├─ m.unloadPlugin(pluginName)  ← 卸载内存中的插件
  │   │   │   │   ├─ dbPlugin.Enabled = false      ← 标记为禁用
  │   │   │   │   └─ repo.Put(dbPlugin)
  │   │   │   └─ **注意：此路径不调用 sendPluginRefreshEvent！**
  │   │   │      前端不会收到 SSE 刷新通知（潜在 bug）
  │   │   │
  │   │   └─ 提取成功：
  │   │       └─ 调用 m.updatePluginInDB()  [L201]
  │   │           ├─ 更新 DB 中的 manifest、SHA256、版本等信息
  │   │           ├─ 如果插件已启用 → unloadPlugin() + loadPluginWithConfig()
  │   │           └─ 调用 sendPluginRefreshEvent() 广播 SSE 事件
  │   │
  │   └─ 小结：
  │      ✅ 提取成功 → 重载 + 广播
  │      ❌ 提取失败 + 已启用 → 禁用 + 不广播（前端需手动刷新）
  │
  └─ actionRemove（文件被删除）：
      │
      ├─ repo.Get(pluginName) → 从 DB 获取插件
      └─ 调用 m.removePluginFromDB()
          ├─ 如果插件已启用 → unloadPlugin()
          ├─ 从 DB 删除插件记录
          └─ 调用 sendPluginRefreshEvent() 广播 SSE 事件
```

`sendPluginRefreshEvent` 的实现（`manager.go` L86-L92）：

```go
func (m *Manager) sendPluginRefreshEvent(ctx context.Context, pluginIDs ...string) {
    if m.broker == nil {
        return
    }
    event := (&events.RefreshResource{}).With("plugin", pluginIDs...)
    m.broker.SendBroadcastMessage(ctx, event)
}
```

**插件文件变更行为总结**：

| 场景 | 内存操作 | 广播 SSE |
|------|---------|---------|
| SHA256 未变 | 跳过 | ❌ |
| SHA256 变化 + manifest 提取成功 + 已启用 | unload + reload | ✅ `{"plugin":["id"]}` |
| SHA256 变化 + manifest 提取成功 + 未启用 | 仅更新 DB | ✅ `{"plugin":["id"]}` |
| SHA256 变化 + manifest 提取失败 + 已启用 | unload + Enabled=false | ❌（潜在 bug） |
| SHA256 变化 + manifest 提取失败 + 未启用 | 仅更新 DB + LastError | ❌（潜在 bug） |
| .ndp 文件被删除 + 已启用 | unload + DB 删除 | ✅ `{"plugin":["id"]}` |
| .ndp 文件被删除 + 未启用 | 仅 DB 删除 | ✅ `{"plugin":["id"]}` |

### 3.4 用户（User）变更完整链路

用户管理的三种操作（修改基础信息、修改库关联、删除用户）在后端有**完全不同**的广播行为，需要仔细区分。

#### 3.4.1 修改用户基础信息

**前端入口**：`ui/src/user/UserEdit.jsx#L83-L105`

修改用户名、密码、邮箱、管理员权限等基础信息：

```
UserEdit.save()
  │
  ├─ useMutation()
  │   type: 'update'
  │   resource: 'user'
  │   payload: { id, data: { userName, name, email, isAdmin, ... } }
  │
  ▼
wrapperDataProvider.update('user', params)  [wrapperDataProvider.js#L178-L184]
  │
  ▼
updateUser(params) 阶段 1  [wrapperDataProvider.js#L128-L146]
  │
  ▼
PUT /api/user/{id}
  Body: { "userName": "...", "name": "...", "email": "...", ... }
  │
  ▼
Go 后端 Router
  │
  ▼
userRepositoryWrapper.Update(id, entity)  [core/user.go#L57-L60]
  │
  └─ 直接委托给底层 UserRepository.Update()
     ↓
     仅更新 DB，**不调用 broker.SendBroadcastMessage()**
     ❌ 无 SSE 广播
```

**关键代码**（`core/user.go` L57-L60）：

```go
func (r *userRepositoryWrapper) Update(id string, entity any, cols ...string) error {
    return r.UserRepository.(rest.Persistable).Update(id, entity, cols...)
}
```

`Update()` 方法完全透传到底层，**没有任何事件广播逻辑**。同理，`Save()`（新建用户）也是直接透传，不广播事件。

> 🔍 **行为结论**：修改用户基础信息（用户名、密码、邮箱、isAdmin 等）后，**所有在线客户端都不会收到 SSE 刷新通知**。如果另一个管理员打开了用户列表页面，他必须手动刷新浏览器才能看到变更。

#### 3.4.2 修改用户库关联

**前端入口**：同上，UserEdit 中的 `LibrarySelectionField` 组件

当为非管理员用户分配/取消库访问权限时，走独立的 API 端点：

```
UserEdit.save()
  │
  ▼
updateUser(params) 阶段 2  [wrapperDataProvider.js#L140-L146]
  │
  └─ if !userData.isAdmin && libraryIds !== undefined:
      handleUserLibraryAssociation(userId, libraryIds)
        ↓
        PUT /api/user/{id}/library
        Body: { "libraryIds": ["1", "3"] }
  │
  ▼
Go 后端 Router → libraryService.SetUserLibraries()
  [core/library.go#L69-L105]
  │
  ├─ 1) 校验：admin 用户不能手动分配，普通用户至少 1 个库
  ├─ 2) s.ds.User(ctx).SetUserLibraries(userID, libraryIDs)  ← 更新 DB
  │
  └─ 3) ✅ 广播 SSE 事件：
      event := &events.RefreshResource{}
      event = event.With("user", userID).With("library", libIDs...)
      s.broker.SendBroadcastMessage(ctx, event)
```

**关键代码**（`core/library.go` L99-L104）：

```go
// Send refresh event to all clients
event := &events.RefreshResource{}
libIDs := slice.Map(libraryIDs, func(id int) string { return strconv.Itoa(id) })
event = event.With("user", userID).With("library", libIDs...)
s.broker.SendBroadcastMessage(ctx, event)
```

> 🔍 **行为结论**：修改用户库关联后，**会同时广播 `user` 和 `library` 两个资源的刷新事件**。所有在线客户端的用户列表和库列表都会自动刷新。

#### 3.4.3 用户删除

**前端入口**：`ui/src/user/DeleteUserButton.jsx`（react-admin 标准删除按钮）

```
DELETE /api/user/{id}
  │
  ▼
userRepositoryWrapper.Delete(id)  [core/user.go#L62-L75]
  │
  ├─ 1) 底层 Delete(id)  ← DB 级联清理
  │   └─ 清理 plugin_user 权限引用表
  │
  ├─ 2) r.pluginManager.UnloadDisabledPlugins(r.ctx)
  │   │   [plugins/manager.go#L546-L588]
  │   │
  │   ├─ 查询 DB 中所有 enabled=false 的插件
  │   ├─ 检查是否仍在内存中（m.plugins map）
  │   ├─ 对仍在内存中的执行 unloadPlugin()
  │   │   ├─ delete(m.plugins, id)
  │   │   ├─ plugin.Close()
  │   │   └─ compiledPlugin.Close(ctx)
  │   │
  │   └─ 如果有卸载的插件：
  │      m.sendPluginRefreshEvent(ctx, unloaded...)
  │      ↓
  │      广播 {"plugin": ["id1", "id2"]}
  │
  └─ ❌ **没有直接广播 user 资源的刷新事件！**
```

**关键代码**（`core/user.go` L62-L75）：

```go
func (r *userRepositoryWrapper) Delete(id string) error {
    err := r.UserRepository.(rest.Persistable).Delete(id)
    if err != nil {
        return err
    }
    r.pluginManager.UnloadDisabledPlugins(r.ctx)
    return nil
}
```

注意：`Delete()` 方法本身**没有调用 broker.SendBroadcastMessage() 广播 user 事件**！

> 🔍 **行为结论**：
> - 用户删除后，**不会直接广播 `user` 资源的刷新事件**
> - 只有当删除用户导致某些插件因权限不满足而被卸载时，才会**间接广播 `plugin` 资源的刷新事件**
> - 其他在线客户端的用户列表不会自动刷新，需要手动刷新

#### 3.4.4 用户管理广播行为总结

| 操作 | 直接广播 SSE | 间接广播 SSE | 前端自动刷新 |
|------|-------------|-------------|-------------|
| 修改用户基础信息（用户名、密码等） | ❌ 无 | ❌ 无 | ❌ 需手动刷新 |
| 修改用户库关联 | ✅ `{"user":["id"], "library":["1","3"]}` | ❌ 无 | ✅ 用户列表 + 库列表自动刷新 |
| 删除用户 | ❌ 无 | ✅ `{"plugin":["id"]}`（如果有插件被卸载） | ⚠️ 仅插件列表自动刷新，用户列表需手动刷新 |
| 新建用户 | ❌ 无 | ❌ 无 | ❌ 需手动刷新 |

### 3.5 Subsonic API 调用链路

扫描触发、播放报告等功能走 **Subsonic API**（`/rest/*`）而非 Native API（`/api/*`），URL 格式不同但使用同一个 `httpClient`：

```
subsonic/url() 构建
  │
  ├─ 读取 localStorage:
  │   username, subsonic-token, subsonic-salt
  │
  ├─ 标准 Subsonic 参数：
  │   u=username, t=token, s=salt
  │   f=json, v=1.8.0, c=NavidromeUI
  │
  └─ 附加业务参数 (id, options 展开)

示例：/rest/startScan?u=admin&t=abc&s=xyz&f=json&v=1.8.0&c=NavidromeUI&target=1:
```

---

## 四、运行时数据变更广播与前端消费

### 4.1 变更广播基础设施：SSE Broker

`server/events/events.go` 和 `server/events/sse.go` 实现了基于 Server-Sent Events 的实时推送：

```
┌──────────────────────────────────────────────┐
│                 Broker (单例)                  │
│  GetBroker() → singleton                     │
│                                              │
│  publish chan  ──→  listen() goroutine        │
│                         │                    │
│                    遍历 clients map           │
│                    ├─ shouldSend() 过滤       │
│                    └─ sendOrDrop() 投递       │
│                                              │
│  SendMessage()        ── 发送给同用户的其他客户端  │
│  SendBroadcastMessage() ── 发送给所有客户端       │
└──────────────────────────────────────────────┘
```

SSE 端点挂载在 `server/server.go`：

```go
if conf.Server.DevActivityPanel {
    r.Handle(path.Join(..., "events"), s.broker)
}
```

前端通过 `ui/src/eventStream.js` 订阅事件：

```js
stream.addEventListener('refreshResource', eventHandler(dispatchFn))
stream.addEventListener('scanStatus', throttledEventHandler(dispatchFn))
stream.addEventListener('nowPlayingCount', eventHandler(dispatchFn))
```

### 4.2 核心事件类型

| 事件类型 | 触发场景 | 数据格式 |
|---------|---------|---------|
| `RefreshResource` | 资源变更后通知 UI 刷新 | `{"library":["1"]}` 或 `{"*":"*"}` |
| `ScanStatus` | 扫描进度更新 | `{"scanning":true,"count":100,...}` |
| `NowPlayingCount` | 正在播放数量变化 | `{"count":3}` |
| `ServerStart` | 新客户端连接时推送 | `{"startTime":"...","version":"..."}` |
| `KeepAlive` | 每 15 秒心跳 | `{"ts":1700000000}` |

### 4.3 前端消费 SSE 事件的两种 Hook

#### useResourceRefresh（react-admin 资源刷新）

`ui/src/common/useResourceRefresh.jsx`：适用于 react-admin 管理的资源（library、plugin、album 等）。

工作流程：
1. 从 Redux store 读取 `state.activity.refresh`
2. 比较 `lastReceived` 时间戳判断是否有新事件
3. 按资源类型匹配：
   - `{"*":"*"}` → 调用 `refresh()` 全局刷新
   - `{"library":["1","2"]}` → 调用 `dataProvider.getMany('library', {ids:["1","2"]})`
4. react-admin 缓存更新后，订阅该资源的组件自动重渲染

使用示例：
```jsx
// PluginShow.jsx
useResourceRefresh('plugin')  // 插件数据变化时自动 getMany 刷新

// LibraryList.jsx
useResourceRefresh('library') // 库列表变化时自动刷新
```

#### useRefreshOnEvents（自定义回调刷新）

`ui/src/common/useRefreshOnEvents.jsx`：适用于需要自定义刷新逻辑的场景（非 react-admin 直接管理的数据）。

工作流程：
1. 同样监听 `state.activity.refresh`
2. 按 `events` 数组匹配资源类型
3. 匹配成功后调用用户传入的 `onRefresh()` 异步回调

使用示例：
```jsx
// LibrarySelector 中重新加载用户可用库列表
useRefreshOnEvents({
    events: ['library', 'user'],
    onRefresh: loadUserLibraries  // 自定义异步函数
})
```

### 4.4 Redux 事件分发

SSE 事件到达前端后，通过 Redux reducer 更新全局状态：

`ui/src/reducers/activityReducer.js`：

```js
case EVENT_REFRESH_RESOURCE:
    return {
        ...previousState,
        refresh: {
            lastReceived: Date.now(),  // Hook 用时间戳判断新事件
            resources: data,           // {"library": ["1"], "plugin": ["x"]}
        },
    }
```

两个 Hook 都通过 `useSelector(state => state.activity.refresh)` 订阅这个状态，当 `lastReceived` 变化时触发刷新逻辑。

---

## 五、配置注入 UI 的方式

### 5.1 页面加载时注入

`server/serve_index.go` 在渲染 `index.html` 时将 `conf.Server` 的部分字段序列化为 JSON 注入到页面：

```go
appConfig := map[string]any{
    "enableDownloads":        conf.Server.EnableDownloads,
    "enableSharing":         conf.Server.EnableSharing,
    "enableFavourites":      conf.Server.EnableFavourites,
    // ... 30+ 个配置项
}
```

这些值在页面刷新前不会变化。用户需要 **刷新浏览器** 才能看到服务器级配置变更的效果。

### 5.2 SSE 事件驱动更新

运行时数据（Library、Plugin 等）的变更通过 SSE 实时推送到前端，不需要刷新页面。

---

## 六、总结：热加载能力矩阵

### 6.1 完整矩阵（含前端链路）

| 设置类别 | 前端入口组件 | 请求方法与路径 | 持久化 | 内存生效 | 前端自动生效 | SSE 广播内容 | 触发副作用 |
|---------|------------|--------------|--------|---------|-------------|-------------|-----------|
| 服务器配置 `conf.Server` | (无，只读) | GET /api/config/ | ✅ 文件 | ❌ 需重启 | ❌ 需刷新 | ❌ 无 | — |
| 编辑音乐库 | LibraryEdit.jsx Save 按钮 | PUT /api/library/{id} | ✅ DB | ✅ 立即 | ✅ | `{"library":["id"]}` | 路径变化时重启监控 + 触发扫描 |
| 新建音乐库 | LibraryCreate.jsx Save 按钮 | POST /api/library | ✅ DB | ✅ 立即 | ✅ | `{"library":["id"]}` | 启动监控 + 触发扫描 |
| 删除音乐库 | DeleteLibraryButton.jsx | DELETE /api/library/{id} | ✅ DB | ✅ 立即 | ✅ | `{"library":["id"]}` | 停止监控 + 触发扫描 + 清理插件权限 |
| 插件配置/权限 | PluginShow.jsx Save 按钮 | PUT /api/plugin/{id} | ✅ DB | ✅ 重载WASM | ✅ | `{"plugin":["id"]}` | unload+load 插件 + 权限门控检查 |
| 插件启用切换 | ToggleEnabledSwitch.jsx | PUT /api/plugin/{id} | ✅ DB | ✅ 重载WASM | ✅ | `{"plugin":["id"]}` | unload+load 插件 |
| 插件文件 `.ndp` 变更（SHA256 变 + 提取成功） | (文件系统事件) | 无 API | ✅ DB | ✅ 重载WASM | ✅ | `{"plugin":["id"]}` | AutoReload 时自动 |
| 插件文件 `.ndp` 变更（SHA256 变 + 提取失败 + 已启用） | (文件系统事件) | 无 API | ✅ DB | ✅ 禁用（unload） | ❌ 需手动刷新 | ❌ 无（潜在 bug） | 插件被禁用但前端无通知 |
| 插件文件 `.ndp` 被删除 | (文件系统事件) | 无 API | ✅ DB | ✅ unload | ✅ | `{"plugin":["id"]}` | AutoReload 时自动 |
| 用户基础信息修改 | UserEdit.jsx Save 按钮 | PUT /api/user/{id} | ✅ DB | ✅ 立即 | ❌ 需手动刷新 | ❌ 无 | — |
| 用户库关联修改 | UserEdit.jsx Save 按钮 | PUT /api/user/{id}/library | ✅ DB | ✅ 立即 | ✅ | `{"user":["id"], "library":["1","3"]}` | 权限校验 + 级联插件权限 |
| 用户删除 | DeleteUserButton.jsx | DELETE /api/user/{id} | ✅ DB | ✅ 立即 | ⚠️ 仅插件自动刷新 | ⚠️ 仅 `{"plugin":["id"]}`（间接） | 级联清理 + UnloadDisabledPlugins |
| 触发扫描 | LibraryScanButton.jsx | GET /rest/startScan | ✅ DB | ✅ 立即 | ✅ | 扫描完成后 `{"*":"*"}` | 全库扫描 + 全局 RefreshResource |

### 6.2 关键不一致性总结

| 模块 | 预期行为 | 实际行为 | 影响 |
|------|---------|---------|------|
| `.ndp` 提取失败 + 已启用 | 应广播 plugin 变更 | ❌ 不广播 | 前端显示"已启用"但实际已禁用，需手动刷新 |
| 用户基础信息修改 | 应广播 user 变更 | ❌ 不广播 | 多管理员协作时数据不一致 |
| 用户删除 | 应广播 user 变更 | ❌ 不广播（仅间接广播 plugin） | 其他管理员仍看到已删除用户 |

### 6.3 核心数据结构一致性对比

**音乐库（Library）** — 一致性最好：
- 增删改 → 持久化 → 副作用（watcher/scanner）→ 广播 `library` 事件 → 所有前端自动刷新
- 代码位置：`core/library.go` Save/Update/Delete 方法均显式调用 `broker.SendBroadcastMessage()`

**插件（Plugin）** — API 触发一致，文件触发不一致：
- REST API 触发的增删改 → `updatePluginSettings()` 或 Enable/Disable → 均调用 `sendPluginRefreshEvent()` ✅
- 文件系统触发 → `updatePluginInDB()` 成功时广播 ✅，但提取失败时不广播 ❌

**用户（User）** — 一致性最差：
- `Save()` / `Update()` 方法完全透传，无任何广播 ❌
- `Delete()` 方法不直接广播 user 事件 ❌
- 只有通过 `libraryService.SetUserLibraries()` 修改库关联时才广播 ✅

---

**核心结论**：Navidrome 的"管理端设置变更热加载"主要体现在 **数据库驱动的运行时数据**（Library、Plugin、User 等），完整链路为：

```
UI 组件交互 (useMutation/useUpdate)
  → react-admin wrapperDataProvider (统一资源路由 + 特殊处理)
    → httpClient (注入 JWT + X-ND-Client-Unique-Id + 拦截 token 刷新)
      → Go Chi Router (JWT 中间件 → adminOnly 中间件 → URL 解析)
        → Repository Wrapper (DB 持久化 + 副作用执行 + SSE 广播)
          ↓
    ┌───── 分歧点：并非所有操作都广播 ─────┐
    │                                       │
    ✅ Library 所有操作 → 都广播          ❌ User 基础信息修改 → 不广播
    ✅ Plugin API 操作 → 都广播           ❌ User 删除 → 不直接广播
    ✅ Plugin 文件变更(成功) → 广播        ❌ Plugin 文件变更(失败) → 不广播
    ✅ User 库关联修改 → 广播             
          ↓
          SSE Broker SendBroadcastMessage
            ↓
          所有在线前端 EventSource
            ↓
          Redux activityReducer → state.activity.refresh.lastReceived
            ↓
    ┌───── 前端 Hook 分发 ─────┐
    │                          │
    useResourceRefresh    useRefreshOnEvents
    (资源局部 getMany)    (自定义回调)
          ↓
          UI 自动重渲染
```

而 **服务器级配置**（`conf.Server`）不支持运行时热加载，变更后必须重启进程。
