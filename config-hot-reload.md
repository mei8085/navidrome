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

[configuration.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/conf/configuration.go#L293-L294) 定义全局单例：

```go
var Server = &configOptions{}
```

启动时通过 `conf.Load()` 一次性加载，流程为：

1. `conf.InitConfig()` — 初始化 Viper，绑定环境变量前缀 `ND_`
2. `conf.Load()` — 读取配置文件 → 反序列化到 `conf.Server` → 运行校验 → 执行注册的 Hook

关键代码位于 [configuration.go#L323-L475](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/conf/configuration.go#L323-L475)：

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

[native_api.go#L234-L238](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/nativeapi/native_api.go#L234-L238) 注册了只读的配置端点：

```go
func (api *Router) addConfigRoute(r chi.Router) {
    if conf.Server.DevUIShowConfig {
        r.Get("/config/*", getConfig)
    }
}
```

[config.go#L96-L129](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/nativeapi/config.go#L96-L129) 中 `getConfig` 仅做 GET 读取，序列化 `conf.Server` 并对敏感字段脱敏后返回。**没有 PUT/POST handler，不支持运行时修改**。

### 2.3 配置在运行时的消费方式

大部分代码在 **每次请求时** 直接读取 `conf.Server.*` 字段（如 `conf.Server.EnableDownloads`、`conf.Server.EnableSharing`），属于"引用式"消费，如果 `conf.Server` 被修改，理论上会立即生效。但问题在于 **没有任何运行时机制去修改它**：

- 配置 API 是只读的
- 没有对配置文件的 `fsnotify` 监听
- `SIGHUP` 信号在 [root.go#L108](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/cmd/root.go#L108) 中被注册，但仅用于触发 context 取消（关闭进程），不会重载配置
- `SIGUSR1` 信号（[signaller_unix.go#L15-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/cmd/signaller_unix.go#L15-L40)）只触发音乐库扫描，不涉及配置

### 2.4 部分配置在启动时缓存

一些子系统在启动时读取配置并缓存，即使 `conf.Server` 变化也无法生效：

| 子系统 | 缓存的配置项 | 代码位置 |
|--------|-------------|---------|
| 定时扫描调度 | `Scanner.Schedule` | [root.go#L144-L167](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/cmd/root.go#L144-L167) |
| 定时备份调度 | `Backup.Schedule` | [root.go#L243-L276](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/cmd/root.go#L243-L276) |
| 文件监控防抖间隔 | `Scanner.WatcherWait` | [watcher.go#L50](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/scanner/watcher.go#L50) |
| Scanner 控制器 | `DevExternalScanner` | [controller.go#L37](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/scanner/controller.go#L37) |
| 插件文件监控 | `Plugins.AutoReload` | [manager.go#L164](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/plugins/manager.go#L164) |

### 2.5 `conf.AddHook` 机制

[configuration.go#L716-L718](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/conf/configuration.go#L716-L718) 提供了钩子注册：

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

## 三、运行时数据变更 — 支持热加载

以下设置可通过管理端 API 运行时修改，变更后 **立即对在线客户端生效**。

### 3.1 变更广播基础设施：SSE Broker

[events.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/events/events.go) 和 [sse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/events/sse.go) 实现了基于 Server-Sent Events 的实时推送：

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

SSE 端点挂载在 [server.go#L188-L195](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/server.go#L188-L195)：

```go
if conf.Server.DevActivityPanel {
    r.Handle(path.Join(..., "events"), s.broker)
}
```

前端通过 [eventStream.js](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/ui/src/eventStream.js) 订阅事件：

```js
stream.addEventListener('refreshResource', eventHandler(dispatchFn))
stream.addEventListener('scanStatus', throttledEventHandler(dispatchFn))
stream.addEventListener('nowPlayingCount', eventHandler(dispatchFn))
```

### 3.2 核心事件类型

| 事件类型 | 触发场景 | 数据格式 |
|---------|---------|---------|
| `RefreshResource` | 资源变更后通知 UI 刷新 | `{"library":["1"]}` 或 `{"*":"*"}` |
| `ScanStatus` | 扫描进度更新 | `{"scanning":true,"count":100,...}` |
| `NowPlayingCount` | 正在播放数量变化 | `{"count":3}` |
| `ServerStart` | 新客户端连接时推送 | `{"startTime":"...","version":"..."}` |
| `KeepAlive` | 每 15 秒心跳 | `{"ts":1700000000}` |

### 3.3 各类设置的热加载路径

---

#### 3.3.1 音乐库（Library）设置

**变更入口**：`PUT /api/library/{id}`

**完整路径**：

```
API 请求
  │
  ▼
native_api.go: Router.routes() → adminOnlyMiddleware
  │
  ▼
library.go: libraryRepositoryWrapper.Update()
  │
  ├─ LibraryRepository.Put(lib)        ← 持久化到 DB
  │
  ├─ 检查 path 是否变化
  │   ├─ 是 → watcher.Watch(ctx, lib)  ← 重启文件监控
  │   │      scanner.ScanAll()          ← 异步触发扫描
  │   └─ 否 → 跳过
  │
  └─ broker.SendBroadcastMessage(
       &RefreshResource{}.With("library", id)
     )                                  ← 广播 SSE 事件
  │
  ▼
前端 eventStream.js 收到 refreshResource 事件
  → Redux dispatch(processEvent)
  → re-fetch 数据
```

关键代码：[library.go#L194-L240](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/core/library.go#L194-L240)

新建和删除库的流程类似，删除时额外调用 `pluginManager.UnloadDisabledPlugins()` 清理因权限丢失而被自动禁用的插件。

---

#### 3.3.2 插件（Plugin）设置

**变更入口**：`PUT /api/plugin/{id}`

**完整路径**：

```
API 请求 (PluginUpdateRequest)
  │
  ▼
plugin.go: updatePlugin()
  │
  ├─ Config 变更?
  │   └─ ValidatePluginConfig() → UpdatePluginConfig()
  │       │
  │       ▼ manager.go: updatePluginSettings()
  │         ├─ repo.Get(id)
  │         ├─ updateFn(plugin)           ← 修改内存中的 plugin 对象
  │         ├─ repo.Put(plugin)           ← 持久化到 DB
  │         ├─ unloadPlugin(id)           ← 卸载 WASM 插件实例
  │         └─ loadPluginWithConfig(plugin) ← 用新配置重新加载
  │
  ├─ Users/Libraries 变更?
  │   └─ UpdatePluginUsers() / UpdatePluginLibraries()
  │       └─ 同样走 updatePluginSettings()
  │       └─ 检查权限门控，不满足则自动 DisablePlugin()
  │
  ├─ Enabled 变更?
  │   ├─ EnablePlugin()
  │   │   ├─ loadPluginWithConfig()      ← 加载 WASM 到内存
  │   │   ├─ repo.Put(plugin)            ← 标记 enabled
  │   │   └─ sendPluginRefreshEvent()    ← 广播 SSE
  │   └─ DisablePlugin()
  │       ├─ unloadPlugin()              ← 从内存卸载 WASM
  │       ├─ repo.Put(plugin)            ← 标记 disabled
  │       └─ sendPluginRefreshEvent()    ← 广播 SSE
  │
  └─ 返回更新后的 plugin JSON
```

关键代码：
- [plugin.go#L68-L167](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/nativeapi/plugin.go#L68-L167)
- [manager.go#L440-L510](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/plugins/manager.go#L440-L510)

**插件热加载的真正核心**：`unloadPlugin()` + `loadPluginWithConfig()` 组合操作：
1. 从 `Manager.plugins` map 中移除
2. 调用 `plugin.Close()` 执行插件定义的清理函数
3. 关闭 `wazero.CompiledPlugin` 释放 WASM 编译缓存
4. 用新配置重新编译和实例化 WASM 插件

**文件级自动重载**：当 `Plugins.AutoReload = true` 时，[manager_watcher.go](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/plugins/manager_watcher.go) 监控 `.ndp` 插件包文件变化：
- 2 秒防抖（`debounceDuration`）
- 检测文件存在性而非事件类型（处理 Rename/临时文件等边界情况）
- SHA256 比对判断是否真正变更

---

#### 3.3.3 用户（User）设置

**变更入口**：`PUT /api/user/{id}`, `DELETE /api/user/{id}`

用户删除的完整路径：

```
DELETE /api/user/{id}
  │
  ▼
user.go: userRepositoryWrapper.Delete()
  │
  ├─ UserRepository.Delete(id)           ← DB 级联清理
  │   └─ cleanupPluginUserReferences()   ← 清理插件用户权限引用
  │
  └─ pluginManager.UnloadDisabledPlugins()
      │                                  ← 检查因用户删除导致权限不满足的插件
      ├─ 查询 DB 中所有 enabled=false 的插件
      ├─ 对仍在内存中的执行 unloadPlugin()
      └─ sendPluginRefreshEvent()        ← 广播 SSE
```

关键代码：[user.go#L63-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/core/user.go#L63-L75)

---

#### 3.3.4 扫描（Scan）触发

扫描完成后，若检测到变更，通过 SSE 广播 `RefreshResource` 通知所有客户端刷新：

```go
// scanner/controller.go#L233-L236
if s.changesDetected {
    s.broker.SendBroadcastMessage(ctx, &events.RefreshResource{})
}
```

这里的 `RefreshResource` 不携带具体资源 ID（等效于 `{"*":"*"}`），意味着前端会刷新所有数据。

---

## 四、配置注入 UI 的方式

### 4.1 页面加载时注入

[serve_index.go#L42-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/39-navidrome/server/serve_index.go#L42-L83) 在渲染 `index.html` 时将 `conf.Server` 的部分字段序列化为 JSON 注入到页面：

```go
appConfig := map[string]any{
    "enableDownloads":        conf.Server.EnableDownloads,
    "enableSharing":         conf.Server.EnableSharing,
    "enableFavourites":      conf.Server.EnableFavourites,
    // ... 30+ 个配置项
}
```

这些值在页面刷新前不会变化。用户需要 **刷新浏览器** 才能看到服务器级配置变更的效果。

### 4.2 SSE 事件驱动更新

运行时数据（Library、Plugin 等）的变更通过 SSE 实时推送到前端，不需要刷新页面。

---

## 五、总结：热加载能力矩阵

| 设置类别 | 修改方式 | 持久化 | 内存生效 | 前端生效 | 触发副作用 |
|---------|---------|--------|---------|---------|-----------|
| 服务器配置 `conf.Server` | 改配置文件/环境变量 + 重启 | ✅ 文件 | ❌ 需重启 | ❌ 需刷新 | — |
| 音乐库 Library | REST API | ✅ DB | ✅ 立即 | ✅ SSE | 重启监控 + 触发扫描 |
| 插件 Plugin | REST API | ✅ DB | ✅ 重载WASM | ✅ SSE | unload+load 插件 |
| 插件文件 `.ndp` | 文件系统变更 | ✅ DB | ✅ 重载WASM | ✅ SSE | AutoReload 时自动 |
| 用户 User | REST API | ✅ DB | ✅ 立即 | 需要时 | 清理插件权限 |
| 扫描触发 | REST API / 信号 | ✅ DB | ✅ 立即 | ✅ SSE | 全局 RefreshResource |

**核心结论**：Navidrome 的"管理端设置变更热加载"主要体现在 **数据库驱动的运行时数据**（Library、Plugin、User 等），通过 SSE 广播 + 直接操作内存对象实现实时生效。而 **服务器级配置**（`conf.Server`）不支持运行时热加载，变更后必须重启进程。
