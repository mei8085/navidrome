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
    WriteInitialMetrics(ctx context.Context)
    WriteAfterScanMetrics(ctx context.Context, success bool)
    RecordRequest(ctx context.Context, endpoint, method, client string, status int32, elapsed int64)
    RecordPluginRequest(ctx context.Context, plugin, method string, ok bool, elapsed int64)
    GetHandler() http.Handler
}
```

存在两种实现：
- **`metrics`（启用时）**：真正写入 Prometheus 的实现，单例模式
- **`noopMetrics`（禁用时）**：空实现，所有方法为空操作

### 2.3 指标定义与数据来源

所有 Prometheus 指标在 `getPrometheusMetrics` 中一次性注册，定义于 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L109-L197)：

| 指标名 | 类型 | 标签 | 数据来源 | 写入时机 |
|--------|------|------|----------|----------|
| `navidrome_info` | GaugeVec | version | consts.Version | 服务启动时 WriteInitialMetrics |
| `db_model_totals` | GaugeVec | model | 数据库 CountAll() 查询 | 启动时 + 每次扫描后 |
| `media_scan_last` | GaugeVec | success | 当前时间戳 | 每次媒体扫描结束时 |
| `media_scans` | CounterVec | success | 计数器自增 | 每次媒体扫描结束时 |
| `http_request_count` | CounterVec | endpoint, method, client, status | Subsonic API 中间件 | 每个 HTTP 请求完成时 |
| `http_request_latency` | SummaryVec | endpoint, method, client | 请求耗时毫秒数 | 每个 HTTP 请求完成时 |
| `plugin_request_count` | CounterVec | plugin, method, ok | 插件管理器 | 每次插件函数调用完成时 |
| `plugin_request_latency` | SummaryVec | plugin, method | 插件调用耗时毫秒数 | 每次插件函数调用完成时 |

### 2.4 指标采集链路详解

#### 链路 1：HTTP 请求指标（Subsonic API）

**中间件挂载**：在 [subsonic/api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/api.go#L91-L93) 的 `routes()` 中，通过 `recordStats` 中间件在每个请求完成时调用 `RecordRequest()`。

#### 链路 2：媒体扫描指标

**写入时机**：[scanner/controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/scanner/controller.go#L238-L243) 的 `ScanFolders` 方法中，扫描完成时调用 `WriteAfterScanMetrics(ctx, success)`。

#### 链路 3：插件调用指标

**核心函数**：所有插件调用都流经 [plugins/manager_call.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager_call.go#L38-L96) 的 `callPluginFunction` 泛型函数。

##### 完整分支与指标记录对照表

| 代码位置 | 场景 | 是否记录指标 | 记录值(ok) |
|----------|------|--------------|------------|
| 第 44-47 行 | `plugin.instance(ctx)` 创建插件实例失败 | NO - 不记录 | - |
| 第 50-53 行 | `p.FunctionExists(funcName)` 函数不存在 | NO - 不记录 | - |
| 第 55-58 行 | `json.Marshal(input)` 输入序列化失败 | NO - 不记录 | - |
| 第 65-68 行 | `CallWithContext` 错误且 `ctx.Err()!=nil`（Context 取消） | NO - 不记录 | - |
| 第 69 行 | `CallWithContext` 其他错误（非 Context 取消） | YES | false |
| 第 74-78 行 | `exit == notImplementedCode`（第 77 行代码被注释） | NO - 不记录 | - |
| 第 80 行 | `exit != 0` 且不是 `notImplementedCode` | YES | false |
| 第 92 行 | `exit == 0` 且 JSON 反序列化成功 | YES | true |
| 第 92 行 | `exit == 0` 且 JSON 反序列化失败 | YES | false |

##### 对巡检判断的影响

| 影响点 | 代码证据 | 巡检口径 |
|--------|----------|----------|
| 指标覆盖不完整 | 9 条退出路径中仅 4 条调用 RecordPluginRequest | `plugin_request_count` 仅反映部分调用，不能作为总调用量 |
| 失败率分母偏小 | 5 条失败路径不在分母中 | 指标显示的失败率是已记录调用中的失败率，实际失败率无法仅从指标推算 |
| 静默失败无指标 | 5 条路径直接 return 无 RecordPluginRequest | 需通过日志关键词补充监控 |
| NotImplemented 盲区 | 第 77 行被注释掉 | 无法从指标判断插件是否实现预期接口 |
| Context 取消无指标 | 第 65-67 行检测 ctx.Err() 后直接返回 | 请求超时/客户端断开不体现在指标中 |

##### 需日志补充监控的静默场景

| 代码位置 | 日志关键词 | 场景 |
|----------|------------|------|
| 第 46 行 | `failed to create plugin` | WASM 实例创建失败 |
| 第 52 行 | `Plugin function not found` | 插件未导出该函数 |
| 第 57 行 | （无日志） | 输入序列化失败 |
| 第 66 行 | `Plugin call cancelled` | Context 取消 |
| 第 75 行 | `Plugin function not implemented` | 函数存在但未实现 |

### 2.5 Prometheus 端点安全

在 [prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go#L91-L107) 的 `GetHandler()` 中：
- 若配置了 `Prometheus.Password`，使用 `middleware.BasicAuth`，用户名固定为 `consts.PrometheusAuthUser`（值为 `"navidrome"`）

---

## 三、健康检查与状态端点

### 3.1 端点清单

| 端点路径 | 认证 | 用途 | 定义位置 |
|----------|------|------|----------|
| `GET /ping` | 否 | Liveness 探针 | chi 内置 Heartbeat 中间件 |
| `GET /api/keepalive/*` | JWT | 前端会话保活 | Native API Router |
| `GET /api/insights/*` | JWT | Insights 采集器状态 | Native API Router |
| `GET /api/inspect` | Admin+JWT | 数据库调试 | Native API Router（需配置） |
| `GET /metrics` | Basic Auth（可选） | Prometheus 指标抓取 | Metrics Handler |

### 3.2 /ping - Liveness 探针

挂载于 [server.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/server.go#L178)，使用 chi 内置 `Heartbeat`，不触及数据库。

### 3.3 /api/keepalive - 前端会话保活

定义于 [native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L240-L244)，返回固定 JSON，用于前端 JWT token 续期。

### 3.4 /api/insights - Insights 采集状态

定义于 [native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L246-L255)，返回最后一次 Insights 采集的时间戳和成功状态。

### 3.5 /api/inspect - 数据库调试端点

定义于 [native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go#L220-L232)，需 Admin 权限且显式启用 `Inspect.Enabled`。

---

## 四、Insights 遥测采集链路

在 [root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/root.go#L304-L319) 中作为独立 goroutine 启动，默认 30 分钟延迟后开始采集，每 24 小时执行一次。

采集内容包括：版本信息、配置开关、库统计、7 日活跃数据、内存信息、已安装插件版本。

---

## 五、对运维巡检的支撑能力

### 5.1 容器编排场景（Kubernetes/Docker）

| 探针类型 | 推荐端点 | 理由 |
|----------|----------|------|
| Liveness | `GET /ping` | 轻量级，不依赖数据库 |
| Readiness | `GET /ping` 或 `GET /metrics` | `/metrics` 返回说明 HTTP 栈和指标系统已就绪 |
| Startup | `GET /ping` | 同上 |

### 5.2 巡检告警规则（统一版本 - 全文仅此一套）

以下告警规则按指标链路分类，每条规则均标注代码依据和已知盲区。

```yaml
# ====================================================
# 媒体扫描
# 代码依据：scanner/controller.go ScanFolders() 结束后调用
#   WriteAfterScanMetrics(ctx, success)，写入 media_scan_last 和 media_scans
# 已知盲区：无。扫描成功/失败均会记录指标。
# ====================================================

- alert: MediaScanStale
  expr: time() - media_scan_last{success="true"} > 3 * 24 * 3600
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "超过 3 天无成功扫描"

- alert: MediaScanAlwaysFailing
  expr: increase(media_scans{success="false"}[1h]) > 0 and increase(media_scans{success="true"}[1h]) == 0
  for: 30m
  labels:
    severity: critical
  annotations:
    summary: "扫描持续失败，1 小时内无成功记录"

# ====================================================
# HTTP 请求（Subsonic API）
# 代码依据：server/subsonic/middlewares.go recordStats() 在每个请求
#   完成后调用 metrics.RecordRequest()，无论成功失败均记录。
# 已知盲区：仅覆盖 Subsonic API 路由，Native API 和静态资源不记录。
# ====================================================

- alert: HighHTTPErrorRate
  expr: |
    sum(rate(http_request_count{status=~"5.."}[5m]))
    / sum(rate(http_request_count[5m])) > 0.05
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Subsonic API 5xx 错误率超过 5%"

# ====================================================
# 插件调用
# 代码依据：plugins/manager_call.go callPluginFunction() 中，
#   仅 4/9 条退出路径调用 RecordPluginRequest：
#     记录(ok=false): CallWithContext 非取消错误
#     记录(ok=false): exit!=0 且非 notImplemented
#     记录(ok=true):  exit==0 且 JSON 反序列化成功
#     记录(ok=false): exit==0 且 JSON 反序列化失败
#   5 条路径不记录：实例失败/函数不存在/序列化失败/Context取消/NotImplemented
#
# 巡检口径：
#   - plugin_request_count 的分母仅含已记录的调用
#   - 指标中 0% 失败率不代表无任何失败
#   - 调用量突降可能意味着系统级故障进入了不记录指标的分支
# ====================================================

- alert: PluginHighFailureRate
  expr: |
    sum(rate(plugin_request_count{ok="false"}[5m]))
    / sum(rate(plugin_request_count[5m])) > 0.05
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "已记录的插件调用中失败率超过 5%"
    note: "该比率分母不含实例创建失败、Context取消、NotImplemented等不记录指标的场景"

- alert: PluginCallVolumeDrop
  expr: |
    sum(rate(plugin_request_count[10m]))
    < (sum(rate(plugin_request_count[6h] offset 10m)) * 0.3)
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "已记录的插件调用量降至基线的 30% 以下"
    note: "可能原因：插件 WASM 加载失败、输入序列化失败等不记录指标的系统级故障"
    mechanism: "对比当前 10 分钟速率与 6 小时前同窗口速率，降幅超 70% 触发"

# ====================================================
# 日志补充监控（Loki / Promtail 示例）
# 以下场景不写入 Prometheus 指标，需通过日志捕获：
#   count_over_time({job="navidrome"} |= "failed to create plugin" [5m]) > 0
#   count_over_time({job="navidrome"} |= "Plugin function not found" [5m]) > 0
#   count_over_time({job="navidrome"} |= "Plugin call cancelled" [5m]) > 0
#   count_over_time({job="navidrome"} |= "Plugin function not implemented" [5m]) > 0
# ====================================================
```

### 5.3 运维操作接口

| 操作场景 | 接口/方式 | 说明 |
|----------|-----------|------|
| 验证服务存活 | `curl http://host:port/ping` | 返回 200 即正常 |
| 抓取运行指标 | Prometheus scrape `/metrics` | 支持 Basic Auth |
| 检查 Insights 上报状态 | `GET /api/insights` | 需登录，查看最后上报时间 |
| 调试数据库状态 | `GET /api/inspect` | 需 Admin 且配置 `Inspect.Enabled=true` |
| 检查扫描运行状态 | `/api/inspect` 或观察 `media_scan_last` 指标 | - |

---

## 六、核心模块协作关系图

```
cmd/root.go (启动入口)
  |
  +-- CreatePrometheus() --> core/metrics/prometheus.go
  |       |                       |
  |       |                       +-- WriteInitialMetrics() --> 数据库 CountAll
  |       |                       +-- WriteAfterScanMetrics() <-- scanner/controller.go
  |       |                       +-- RecordRequest()        <-- server/subsonic/middlewares.go (recordStats)
  |       |                       |
  |       |                       +-- RecordPluginRequest()  <-- plugins/manager_call.go (callPluginFunction)
  |       |                                                |
  |       |                                                +-- 仅 4/9 分支记录指标
  |       |                                                +-- 5/9 分支静默失败无指标
  |       |
  |       +-- GetHandler() --> MountRouter("/metrics")
  |
  +-- CreateInsights() --> core/metrics/insights.go
  |       |                       |
  |       |                       +-- Run() --> 每 24h collect() + sendInsights()
  |       |                       +-- LastRun() <-- server/nativeapi/native_api.go (/api/insights)
  |       |
  |       +-- startInsightsCollector (goroutine)
  |
  +-- CreateServer() --> server/server.go
          |
          +-- middleware.Heartbeat("/ping")  --> Liveness 探针
          +-- MountRouter(Native API) --> /keepalive, /insights, /inspect
          +-- MountRouter(Subsonic API) --> recordStats 中间件 --> RecordRequest()
```

---

## 七、关键代码文件索引

| 文件 | 职责 |
|------|------|
| [core/metrics/prometheus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/prometheus.go) | Prometheus 指标定义、注册、采集接口实现 |
| [core/metrics/insights.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/core/metrics/insights.go) | Insights 遥测数据采集与上报 |
| [server/server.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/server.go) | HTTP 服务器初始化、/ping 中间件、路由挂载 |
| [server/nativeapi/native_api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/nativeapi/native_api.go) | Native API 路由：/keepalive、/insights、/inspect |
| [server/subsonic/middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/middlewares.go) | Subsonic API 请求指标中间件 recordStats |
| [server/subsonic/api.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/server/subsonic/api.go) | Subsonic API 路由，条件挂载 recordStats |
| [scanner/controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/scanner/controller.go) | 媒体扫描控制器，扫描结束写入指标 |
| [plugins/manager_call.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/plugins/manager_call.go) | 插件调用执行，记录调用指标 |
| [cmd/root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/root.go) | 服务启动入口，条件挂载指标端点、启动 Insights |
| [cmd/wire_injectors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/cmd/wire_injectors.go) | Wire 依赖注入绑定 |
| [conf/configuration.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/conf/configuration.go) | 配置结构定义及默认值 |
| [consts/consts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/40-navidrome/consts/consts.go) | 常量定义 |
