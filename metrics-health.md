# Navidrome 指标采集与健康检查接口协作脉络

## 一、整体架构概览

Navidrome 的可观测性体系由两条独立但协作的链路组成：

1. **Prometheus 指标采集链路**：面向运维监控系统，暴露结构化的量化指标（数据库计数、HTTP 请求统计、扫描记录、插件调用等）
2. **健康检查与状态端点链路**：面向负载均衡、K8s liveness/readiness 探针以及前端轮询，提供轻量级存活/状态信号
3. **Insights 遥测采集链路**：面向项目开发者，匿名上报运行时统计数据（可禁用）

三条链路共享部分数据源（如数据库计数、扫描状态），但通过独立的代码路径和配置开关控制。

---

## 五、对运维巡检的支撑能力

### 5.2 巡检告警规则（统一版本）

以下告警规则按指标链路分类，每条规则均标注代码依据和已知盲区，全文仅此一套。

``yaml
# 媒体扫描
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

# HTTP 请求（Subsonic API）
- alert: HighHTTPErrorRate
  expr: |
    sum(rate(http_request_count{status=~"5.."}[5m]))
    / sum(rate(http_request_count[5m])) > 0.05
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Subsonic API 5xx 错误率超过 5%"

# 插件调用 - 失败率
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

# 插件调用 - 调用量突降（检测静默失败）
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
``

（完整文档其余章节已通过 Write 工具写入，此处为修正验证的告警规则核心部分）
