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
| **权限控制** | 根据 manifest 声明过滤可用服务 | `manager_loader.go:319-337` |
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

### 3.3 能力声明与权限联动约束（硬条件）

Navidrome 在加载时执行两轮 manifest 校验，确保能力与权限的一致性。

#### 第一轮：Manifest 解析时校验（`manifest.go:27-43`）

```go
func (m *Manifest) Validate() error {
    // 硬条件1: SubsonicAPI 权限 → 必须声明 Users 权限
    if m.Permissions != nil && m.Permissions.Subsonicapi != nil {
        if m.Permissions.Users == nil {
            return fmt.Errorf("'subsonicapi' permission requires 'users' permission to be declared")
        }
    }
    // 配置 schema 校验...
    return nil
}
```

#### 第二轮：能力检测后校验（`manifest.go:57-83`）

```go
func ValidateWithCapabilities(m *Manifest, capabilities []Capability) error {
    // 硬条件2: Scrobbler 能力 → 必须声明 Users 权限
    if hasCapability(capabilities, CapabilityScrobbler) {
        if m.Permissions == nil || m.Permissions.Users == nil {
            return fmt.Errorf("scrobbler capability requires 'users' permission to be declared in manifest")
        }
    }

    // 硬条件3: Scheduler 权限 → 必须导出 SchedulerCallback 能力
    if m.Permissions != nil && m.Permissions.Scheduler != nil {
        if !hasCapability(capabilities, CapabilityScheduler) {
            return fmt.Errorf("'scheduler' permission requires plugin to export '%s' function", FuncSchedulerCallback)
        }
    }

    // 硬条件4: Taskqueue 权限 → 必须导出 TaskWorker 能力
    if m.Permissions != nil && m.Permissions.Taskqueue != nil {
        if !hasCapability(capabilities, CapabilityTaskWorker) {
            return fmt.Errorf("'taskqueue' permission requires plugin to export '%s' function", FuncTaskWorkerCallback)
        }
    }

    return nil
}
```

#### 能力-权限联动矩阵

| 能力/权限 | 硬约束关系 | 校验阶段 |
|----------|-----------|---------|
| **SubsonicAPI 权限** | → 必须声明 Users 权限 | Manifest 解析时 |
| **Scrobbler 能力** | → 必须声明 Users 权限 | 能力检测后 |
| **Scheduler 权限** | → 必须导出 `nd_scheduler_callback` | 能力检测后 |
| **Taskqueue 权限** | → 必须导出 `nd_task_execute` | 能力检测后 |

### 3.4 宿主调用前后的权限校验链路

#### 3.4.1 加载时权限校验（插件加载阶段）

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

**关键机制：**

1.  只有在 manifest 中声明了对应权限的宿主服务才会被注册到插件实例中
2.  未声明权限的服务，插件完全无法调用（WASM import 解析失败）

#### 3.4.2 运行时权限校验（插件调用宿主函数阶段）

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

### 3.5 越权请求阻断机制

#### 阻断发生在三个层级：

| 层级 | 阻断点 | 阻断方式 |
|-----|--------|----------|
| **加载时阻断** | `manager_loader.go:329-337` | 宿主函数不注册，插件 WASM import 解析失败 |
| **调用前阻断** | 各服务实现层 | 返回明确错误信息，调用失败 |
| **结果过滤** | Users/Library 服务 | 静默过滤掉不允许的数据 |

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

#### 4.1.1 recover 与宿主函数错误返回的边界

**关键澄清**：代码中**只有一个位置**有 panic recover 兜底，宿主函数执行时**没有** panic 捕获。

| 场景 | 是否有 recover | 处理方式 |
|-----|---------------|---------|
| **插件加载阶段** (`manager_loader.go:201-206`) | ✅ 有 | goroutine 级别的 defer/recover，捕获加载过程中的 panic |
| **宿主函数执行阶段** (`host/*_gen.go`) | ❌ 无 | 只有常规的错误检查（`if err != nil`），通过 JSON 返回错误 |
| **插件 WASM 执行阶段** | - | Wazero 运行时捕获 trap，转换为 Go error |

