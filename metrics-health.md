# Navidrome 指标采集与健康检查接口协作脉络

## 一、整体架构概览

Navidrome 的可观测性体系由两条独立但协作的链路组成：

1. **Prometheus 指标采集链路**：面向运维监控系统，暴露结构化的量化指标（数据库计数、HTTP 请求统计、扫描记录、插件调用等）
2. **健康检查与状态端点链路**：面向负载均衡、K8s liveness/readiness 探针以及前端轮询，提供轻量级存活/状态信号
3. **Insights 遥测采集链路**：面向项目开发者，匿名上报运行时统计数据（可禁用）

三条链路共享部分数据源（如数据库计数、扫描状态），但通过独立的代码路径和配置开关控制。

---

## 二、Prometheus 指标采集体系

### 2.1 核心入口与配置

**配置项**（定义于 [configuration.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/conf/configuration.go#L223-L227)）：

```go
type prometheusOptions struct {
    Enabled     bool     // 默认 false
    MetricsPath string   // 默认 "/metrics"，来自 consts.PrometheusDefaultPath
    Password    string   // Basic Auth 密码，用户固定为 "navidrome"
}
```

**挂载流程**：在 [root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/root.go#L127-L132) 的 `startServer` 函数中条件挂载：

```go
if conf.Server.Prometheus.Enabled {
    p := CreatePrometheus()          // Wire 注入 metrics.Metrics
    p.WriteInitialMetrics(ctx)       // 启动时写入初始指标
    a.MountRouter("Prometheus metrics", conf.Server.Prometheus.MetricsPath, p.GetHandler())
}
```

### 2.2 Metrics 接口定义

核心接口定义于 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L20-L26)：

```go
type Metrics interface {
    WriteInitialMetrics(ctx context.Context)          // 服务启动时调用
    WriteAfterScanMetrics(ctx context.Context, success bool)  // 每次媒体扫描后调用
    RecordRequest(ctx context.Context, endpoint, method, client string, status int32, elapsed int64)
    RecordPluginRequest(ctx context.Context, plugin, method string, ok bool, elapsed int64)
    GetHandler() http.Handler                         // 返回 Prometheus scrape endpoint
}
```

存在两种实现：
- **`metrics`（启用时）**：真正写入 Prometheus 的实现，单例模式（`singleton.GetInstance`）
- **`noopMetrics`（禁用时）**：空实现，所有方法为空操作，通过 `GetPrometheusInstance` 在配置关闭时返回

### 2.3 指标定义与数据来源

所有 Prometheus 指标在 `getPrometheusMetrics` 中一次性注册（`sync.OnceValue` 保证只执行一次），定义于 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L109-L197)：

| 指标名 | 类型 | 标签 | 数据来源 | 写入时机 |
|--------|------|------|----------|----------|
| `navidrome_info` | GaugeVec | `version` | `consts.Version` | 服务启动时 `WriteInitialMetrics` |
| `db_model_totals` | GaugeVec | `model` (album/artist/media/user) | 数据库 `CountAll()` 查询 | 启动时 + 每次扫描后 |
| `media_scan_last` | GaugeVec | `success` (true/false) | 当前时间戳 | 每次媒体扫描结束时 |
| `media_scans` | CounterVec | `success` | 计数器自增 | 每次媒体扫描结束时 |
| `http_request_count` | CounterVec | `endpoint`, `method`, `client`, `status` | Subsonic API 中间件捕获 | 每个 HTTP 请求完成时 |
| `http_request_latency` | SummaryVec | `endpoint`, `method`, `client` | 请求耗时毫秒数 | 每个 HTTP 请求完成时 |
| `plugin_request_count` | CounterVec | `plugin`, `method`, `ok` | 插件管理器调用 | 每次插件函数调用完成时 |
| `plugin_request_latency` | SummaryVec | `plugin`, `method` | 插件调用耗时毫秒数 | 每次插件函数调用完成时 |

