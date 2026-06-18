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

#### 链路 3：插件调用指标

**接口抽象**：为避免循环依赖，插件包在 [plugins/manager.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager.go#L41-L45) 定义了独立的 `PluginMetricsRecorder` 接口，在 Wire 注入时绑定到 `metrics.Metrics`（见 [wire_injectors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/wire_injectors.go#L54)）。

**写入逻辑**：[plugins/manager_call.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager_call.go) 的 `callFunction` 在三种分支都会记录：
- 第 69 行：context 取消或插件执行错误 → `ok=false`
- 第 80 行：插件退出码非零（非 notImplemented） → `ok=false`
- 第 92 行：正常执行完成（含 JSON 反序列化成功与否） → `ok=(err==nil)`

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