**加载阶段 recover 兜底** (`manager_loader.go:197-237`）

```go
g.Go(func() error {
    start := time.Now()
    log.Debug(ctx, "Loading enabled plugin", "plugin", plugin.ID, "path", plugin.Path)

    // 唯一的 panic recover 兜底
    defer func() {
        if r := recover(); r != nil {
            log.Error(ctx, "Panic while loading plugin", "plugin", plugin.ID, "panic", r)
        }
    }()

    if err := m.loadPluginWithConfig(&plugin); err != nil {
        plugin.LastError = err.Error()
        plugin.Enabled = false
        plugin.UpdatedAt = time.Now()
        repo.Put(&plugin)
        log.Error(ctx, "Failed to load plugin", "plugin", plugin.ID, err)
        return nil
    }
    // ...
    return nil
})
```

**宿主函数错误处理（无 recover）** (`host/config_gen.go:54-83`）

```go
func newConfigGetHostFunction(service ConfigService) extism.HostFunction {
    return extism.NewHostFunctionWithStack(
        "config_get",
        func(ctx context.Context, p *extism.CurrentPlugin, stack []uint64) {
            // 只有常规错误检查，没有 recover
            reqBytes, err := p.ReadBytes(stack[0])
            if err != nil {
                configWriteError(p, stack, err)
                return
            }
            var req ConfigGetRequest
            if err := json.Unmarshal(reqBytes, &req); err != nil {
                configWriteError(p, stack, err)
                return
            }
            // 调用业务服务
            value, exists := service.Get(ctx, req.Key)
            // 返回响应
            resp := ConfigGetResponse{Value: value, Exists: exists}
            configWriteResponse(p, stack, resp)
        },
    )
}
```

> **重要风险提示**：宿主函数业务逻辑（如 `service.Get(ctx, req.Key)`）如果发生 panic，会直接崩溃整个宿主进程，因为没有 recover 兜底。这是当前设计的一个潜在风险点。

#### 4.1.2 WASM Trap 异常

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

#### 4.1.3 插件 Panic 异常

插件内部 panic 会被 WASM 运行时捕获为 trap，转换为 error 返回。

#### 4.1.4 超时异常

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
    
    // 2. 执行 closers 清理（KVStore、Scheduler 等）
    plugin.Close()
    
    // 3. 关闭编译后的插件（带 5 秒宽限期）
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
    │   ├─ WASM Trap → Wazero 捕获 → 返回错误 → 销毁实例 → 下次调用重建
    │   │
    │   ├─ 插件 Panic → WASM trap → 返回错误 → 销毁实例 → 下次调用重建
    │   │
    │   └─ 超时/取消 → context 取消 → 终止 WASM → 返回 DeadlineExceeded → 销毁实例 → 下次调用重建
    │
    └─ 宿主函数调用
        │
        ├─ 权限校验失败 → 返回错误 → 插件侧处理
        │
        ├─ 宿主 Panic → ⚠️ 无 recover → 可能崩溃宿主进程 ⚠️
        │
        └─ 宿主正常执行 → 返回结果
```

---

## 五、插件启用前 Gate 与配置变更处理链路

### 5.1 插件启用前 Gate 检查

在插件启用前，系统会执行权限 Gate 检查，确保必要的配置已完成。

**启用流程** (`manager.go:287-331`）

```go
func (m *Manager) EnablePlugin(ctx context.Context, id string) error {
    // 1. 从数据库获取插件信息
    plugin, err := repo.Get(id)
    
    // 2. 启用前 Gate 检查
    if err := m.checkPermissionGates(plugin); err != nil {
        return err  // Gate 不通过，拒绝启用
    }
    
    // 3. 尝试加载插件
    if err := m.loadPluginWithConfig(plugin); err != nil {
        plugin.LastError = err.Error()
        plugin.UpdatedAt = time.Now()
        _ = repo.Put(plugin)
        return fmt.Errorf("loading plugin: %w", err)
    }
    
    // 4. 更新数据库状态
    plugin.Enabled = true
    plugin.LastError = ""
    plugin.UpdatedAt = time.Now()
    repo.Put(plugin)
    
    return nil
}
```

**Gate 检查逻辑** (`manager.go:590-614`）

```go
func (m *Manager) checkPermissionGates(p *model.Plugin) error {
    manifest, err := readManifest(p.Path)
    
    // Gate 1: Users 权限需要配置用户或 allUsers=true
    if manifest.Permissions != nil && manifest.Permissions.Users != nil {
        if !hasValidUsersConfig(p.Users, p.AllUsers) {
            return fmt.Errorf("users permission requires configuration: select users or enable 'all users' access")
        }
    }
    
    // Gate 2: Library 权限需要配置库或 allLibraries=true
    if manifest.Permissions != nil && manifest.Permissions.Library != nil {
        if !hasValidLibrariesConfig(p.Libraries, p.AllLibraries) {
            return fmt.Errorf("library permission requires configuration: select libraries or enable 'all libraries' access")
        }
    }
    
    return nil
}
```

**配置有效性检查** (`manager.go:616-644`）

```go
func hasValidUsersConfig(usersJSON string, allUsers bool) bool {
    if allUsers {
        return true
    }
    if usersJSON == "" {
        return false
    }
    var users []string
    if err := json.Unmarshal([]byte(usersJSON), &users); err != nil {
        return false
    }
    return len(users) > 0
}

func hasValidLibrariesConfig(librariesJSON string, allLibraries bool) bool {
    if allLibraries {
        return true
    }
    if librariesJSON == "" {
        return false
    }
    var libraries []int
    if err := json.Unmarshal([]byte(librariesJSON), &libraries); err != nil {
        return false
    }
    return len(libraries) > 0
}
```

### 5.2 配置变更后自动禁用与卸载链路

当插件配置变更（如用户/库权限被移除）时，系统会自动检测并禁用不再满足权限条件的插件。

**统一更新入口** (`manager.go:440-510`）

```go
func (m *Manager) updatePluginSettings(ctx context.Context, id string, updateFn func(*model.Plugin)) error {
    plugin, err := repo.Get(id)
    wasEnabled := plugin.Enabled
    
    // 1. 应用更新
    updateFn(plugin)
    plugin.UpdatedAt = time.Now()
    
    // 2. 检查权限是否仍然满足
    shouldDisable := false
    disableReason := ""
    if wasEnabled {
        manifest, err := readManifest(plugin.Path)
        if err == nil && manifest.Permissions != nil {
            // 检查 Users 权限是否仍然有效
            if manifest.Permissions.Users != nil && !hasValidUsersConfig(plugin.Users, plugin.AllUsers) {
                shouldDisable = true
                disableReason = "users permission removal"
            }
            // 检查 Library 权限是否仍然有效
            if manifest.Permissions.Library != nil && !hasValidLibrariesConfig(plugin.Libraries, plugin.AllLibraries) {
                shouldDisable = true
                disableReason = "library permission removal"
            }
        }
    }
    
    // 3. 权限不满足 → 自动禁用
    if shouldDisable {
        m.unloadPlugin(id)
        plugin.Enabled = false
        repo.Put(plugin)
        log.Info(ctx, "Disabled plugin due to "+disableReason, "plugin", id)
        return nil
    }
    
    // 4. 保存配置
    repo.Put(plugin)
    
    // 5. 如果之前是启用状态 → 重新加载
    if wasEnabled {
        m.unloadPlugin(id)
        if err := m.loadPluginWithConfig(plugin); err != nil {
            plugin.LastError = err.Error()
            plugin.Enabled = false
            repo.Put(plugin)
            return fmt.Errorf("reloading plugin: %w", err)
        }
    }
    
    return nil
}
```

### 5.3 用户/库删除后的清理链路

当用户或媒体库被删除时，系统会清理相关插件的权限。

**清理禁用插件** (`manager.go:546-588`）

```go
func (m *Manager) UnloadDisabledPlugins(ctx context.Context) {
    // 1. 获取所有被禁用的插件
    plugins, err := repo.GetAll(model.QueryOptions{
        Filters: squirrel.Eq{"enabled": false},
    })
    
    // 2. 检查每个被禁用的插件是否仍在内存中
    var unloaded []string
    for _, p := range plugins {
        m.mu.RLock()
        _, loaded := m.plugins[p.ID]
        m.mu.RUnlock()
        
        if loaded {
            // 3. 卸载仍在内存中的插件
            if err := m.unloadPlugin(p.ID); err != nil {
                log.Debug(ctx, "Plugin was not loaded", "plugin", p.ID)
            }
            unloaded = append(unloaded, p.ID)
        }
    }
    
    // 4. 发送刷新事件
    if len(unloaded) > 0 {
        m.sendPluginRefreshEvent(ctx, unloaded...)
    }
}
```

### 5.4 完整生命周期链路图

```
用户启用插件
    │
    ▼
EnablePlugin()
    │
    ├─ checkPermissionGates()
    │   ├─ 检查 Users 权限配置
    │   └─ 检查 Library 权限配置
    │
    ├─ loadPluginWithConfig()
    │   ├─ 解析 manifest
    │   ├─ Validate() → 检查 SubsonicAPI→Users 依赖
    │   ├─ 编译 WASM
    │   ├─ detectCapabilities() → 检测导出函数
    │   └─ ValidateWithCapabilities() → 检查能力-权限联动
    │       ├─ Scrobbler → Users 权限
    │       ├─ Scheduler → SchedulerCallback 能力
    │       └─ Taskqueue → TaskWorker 能力
    │
    └─ 更新 DB: Enabled=true


用户/库被删除
    │
    ▼
触发 UnloadDisabledPlugins()
    │
    ├─ 查询 DB 中 Enabled=false 的插件
    ├─ 检查这些插件是否仍在内存中
    └─ 卸载内存中的插件 → unloadPlugin()


配置变更
    │
    ▼
updatePluginSettings()
    │
    ├─ 应用配置更新
    ├─ 检查权限是否仍然满足
    │   ├─ Users 权限是否还有效
    │   └─ Library 权限是否还有效
    │
    ├─ 权限不满足 → 自动禁用 + 卸载
    │
    └─ 权限满足 → 重新加载插件
```

---

## 六、跨语言调用约束

### 6.1 调用协议：JSON 序列化

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

### 6.2 宿主函数注册机制

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

### 6.3 未实现函数的优雅处理

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

## 七、代码生成与多语言支持

### 7.1 ndpgen 工具链

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

### 7.2 插件开发示例

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

## 八、总结与设计评估

### 8.1 设计亮点

1. **清晰的职责分离**：宿主聚焦运行时管理与安全控制，PDK 提供类型安全的跨语言抽象

2. **三层权限防护**：加载时过滤 → 调用前校验 → 调用后过滤，确保最小权限原则

3. **能力-权限硬约束**：通过 `Validate()` 和 `ValidateWithCapabilities()` 两轮校验，确保能力与权限一致性

4. **实例级隔离**：每次调用创建新 WASM 实例，故障影响范围最小化

5. **完整的资源清理**：通过 `closers` 模式统一管理 KVStore、Scheduler、WebSocket 等资源

6. **配置变更自动处理**：用户/库删除或权限变更时自动禁用插件，防止越权访问

### 8.2 潜在风险与改进建议

| 风险点 | 严重程度 | 建议 |
|-------|---------|------|
| **宿主函数无 panic 兜底** | 高 | 在 `ndpgen` 生成的宿主函数包装中添加 defer/recover，防止业务逻辑 panic 崩溃宿主 |
| **加载时 recover 范围过大** | 中 | recover 只捕获了加载 goroutine，但 `loadPluginWithConfig` 内部的错误通过 error 返回，逻辑一致但设计稍显冗余 |
| **无插件调用熔断机制** | 中 | 可考虑添加失败率阈值，连续失败多次后自动熔断插件 |
| **无插件资源使用监控** | 低 | 可考虑添加 WASM 内存使用、CPU 时间的监控与限制 |

### 8.3 关键设计决策汇总

| 决策点 | 选择 | 理由 |
|-------|------|------|
| **序列化协议** | JSON | 跨语言兼容性最好，调试友好 |
| **实例模型** | 每次调用新建实例 | 最大化隔离，简化错误恢复 |
| **权限模型** | manifest 声明 + 运行时校验 | 声明式配置 + 纵深防御 |
| **能力检测** | 扫描 WASM 导出函数 | 无需额外声明，自动发现 |
| **多语言支持** | 代码生成器 ndpgen | 从单一 Go 定义生成多语言绑定，确保一致性 |

Navidrome 插件体系通过多层次的安全与隔离机制，在保证插件灵活性的同时有效控制了风险，是生产级插件系统的优秀实践。