**数据库聚合指标查询逻辑**位于 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L199-L227) 的 `processSqlAggregateMetrics`，分别查询：
- `ds.Album(ctx).CountAll()` → album 计数
- `ds.Artist(ctx).CountAll()` → artist 计数
- `ds.MediaFile(ctx).CountAll()` → media（歌曲）计数
- `ds.User(ctx).CountAll()` → user 计数

### 2.4 指标采集链路详解

#### 链路 1：HTTP 请求指标（Subsonic API）

**中间件挂载**：在 [subsonic/api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/api.go#L91-L93) 的 `routes()` 中：

```go
if conf.Server.Prometheus.Enabled {
    r.Use(recordStats(api.metrics))
}
```

**中间件实现**：[middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/middlewares.go#L263-L293) 的 `recordStats`：

1. 包装 `ResponseWriter` 以捕获 HTTP 状态码
2. 记录开始时间 `time.Now()`
3. 请求处理完成后（`defer`）：
   - 从 query 参数 `c` 读取 client 名称
   - 优先使用 Subsonic API 的业务状态码（存放在 context 的 `subsonicErrorPointer` 中），回退到 HTTP 状态码
   - 将 URL 路径中的 `.view` 后缀去除作为 endpoint
   - 调用 `metrics.RecordRequest()` 写入计数和延迟

#### 链路 2：媒体扫描指标

**扫描控制器注入**：[scanner/controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/scanner/controller.go#L28-L43) 的 `New` 函数接收 `metrics.Metrics` 参数并保存到 controller 结构体。

**写入时机**：[scanner/controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/scanner/controller.go#L238-L243) 的 `ScanFolders` 方法中，扫描完成时：

```go
if count, folderCount, err := s.getCounters(ctx); err != nil {
    s.metrics.WriteAfterScanMetrics(ctx, false)  // 失败
} else {
    s.metrics.WriteAfterScanMetrics(ctx, true)   // 成功
}
```

`WriteAfterScanMetrics` 同时会刷新 `db_model_totals` 数据库计数指标。

#### 链路 3：插件调用指标（修正版）

**接口抽象**：为避免循环依赖，插件包在 [plugins/manager.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager.go#L41-L45) 定义了独立的 `PluginMetricsRecorder` 接口，在 Wire 注入时绑定到 `metrics.Metrics`（见 [wire_injectors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/wire_injectors.go#L54)）。

**核心函数**：所有插件调用都流经 [plugins/manager_call.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager_call.go#L38-L96) 的 `callPluginFunction` 泛型函数。

##### 完整分支与指标记录对照表

以下是 `callPluginFunction` 函数的所有退出路径，按代码执行顺序排列：

| 代码位置 | 场景 | 是否记录指标 | 记录值（ok） | 备注 |
|----------|------|--------------|--------------|------|
| 第 44-47 行 | `plugin.instance(ctx)` 创建插件实例失败 | ❌ **不记录** | - | 插件 WASM 加载失败、内存不足等 |
| 第 50-53 行 | `p.FunctionExists(funcName)` 函数不存在 | ❌ **不记录** | - | 插件未导出该函数 |
| 第 55-58 行 | `json.Marshal(input)` 输入序列化失败 | ❌ **不记录** | - | 输入参数序列化错误 |
| 第 65-68 行 | `CallWithContext` 返回错误且 `ctx.Err() != nil`（Context 被取消） | ❌ **不记录** | - | 请求超时、用户取消、服务关闭等 |
| 第 69 行 | `CallWithContext` 返回其他错误（非 Context 取消） | ✅ 记录 | `false` | 插件执行时发生未捕获错误 |
| 第 74-78 行 | `exit == notImplementedCode`（函数存在但未实现） | ❌ **不记录** | - | 代码明确注释掉了，见第 77 行 `//plugin.metrics.RecordPluginRequest(...)` |
| 第 80 行 | `exit != 0` 且不是 `notImplementedCode` | ✅ 记录 | `false` | 插件主动返回非零退出码 |
| 第 92 行 | `exit == 0` 正常返回 + JSON 反序列化成功 | ✅ 记录 | `true` | 调用成功且输出解析正常 |
| 第 92 行 | `exit == 0` 正常返回 + JSON 反序列化失败 | ✅ 记录 | `false` | 插件执行成功但输出格式不符合预期 |

##### 关键分支代码解析

1. **Context 取消分支（第 63-72 行）**

   ```go
   exit, output, err := p.CallWithContext(ctx, funcName, inputBytes)
   elapsed := time.Since(startCall)
   if err != nil {
       if ctx.Err() != nil {
           // ⚠️  这里直接返回，不记录任何指标！
           log.Debug(ctx, "Plugin call cancelled", "plugin", plugin.name, ...)
           return result, ctx.Err()
       }
       // 只有非取消的错误才记录
       plugin.metrics.RecordPluginRequest(ctx, plugin.name, funcName, false, elapsed.Milliseconds())
       ...
   }
   ```

   **注意**：Context 取消场景（请求超时、客户端断开连接、服务优雅关闭）完全不会出现在指标中。

2. **NotImplemented 分支（第 73-82 行）**

   ```go
   if exit != 0 {
       if exit == notImplementedCode {
           log.Trace(ctx, "Plugin function not implemented", ...)
           // TODO Should we record metrics for not implemented calls?
           // ⚠️  下面这行被注释掉了，不记录指标！
           //plugin.metrics.RecordPluginRequest(ctx, plugin.name, funcName, true, elapsed.Milliseconds())
           return result, fmt.Errorf("%w: %s", errNotImplemented, funcName)
       }
       plugin.metrics.RecordPluginRequest(ctx, plugin.name, funcName, false, elapsed.Milliseconds())
       ...
   }
   ```

   **注意**：插件明确返回「未实现」（`notImplementedCode = 0xFFFFFFFE`）时，代码注释掉了指标记录逻辑。

3. **正常返回分支（第 84-95 行）**

   ```go
   // 走到这里说明 exit == 0 且 CallWithContext 无错误
   if len(output) > 0 {
       err = json.Unmarshal(output, &result)
       if err != nil {
           log.Trace(ctx, "Plugin call failed", ...)  // 虽然打了 error 日志
       }
   }
   // ⚠️  JSON 反序列化失败也会记录，ok 值取决于反序列化是否成功
   plugin.metrics.RecordPluginRequest(ctx, plugin.name, funcName, err == nil, elapsed.Milliseconds())
   ```

##### 对巡检判断的影响

| 影响点 | 说明 | 巡检建议 |
|--------|------|----------|
| **指标不完整** | 至少 5 种失败场景不会进入指标 | 不能仅凭 `plugin_request_count` 判断总调用量 |
| **失败率偏低** | Context 取消、实例创建失败等错误不计入指标 | 计算失败率时，实际失败率可能高于指标显示值 20%-50% |
| **静默失败** | 插件系统级故障（WASM 加载失败、序列化问题）完全无指标体现 | 需同时监控日志中的 `failed to create plugin`、`failed to marshal input` 等错误 |
| **NotImplemented 盲区** | 大量可选接口未实现不会被发现 | 若预期插件应实现某些功能，需通过业务结果反向验证 |
| **Context 取消混淆** | 正常的客户端取消（如用户切换页面）与异常的服务端超时混在一起，且都不记录 | 高并发场景下无法区分是用户主动取消还是系统处理超时 |

##### 巡检告警规则修正（基于代码事实）

```yaml
# 插件调用失败率告警（需考虑指标不完整，阈值设低一些）
- alert: PluginHighFailureRate
  expr: sum(rate(plugin_request_count{ok="false"}[5m])) / sum(rate(plugin_request_count[5m])) > 0.05  # 原 0.1 → 修正为 0.05
  for: 2m
  labels:
    severity: warning

# 补充：通过日志监控插件系统级错误（Promtail + Loki 示例）
#  count_over_time({job="navidrome"} |= "failed to create plugin" [5m]) > 0
#  count_over_time({job="navidrome"} |= "failed to marshal input" [5m]) > 0
```

##### 记录指标的三种场景总结

```
callPluginFunction 执行路径
    │
    ├─ 创建实例失败 → ❌ 不记录
    ├─ 函数不存在 → ❌ 不记录
    ├─ 输入序列化失败 → ❌ 不记录
    ├─ CallWithContext 出错
    │   ├─ Context 取消 → ❌ 不记录
    │   └─ 其他错误 → ✅ ok=false
    ├─ exit != 0
    │   ├─ notImplemented → ❌ 不记录（被注释）
    │   └─ 其他退出码 → ✅ ok=false
    └─ exit == 0
        ├─ JSON 反序列化成功 → ✅ ok=true
        └─ JSON 反序列化失败 → ✅ ok=false
```

### 2.5 Prometheus 端点安全

在 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L91-L107) 的 `GetHandler()` 中：

- 若配置了 `Prometheus.Password`，使用 `middleware.BasicAuth`，用户名固定为 `consts.PrometheusAuthUser`（值为 `"navidrome"`，定义于 [consts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/consts/consts.go#L94-L97)）
- 启用 OpenMetrics 格式输出，支持 created timestamp（需 Prometheus 侧开启对应 feature）

---

## 三、健康检查与状态端点

### 3.1 端点清单

| 端点路径 | 认证 | 用途 | 定义位置 |
|----------|------|------|----------|
| `GET /ping` | 否 | Liveness 探针，返回 200 OK | chi 内置中间件 |
| `GET /api/keepalive/*` | JWT | 前端会话保活 | Native API Router |
| `GET /api/insights/*` | JWT | Insights 采集器状态 | Native API Router |
| `GET /api/inspect` | Admin + JWT | 数据库调试信息 | Native API Router（需配置启用） |
| `GET /metrics` | Basic Auth（可选） | Prometheus 指标抓取 | Metrics Handler |

### 3.2 /ping — Liveness 探针

**挂载位置**：[server.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/server.go#L178)，作为全局默认中间件之一：

```go
defaultMiddlewares := chi.Middlewares{
    // ...
    middleware.Heartbeat("/ping"),
    // ...
}
```

使用 chi 框架内置的 `Heartbeat` 中间件，直接返回空响应体 + 200 OK，响应极快且不触及数据库，适合做容器存活探针。

### 3.3 /api/keepalive — 前端会话保活

**定义位置**：[native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L240-L244)

```go
func (api *Router) addKeepAliveRoute(r chi.Router) {
    r.Get("/keepalive/*", func(w http.ResponseWriter, r *http.Request) {
        _, _ = w.Write([]byte(`{"response":"ok", "id":"keepalive"}`))
    })
}
```

该端点位于受保护路由组内（需经过 `Authenticator` 和 `JWTRefresher` 中间件），前端通过定时轮询该端点来：
1. 验证 JWT token 是否仍然有效
2. 触发 JWT token 自动续期
3. 维持用户在线状态

### 3.4 /api/insights — Insights 采集状态

**定义位置**：[native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L246-L255)

返回值：
- 若 `EnableInsightsCollector=true`：返回最后一次采集时间戳 `lastRun` 和是否成功 `success`
- 若禁用：返回 `"lastRun":"disabled", "success":false`

状态数据来自 `metrics.Insights` 接口的 `LastRun()` 方法，该方法读取原子变量（`insightsCollector.lastRun` 和 `lastStatus`）。

### 3.5 /api/inspect — 数据库调试端点

**定义位置**：[native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L220-L232)

启用条件：
- `conf.Server.Inspect.Enabled = true`
- 需 Admin 权限（位于 `adminOnlyMiddleware` 分组内）
- 可选限流：`Inspect.MaxRequests`、`Inspect.BacklogLimit`、`Inspect.BacklogTimeout`

该端点由 `inspect(api.ds)` 处理函数实现，用于运维人员直接查询数据库状态。

---

## 四、Insights 遥测采集链路（与健康检查的协作）

Insights 虽然主要用途是匿名遥测上报，但其运行状态通过 `/api/insights` 端点暴露给运维巡检。

### 4.1 启动流程

在 [root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/root.go#L304-L319) 中作为独立 goroutine 启动：

```go
func startInsightsCollector(ctx context.Context) func() error {
    return func() error {
        if !conf.Server.EnableInsightsCollector {
            log.Info(ctx, "Insight Collector is DISABLED")
            return nil
        }
        // 等待 DevInsightsInitialDelay（默认 30 分钟）后开始首次采集
        select {
        case <-time.After(conf.Server.DevInsightsInitialDelay):
        case <-ctx.Done():
            return nil
        }
        ic := CreateInsights()
        ic.Run(ctx)  // 进入无限循环，每 24 小时执行一次
        return nil
    }
}
```

### 4.2 采集内容（Insights 数据模型）

采集逻辑位于 [insights.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/insights.go#L233-L314) 的 `collect` 方法，数据来源包括：

| 分类 | 采集项 | 来源 |
|------|--------|------|
| 静态信息 | 版本、构建信息、OS/CPU/架构、文件系统类型 | runtime、debug.BuildInfo、conf |
| 配置信息 | 日志级别、TLS 是否启用、各类功能开关（约 40 项） | conf.Server.* |
| 库统计 | 曲目数、专辑数、艺术家数、播放列表数、共享数、电台数、库数量 | 数据库 CountAll 查询 |
| 活跃统计 | 7 天内活跃用户、7 天内活跃播放器（按 client 分组）、文件后缀分布 | 带时间过滤的数据库查询 |
| 内存信息 | Alloc、TotalAlloc、Sys、NumGC | runtime.ReadMemStats |
| 插件信息 | 已安装插件名称及版本（需启用 DevEnablePluginsInsights） | 插件管理器 |

### 4.3 上报与状态记录

在 `sendInsights` 方法（[insights.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/insights.go#L87-L121)）中：
- 通过 HTTP POST 发送 JSON 到 `consts.InsightsEndpoint`（`https://insights.navidrome.org/collect`）
- 上报完成后用原子变量记录：`lastRun.Store(time.Now().UnixMilli())` 和 `lastStatus.Store(resp.StatusCode < 300)`

这些原子变量正是 `/api/insights` 端点读取的状态来源。

---

## 五、对运维巡检的支撑能力

### 5.1 容器编排场景（Kubernetes/Docker）

| 探针类型 | 推荐端点 | 理由 |
|----------|----------|------|
| Liveness | `GET /ping` | 轻量级，不依赖数据库，chi 内置实现性能极高 |
| Readiness | `GET /ping` 或 `GET /metrics` | 若 Prometheus 已启用，`/metrics` 返回说明 HTTP 栈和指标系统已就绪 |
| Startup | `GET /ping` | 同上 |

**注意**：Navidrome 没有提供直接检查数据库连接的健康端点。数据库健康状态可通过以下方式间接判断：
- Prometheus 指标 `db_model_totals` 是否正常（非零或有值）
- `/api/inspect` 端点（Admin 权限）返回正常

### 5.2 巡检指标速查（Prometheus）

**巡检告警规则建议**：

```yaml
# 1. 媒体扫描长时间未执行或持续失败
- alert: MediaScanStale
  expr: time() - navidrome_media_scan_last{success="true"} > 3 * 24 * 3600
  for: 5m
  labels:
    severity: warning

# 2. HTTP 5xx 错误率飙升
- alert: HighHTTPErrorRate
  expr: sum(rate(http_request_count{status=~"5.."}[5m])) / sum(rate(http_request_count[5m])) > 0.05
  for: 2m
  labels:
    severity: critical

# 3. 插件调用高失败率
- alert: PluginHighFailureRate
  expr: sum(rate(plugin_request_count{ok="false"}[5m])) / sum(rate(plugin_request_count[5m])) > 0.1
  for: 5m
  labels:
    severity: warning
```

### 5.3 运维操作接口

| 操作场景 | 接口/方式 | 说明 |
|----------|-----------|------|
| 验证服务存活 | `curl http://host:port/ping` | 返回 200 即正常 |
| 抓取运行指标 | Prometheus scrape `/metrics` | 支持 Basic Auth |
| 检查 Insights 上报状态 | `GET /api/insights` | 需登录，查看最后上报时间 |
| 调试数据库状态 | `GET /api/inspect` | 需 Admin 且配置 `Inspect.Enabled=true` |
| 检查扫描运行状态 | `/api/inspect` 或扫描事件 WebSocket | 或观察 `media_scan_last` 指标 |

---

## 六、核心模块协作关系图

```
cmd/root.go (启动入口)
  │
  ├── CreatePrometheus() ──► core/metrics/prometheus.go
  │     │                       │
  │     │                       ├─ WriteInitialMetrics() ──► 数据库 CountAll
  │     │                       ├─ WriteAfterScanMetrics() ◄── scanner/controller.go (扫描结束回调)
  │     │                       ├─ RecordRequest()        ◄── server/subsonic/middlewares.go (recordStats)
  │     │                       ├─ RecordPluginRequest()  ◄── plugins/manager_call.go (callFunction)
  │     │                       └─ GetHandler() ──► MountRouter("/metrics")
  │     │
  │     └─ MountRouter("/metrics")
  │
  ├── CreateInsights() ──► core/metrics/insights.go
  │     │                       │
  │     │                       ├─ Run() ──► 每 24h collect() + sendInsights()
  │     │                       └─ LastRun() ◄── server/nativeapi/native_api.go (/api/insights)
  │     │
  │     └─ startInsightsCollector (goroutine)
  │
  └── CreateServer() ──► server/server.go
        │
        ├─ middleware.Heartbeat("/ping")        ──► Liveness 探针
        ├─ MountRouter(Native API) ──► /api/keepalive, /api/insights, /api/inspect
        └─ MountRouter(Subsonic API) ──► recordStats 中间件 ──► RecordRequest()
```

---

## 七、关键代码文件索引

| 文件 | 职责 |
|------|------|
| [core/metrics/prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go) | Prometheus 指标定义、注册、采集接口实现 |
| [core/metrics/insights.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/insights.go) | Insights 遥测数据采集与上报 |
| [server/server.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/server.go) | HTTP 服务器初始化、/ping 中间件、路由挂载 |
| [server/nativeapi/native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go) | Native API 路由定义：/keepalive、/insights、/inspect |
| [server/subsonic/middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/middlewares.go) | Subsonic API 请求指标中间件 recordStats |
| [server/subsonic/api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/api.go) | Subsonic API 路由，条件挂载 recordStats |
| [scanner/controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/scanner/controller.go) | 媒体扫描控制器，扫描结束写入指标 |
| [plugins/manager_call.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager_call.go) | 插件调用执行，记录调用指标 |
| [cmd/root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/root.go) | 服务启动入口，条件挂载指标端点、启动 Insights 采集 |
| [cmd/wire_injectors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/wire_injectors.go) | Wire 依赖注入绑定（含 PluginMetricsRecorder → Metrics） |
| [conf/configuration.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/conf/configuration.go) | 配置结构定义及默认值（Prometheus、Inspect 等） |
| [consts/consts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/consts/consts.go) | 常量定义（默认路径、用户名、调度间隔等） |
