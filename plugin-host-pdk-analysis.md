# Navidrome 插件体系架构深度分析报告

## 一、整体架构概述

Navidrome 采用 **Extism + Wazero** 技术栈构建 WebAssembly 插件体系，实现了宿主与多语言 PDK 的协作。插件以 `.ndp` 包格式分发（包含编译后的 WASM 字节码和 manifest 声明文件），运行在完全隔离的沙箱环境中。

```
┌──────────────────────────────────────────────────────────────────┐
│                      Navidrome 宿主进程                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                     Plugin Manager                          │  │
│  │  ┌──────────┐  ┌────────────┐  ┌────────────────────────┐  │  │
│  │  │  生命周期  │  │  权限控制   │  │   能力检测(Capability)  │  │  │
│  │  └──────────┘  └────────────┘  └────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│         │                           │                             │
│         ▼                           ▼                             │
│  ┌─────────────┐             ┌─────────────┐                      │
│  │ Extism SDK  │             │ 宿主服务集  │──┐                   │
│  └─────────────┘             └─────────────┘  │                   │
│         │                           │         │                   │
│         ▼                           ▼         ▼                   │
│  ┌─────────────────────────────────────────────────────┐          │
│  │                    Wazero WASM 运行时                │          │
│  │  ┌──────────┐  ┌──────────┐          ┌──────────┐   │          │
│  │  │ 插件实例1 │  │ 插件实例2 │  ...     │ 插件实例N │   │          │
│  │  │ (内存隔离)│  │ (内存隔离)│          │ (内存隔离)│   │          │
│  │  └──────────┘  └──────────┘          └──────────┘   │          │
│  └─────────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、宿主与 PDK 职责边界

### 2.1 宿主（Host）核心职责

| 职责模块 | 核心功能 | 关键文件 |
|---------|---------|---------|
| **插件生命周期管理** | 加载、卸载、启用、禁用、热重载 | `manager.go`, `manager_loader.go` |
| **WASM 运行时环境** | 基于 Wazero + Extism 提供沙箱执行 | `manager_plugin.go` |
| **宿主服务暴露** | 向插件注册可调用的宿主函数 | `host/*_gen.go`, `manager_loader.go:40-148` |
| **权限控制** | 根据 manifest 声明过滤可用服务 | `manager_loader.go:329-337` |
| **能力检测** | 自动识别插件导出函数映射的能力 | `capabilities.go:24-36` |
| **错误隔离与恢复** | panic 捕获、超时控制、实例隔离 | `manager_loader.go:201-206`, `manager_call.go` |
| **资源管理** | 编译缓存、文件系统挂载、内存清理 | `manager.go:126-134`, `manager.go:512-544` |
| **指标监控** | 插件调用成功率、耗时统计 | `manager_call.go:69,81,92` |

#### 宿主服务注册表（`manager_loader.go:40-148`）

```go
var hostServices = []hostServiceEntry{
    {name: "Config", hasPermission: always, create: newConfigService},
    {name: "SubsonicAPI", hasPermission: needSubsonicPerm, create: newSubsonicAPIService},
    {name: "Scheduler", hasPermission: needSchedulerPerm, create: newSchedulerService},
    {name: "WebSocket", hasPermission: needWebsocketPerm, create: newWebSocketService},
    {name: "Artwork", hasPermission: needArtworkPerm, create: newArtworkService},
    {name: "Cache", hasPermission: needCachePerm, create: newCacheService},
    {name: "Library", hasPermission: needLibraryPerm, create: newLibraryService},
    {name: "KVStore", hasPermission: needKvstorePerm, create: newKVStoreService},
    {name: "Users", hasPermission: needUsersPerm, create: newUsersService},
    {name: "HTTP", hasPermission: needHttpPerm, create: newHTTPService},
    {name: "Task", hasPermission: needTaskqueuePerm, create: newTaskQueueService},
}
```

### 2.2 PDK（Plugin Development Kit）核心职责

PDK 为插件开发者提供跨语言的类型安全封装，目前支持 **Go、Python、Rust** 三种语言。

| PDK 模块 | 核心功能 | 关键路径 |
|---------|---------|---------|
| **核心 PDK** | 内存管理、配置读写、变量存储、HTTP 请求、日志 | `pdk/go/pdk/pdk.go` |
| **宿主服务客户端** | 封装宿主函数调用，处理 JSON 序列化 | `pdk/go/host/nd_host_*.go` |
| **能力接口** | 定义插件可实现的能力接口与导出函数 | `pdk/go/metadata/metadata.go`, `pdk/go/lifecycle/lifecycle.go` |
| **测试支持** | 非 WASM 环境下的 Mock/Stub 实现 | `pdk/go/pdk/pdk_stub.go`, `*_stub.go` |

#### PDK 双构建模式

PDK 采用构建标签（build tag）实现同一源码在不同环境下的差异化编译：

| 构建标签 | 模式 | 实现方式 |
|---------|------|---------|
| `wasip1` | WASM 插件构建 | 直接委托给 extism/go-pdk，零开销 |
| `!wasip1` | 本地测试构建 | 提供 testify mock 实现，支持单元测试 |

**WASM 模式** (`pdk/go/pdk/pdk.go:7-13`):

```go
//go:build wasip1
package pdk
import extism "github.com/extism/go-pdk"

func Allocate(length int) Memory { return extism.Allocate(length) }
func Input() []byte { return extism.Input() }
func Output(data []byte) { extism.Output(data) }
```

**测试模式** (`pdk/go/pdk/pdk_stub.go:7-26`):

```go
//go:build !wasip1
package pdk
import "github.com/stretchr/testify/mock"

var PDKMock = &mockPDK{}
func Allocate(length int) Memory {
    args := PDKMock.Called(length)
    return args.Get(0).(Memory)
}
```

---

## 三、能力声明与权限校验链路

### 3.1 能力（Capability）系统

能力是插件功能性的抽象，通过检测插件导出的 WASM 函数自动识别。

#### 能力定义与注册

```go
// capabilities.go:5-7
type Capability string

// 能力与导出函数的映射
var capabilityFunctions = map[Capability][]string{}

// 注册能力（在各 adapter 的 init() 中调用）
func registerCapability(cap Capability, functions ...string) {
    capabilityFunctions[cap] = functions
}
```

**能力自动检测** (`capabilities.go:24-36`):

```go
func detectCapabilities(plugin functionExistsChecker) []Capability {
    var capabilities []Capability
    for cap, functions := range capabilityFunctions {
        if slices.ContainsFunc(functions, plugin.FunctionExists) {
            capabilities = append(capabilities, cap)
        }
    }
    return capabilities
}
```

#### 内置能力清单

| 能力名称 | 导出函数前缀 | 用途 |
|---------|-------------|------|
| `Lifecycle` | `nd_on_init` | 插件初始化钩子 |
| `MetadataAgent` | `nd_get_artist_*`, `nd_get_album_*` | 元数据提供者 |
| `Scrobbler` | `nd_scrobble` | 播放记录上报 |
| `Lyrics` | `nd_get_lyrics` | 歌词提供者 |
| `SonicSimilarity` | `nd_get_similar_songs` | 音频相似度计算 |
| `TaskWorker` | `nd_task_execute` | 后台任务处理 |
| `SchedulerCallback` | `nd_scheduler_callback` | 定时任务回调 |
| `WebSocketCallback` | `nd_websocket_on_*` | WebSocket 事件回调 |

### 3.2 权限（Permissions）系统

权限在 `manifest.json` 中声明，控制插件可访问的宿主服务范围。

```json
{
  "name": "My Plugin",
  "version": "1.0.0",
  "author": "John Doe",
  "permissions": {
    "http": { "requiredHosts": ["api.example.com"] },
    "library": { "filesystem": true },
    "kvstore": { "maxSize": "10MB" }
  }
}
```

**权限类型** (`manifest_gen.go:162-192`):

- `artwork` - 封面图访问
- `cache` - 缓存读写
- `http` - 外发 HTTP 请求（需声明 `requiredHosts`）
- `kvstore` - 持久化键值存储（可限制 `maxSize`）
- `library` - 媒体库元数据/文件系统访问
- `scheduler` - 定时任务调度
- `subsonicapi` - Subsonic API 调用
- `taskqueue` - 后台任务队列
- `users` - 用户信息访问
- `websocket` - WebSocket 连接

### 3.3 宿主调用前后的权限校验链路

#### 3.3.1 加载时权限校验（插件加载阶段）

**阶段一：宿主服务过滤** (`manager_loader.go:319-337`）

```go
svcCtx := &serviceContext{
    pluginName:       p.ID,
    manager:          m,
    permissions:      pkg.Manifest.Permissions,
    config:           pluginConfig,
    allowedUsers:     allowedUsers,
    allUsers:         p.AllUsers,
    allowedLibraries: allowedLibraries,
    allLibraries:     p.AllLibraries,
}

// 遍历所有宿主服务，根据权限过滤
for _, entry := range hostServices {
    if entry.hasPermission(pkg.Manifest.Permissions) {
        funcs, closer := entry.create(svcCtx)
        hostFunctions = append(hostFunctions, funcs...)
        if closer != nil {
            closers = append(closers, closer)
        }
    }
}
```

**关键机制：

1.  只有在 manifest 中声明了对应权限的宿主服务才会被注册到插件实例中
2.  未声明权限的服务，插件完全无法调用（WASM import 解析失败

#### 3.3.2 运行时权限校验（插件调用宿主函数阶段）

每个宿主服务在实现层都有独立的权限校验逻辑：

**Library 服务权限校验** (`host_library.go:34-55`）

```go
func (s *libraryServiceImpl) GetLibrary(ctx context.Context, id int32) (*host.Library, error) {
    // 调用前校验：检查库是否在允许列表中
    if !s.isLibraryAccessible(int(id)) {
        return nil, fmt.Errorf("library not accessible: library ID %d is not in the allowed list", id)
    }
    // ... 实际业务逻辑
}

func (s *libraryServiceImpl) isLibraryAccessible(id int) bool {
    if s.allLibraries {
        return true
    }
    _, ok := s.libraryIDMap[id]
    return ok
}
```

**Users 服务权限校验** (`host_users.go:25-51`）

```go
func (s *usersServiceImpl) GetUsers(ctx context.Context) ([]host.User, error) {
    users, err := s.ds.User(ctx).GetAll()
    // 调用后过滤：只返回允许的用户
    allowedMap := make(map[string]bool)
    for _, id := range s.allowedUsers {
        allowedMap[id] = true
    }
    var result []host.User
    for _, u := range users {
        if s.allUsers || allowedMap[u.ID] {
            result = append(result, host.User{...})
        }
    }
    return result, nil
}
```

**SubsonicAPI 服务权限校验** (`host_subsonicapi.go:137-163`）

```go
func (s *subsonicAPIServiceImpl) checkPermissions(ctx context.Context, username string) error {
    if s.allUsers {
        return nil
    }
    usr, err := s.ds.User(ctx).FindByUsername(username)
    if err != nil {
        return err
    }
    // 校验用户 ID 是否在允许列表中
    if _, ok := s.userIDMap[usr.ID]; !ok {
        return fmt.Errorf("user %s is not authorized for this plugin", username)
    }
    return nil
}
```

**HTTP 服务权限校验** (`host_httpclient.go:143-162`）

```go
func (s *httpServiceImpl) validateHost(ctx context.Context, hostStr string) error {
    hostname := extractHostname(hostStr)
    if len(s.requiredHosts) > 0 {
        if !s.isHostAllowed(hostname) {
            return fmt.Errorf("host %q is not allowed", hostStr)
        }
        return nil
    }
    // 无明确允许列表时，阻止私有IP/回环地址 防止 SSRF
    if isPrivateOrLoopback(hostname) {
        return fmt.Errorf("host %q is not allowed: private/loopback addresses require explicit requiredHosts in manifest", hostStr)
    }
    return nil
}
```

**WebSocket 服务权限校验** (`host_websocket.go:75-90`）

```go
func (s *webSocketServiceImpl) Connect(ctx context.Context, urlStr string, ...) (string, error) {
    parsedURL, err := url.Parse(urlStr)
    // 校验协议
    if parsedURL.Scheme != "ws" && parsedURL.Scheme != "wss" {
        return "", fmt.Errorf("invalid URL scheme: must be ws:// or wss://")
    }
    // 校验主机是否在允许列表中
    if !s.isHostAllowed(parsedURL.Host) {
        return "", fmt.Errorf("host %q is not allowed", parsedURL.Host)
    }
    // ...
}
```

**KVStore 服务权限校验** (`host_kvstore.go:118-133`）

```go
func (s *kvstoreServiceImpl) checkStorageLimit(ctx context.Context, delta int64) error {
    used, err := s.storageUsed(ctx)
    newTotal := used + delta
    if newTotal > s.maxSize {
        return fmt.Errorf("storage limit exceeded: would use %s of %s allowed",
            humanize.Bytes(uint64(newTotal)), humanize.Bytes(uint64(s.maxSize)))
    }
    return nil
}
```

**TaskQueue 服务权限校验** (`host_taskqueue.go:178-196`）

```go
func (s *taskQueueServiceImpl) clampConcurrency(ctx context.Context, name string, config *host.QueueConfig) error {
    var allocated int32
    for _, qs := range s.queues {
        allocated += qs.config.Concurrency
    }
    available := s.maxConcurrency - allocated
    if available <= 0 {
        return fmt.Errorf("concurrency budget exhausted (%d/%d allocated)", allocated, s.maxConcurrency)
    }
    if config.Concurrency > available {
        config.Concurrency = available
    }
    return nil
}
```

### 3.4 越权请求阻断机制

#### 阻断发生在三个层级：

| 层级 | 阻断点 | 阻断方式 |
|-----|--------|----------|
| **加载时阻断** | `manager_loader.go:329-337` | 宿主函数不注册，插件无法 import 失败 |
| **调用前阻断** | 各服务实现层 | 返回明确错误信息，调用失败 |
| **结果过滤** | Users/Library 服务 | 过滤掉不允许的数据 |

#### 越权访问示例（Library 服务）：

```
插件调用 host_library.go:GetLibrary(999)
    │
    ▼
isLibraryAccessible(999) → 返回 false
    │
    ▼
返回错误: "library not accessible: library ID 999 is not in the allowed list"
    │
    ▼
host/library_gen.go:libraryWriteError() 写入错误响应
    │
    ▼
插件 PDK 收到 JSON 错误响应
```

#### 越权访问示例（HTTP 服务）：

```
插件调用 host_httpclient.go:Send("http://192.168.1.1/api")
    │
    ▼
validateHost("192.168.1.1")
    │
    ▼
isPrivateOrLoopback("192.168.1.1") → 返回 true（私有地址）
    │
    ▼
返回错误: "host 192.168.1.1 is not allowed: private/loopback addresses require explicit requiredHosts in manifest"
```

---

## 四、插件异常降级与恢复路径

### 4.1 异常类型与处理机制

#### 4.1.1 WASM Trap 异常

WASM trap 是 WASM 运行时错误，包括：

- 内存访问越界
- 除零错误
- 非法指令
- 栈溢出

**处理方式**：Wazero 运行时捕获 trap，转换为 Go error 返回，不会崩溃宿主进程。

**代码路径**：

```go
// manager_call.go:61-72
exit, output, err := p.CallWithContext(ctx, funcName, inputBytes)
if err != nil {
    // WASM trap 被 Extism 封装为 error 返回
    if ctx.Err() != nil {
        return result, ctx.Err()
    }
    return result, fmt.Errorf("plugin call failed: %w", err)
}
```

#### 4.1.2 插件 Panic 异常

**加载阶段 Panic 捕获** (`manager_loader.go:201-206`）

```go
g.Go(func() error {
    defer func() {
        if r := recover(); r != nil {
            log.Error(ctx, "Panic while loading plugin", "plugin", plugin.ID, "panic", r)
        }
    }()
    return m.loadPluginWithConfig(&plugin)
})
```

**宿主函数执行 Panic 捕获** (`host/config_gen.go:54-83`）

```go
func newConfigGetHostFunction(service ConfigService) extism.HostFunction {
    return extism.NewHostFunctionWithStack(
        "config_get",
        func(ctx context.Context, p *extism.CurrentPlugin, stack []uint64) {
            // 所有错误被捕获并通过 JSON 返回
            reqBytes, err := p.ReadBytes(stack[0])
            if err != nil {
                configWriteError(p, stack, err)
                return
            }
            // ... 业务逻辑
        },
    )
}
```

#### 4.1.3 超时异常

**默认超时**：30 秒（`manager.go:31`）

```go
const defaultTimeout = 30 * time.Second
```

**超时处理路径** (`manager_call.go:38-72`）

```go
func callPluginFunction[I any, O any](ctx context.Context, plugin *plugin, funcName string, input I) (O, error) {
    // 1. 创建可取消的插件实例
    p, err := plugin.instance(ctx)
    defer p.Close(ctx)
    
    // 2. 调用时传入 context，支持超时和取消
    exit, output, err := p.CallWithContext(ctx, funcName, inputBytes)
    
    // 3. 优先返回上下文错误
    if ctx.Err() != nil {
        return result, ctx.Err()
    }
}
```

**Context 传播机制** (`manager_plugin.go:30-39`）

```go
func (p *plugin) instance(ctx context.Context) (*extism.Plugin, error) {
    instance, err := p.compiled.Instance(ctx, extism.PluginInstanceConfig{
        ModuleConfig: wazero.NewModuleConfig().
            WithSysWalltime().
            WithRandSource(rand.Reader),
    })
    // wazero.WithCloseOnContextDone(true) 确保 context 取消时立即终止 WASM 执行
    return instance, nil
}
```

### 4.2 降级策略

| 异常场景 | 处理策略 |
|---------|---------|
| **插件加载失败** | 标记 `Enabled=false`，记录错误到数据库，不影响其他插件 |
| **函数调用崩溃** | 本次调用返回错误，instance 被销毁，下次调用创建新实例 |
| **超时未响应** | context 超时终止执行，返回 `DeadlineExceeded` 错误 |
| **依赖用户/库被删除** | 自动检测权限不满足，卸载插件并禁用 |

**加载失败处理** (`manager_loader.go:208-217`）

```go
if err := m.loadPluginWithConfig(&plugin); err != nil {
    plugin.LastError = err.Error()
    plugin.Enabled = false  // 自动禁用
    plugin.UpdatedAt = time.Now()
    repo.Put(&plugin)       // 持久化错误状态
    log.Error(ctx, "Failed to load plugin", "plugin", plugin.ID, err)
    return nil              // 不返回错误，继续加载其他插件
}
```

### 4.3 恢复路径

#### 4.3.1 实例级恢复：每次调用创建新实例

```go
// manager_plugin.go:28-39
func (p *plugin) instance(ctx context.Context) (*extism.Plugin, error) {
    instance, err := p.compiled.Instance(ctx, extism.PluginInstanceConfig{...})
    return instance, nil
}
```

**关键特性**：

- 每次函数调用创建独立的 WASM 实例
- 调用结束即销毁，无状态污染
- 单个调用失败不影响后续调用

#### 4.3.2 资源清理机制

插件持有资源通过 closers 数组统一管理：

```go
// manager_plugin.go:14-25
type plugin struct {
    closers []io.Closer  // 清理函数列表
}

func (p *plugin) Close() error {
    var errs []error
    for _, f := range p.closers {
        if err := f.Close(); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}
```

**卸载流程** (`manager.go:512-544`）

```go
func (m *Manager) unloadPlugin(name string) error {
    // 1. 从注册表移除
    delete(m.plugins, name)
    
    // 2. 执行 closers 清理（KVStore、Scheduler 等
    plugin.Close()
    
    // 3. 关闭编译后的插件（带 5 秒宽限期
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    plugin.compiled.Close(ctx)
    
    runtime.GC() // 提示 GC 回收 WASM 内存
}
```

#### 4.3.3 TaskQueue 任务恢复

```go
// host_taskqueue.go:234-241
// 重启时重置挂起的任务
_, err = s.db.ExecContext(ctx, `
    UPDATE tasks SET status = ?, updated_at = ? WHERE queue_name = ? AND status = ?
`, taskStatusPending, now, name, taskStatusRunning)
```

**关闭时恢复** (`host_taskqueue.go:581-588`）

```go
// Close 时将运行中的任务标记为 pending，下次启动时恢复
if _, err := s.db.Exec(`UPDATE tasks SET status = ?, updated_at = ? WHERE status = ?`, taskStatusPending, now, taskStatusRunning); err != nil {
    log.Error("Failed to reset running tasks on shutdown", "plugin", s.pluginName, err)
}
```

#### 4.3.4 WebSocket 连接恢复

```go
// host_websocket.go:198-229
// 插件卸载时关闭所有连接
func (s *webSocketServiceImpl) Close() error {
    for connID, wsConn := range connections {
        wsConn.closeMu.Lock()
        wsConn.isClosed = true
        wsConn.closeMu.Unlock()
        closeMsg := websocket.FormatCloseMessage(websocket.CloseGoingAway, "plugin unloaded")
        wsConn.conn.WriteControl(websocket.CloseMessage, closeMsg, time.Now().Add(2*time.Second))
        wsConn.conn.Close()
        close(wsConn.done)
        s.invokeOnClose(ctx, connID, websocket.CloseGoingAway, "plugin unloaded")
    }
    return nil
}
```

#### 4.3.5 Scheduler 任务清理

```go
// host_scheduler.go:147-164
func (s *schedulerServiceImpl) Close() error {
    for scheduleID, entry := range schedules {
        if entry.timer != nil {
            entry.timer.Stop()
        } else {
            s.scheduler.Remove(entry.entryID)
        }
    }
    return nil
}
```

#### 4.3.6 KVStore 清理

```go
// host_kvstore.go:369-375
func (s *kvstoreServiceImpl) Close() error {
    s.cancel()
    s.wg.Wait()
    return s.db.Close()
}
```

#### 4.3.7 Cache 清理

```go
// host_cache.go:143-150
func (s *cacheServiceImpl) Close() error {
    s.cache.Stop()
    s.cache.DeleteAll()
    return nil
}
```

### 4.4 完整异常处理流程图

```
插件函数调用
    │
    ├─ 创建新的 WASM 实例
    │
    ├─ 调用插件函数
    │   │
    │   ├─ 正常执行 → 返回结果 → 销毁实例 → 成功
    │   │
    │   ├─ WASM Trap → Extism 捕获 → 返回错误 → 销毁实例 → 下次调用重建
    │   │
    │   ├─ 插件 Panic → Extism 捕获 → 返回错误 → 销毁实例 → 下次调用重建
    │   │
    │   └─ 超时/取消 → context 取消 → 终止 WASM → 返回 DeadlineExceeded → 销毁实例 → 下次调用重建
    │
    └─ 宿主函数调用
        │
        ├─ 权限校验失败 → 返回错误 → 插件侧处理
        │
        └─ 宿主 Panic → 捕获并序列化为 JSON 错误 → 插件侧处理
        │
        └─ 宿主正常执行 → 返回结果
```

---

## 五、跨语言调用约束

### 5.1 调用协议：JSON 序列化

宿主与插件之间的所有交互统一使用 **JSON** 作为序列化格式，确保跨语言兼容性。

**宿主 → 插件 调用流程** (`manager_call.go:34-96`）

```
1. 宿主创建 plugin instance (每次调用独立实例)
2. 将输入参数序列化为 JSON
3. 通过 extism.CallWithContext 调用插件导出函数
4. 读取返回的 JSON 字节并反序列化
5. 关闭 instance，释放资源
```

**插件 → 宿主 调用流程** (`pdk/go/host/nd_host_config.go:57-90`）

```go
func ConfigGet(key string) (string, bool) {
    // 1. 序列化请求为 JSON
    req := configGetRequest{Key: key}
    reqBytes, _ := json.Marshal(req)
    reqMem := pdk.AllocateBytes(reqBytes)
    defer reqMem.Free()
    
    // 2. 通过 WASM import 调用宿主函数
    responsePtr := config_get(reqMem.Offset())
    
    // 3. 读取并解析响应
    responseMem := pdk.FindMemory(responsePtr)
    var response configGetResponse
    json.Unmarshal(responseMem.ReadBytes(), &response)
    return response.Value, response.Exists
}
```

### 5.2 宿主函数注册机制

宿主函数通过代码生成器 `ndpgen` 从 Go 接口定义自动生成。

**接口定义** (`host/config.go`）

```go
//nd:hostservice name=Config
type ConfigService interface {
    //nd:hostfunc
    Get(ctx context.Context, key string) (value string, exists bool)
}
```

**生成的宿主端注册代码** (`host/config_gen.go:44-52`）

```go
func RegisterConfigHostFunctions(service ConfigService) []extism.HostFunction {
    return []extism.HostFunction{
        newConfigGetHostFunction(service),
        newConfigGetIntHostFunction(service),
        newConfigKeysHostFunction(service),
    }
}
```

### 5.3 未实现函数的优雅处理

插件无需实现能力的所有方法，通过约定的返回码 `0xFFFFFFFE` 标识未实现：

```go
// manager_call.go:17-20
const notImplementedCode uint32 = 0xFFFFFFFE

// 调用时检测
if exit == notImplementedCode {
    return result, fmt.Errorf("%w: %s", errNotImplemented, funcName)
}
```

插件端 PDK 自动处理 (`pdk/go/lifecycle/lifecycle.go:46-58`）

```go
//go:wasmexport nd_on_init
func _NdOnInit() int32 {
    if initImpl == nil {
        return NotImplementedCode // -2, 宿主识别为未实现
    }
    if err := initImpl(); err != nil {
        pdk.SetError(err)
        return -1 // 错误
    }
    return 0 // 成功
}
```

---

## 六、代码生成与多语言支持

### 6.1 ndpgen 工具链

`ndpgen` 是 Navidrome 自研的 PDK 代码生成器，从 Go 接口定义生成多语言绑定：

```
Go 接口定义 (//nd:hostservice, //nd:capability)
         │
         ▼
    ndpgen 解析器
         │
         ├─────────┬─────────┐
         ▼         ▼         ▼
    Go PDK     Python PDK   Rust PDK
```

**支持的生成目标** (`cmd/ndpgen/internal/generator.go`）

- `GenerateHost` - 宿主端函数注册代码（`*_gen.go`）
- `GenerateClientGo` - Go 插件客户端
- `GenerateClientPython` - Python 插件客户端
- `GenerateClientRust` - Rust 插件客户端
- `GenerateCapabilityGo/Python/Rust` - 能力接口包装
- `GeneratePDKGo` - 核心 PDK 包装

### 6.2 插件开发示例

```go
// examples/minimal/main.go
package main

import (
    "github.com/navidrome/navidrome/plugins/pdk/go/metadata"
)

type minimalPlugin struct{}

func init() {
    metadata.Register(&minimalPlugin{})
}

func (p *minimalPlugin) GetArtistBiography(input metadata.ArtistRequest) (*metadata.ArtistBiographyResponse, error) {
    return &metadata.ArtistBiographyResponse{
        Biography: "This is a placeholder biography for " + input.Name + ".",
    }, nil
}

func main() {}
```

编译命令：

```bash
tinygo build -o minimal.wasm -target wasip1 -buildmode=c-shared .
```

---

## 七、总结

Navidrome 插件体系设计体现了清晰的职责分离与健壮的隔离机制：

1. **宿主职责**：聚焦于运行时管理、安全控制、资源隔离，通过声明式权限模型精确控制插件能力边界

2. **PDK 职责**：提供类型安全的跨语言抽象，封装底层 WASM 交互细节，让开发者专注于业务逻辑

3. **权限校验链路**：从加载时过滤 → 调用前校验 → 调用后过滤，三层防护

4. **异常恢复路径**：实例级隔离、自动重试、持久化状态恢复，构建完整容错体系

5. **故障隔离**：从 WASM 内存沙箱、超时控制、Panic 恢复到故障降级，构建了完整的容错体系

这种设计既保证了插件的灵活性与可扩展性，又通过多层次的安全与隔离机制，有效控制了插件引入的风险，是生产级插件系统的优秀实践。
