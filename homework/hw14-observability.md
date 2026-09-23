# 作业 14：可观测性——Metrics、Logs 与 Traces

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W14 |
| 发放日期 | 2027-01-04 |
| 交付日期 | 2027-01-10 |
| 预计工时 | 20 小时 |
| 难度 | 挑战 |
| 对应讲义 | L16 |
| 前置作业 | hw13 |

## 1. 背景与目标

hw12 让服务能互相发现，hw13 让它们通过消息队列异步协作。现在系统的失败模式已经从「函数返回错误」变成「某一个环节悄悄变慢、积压、或部分失败」，靠日志逐个服务翻已经不够。本作业为两个服务补齐可观测性三支柱：**Metrics 回答有没有问题，Traces 回答慢在哪一跳，Logs 回答这一次到底发生了什么**。

目标：

1. 本地用一份 `compose.yaml` 起 `Prometheus` + `Grafana` + `OTel Collector`，一条命令得到可查询的指标与链路。
2. 指标覆盖请求、延迟、饱和度、依赖与业务五个层面，且标签基数受控、可解释。
3. 至少 4 条告警规则能在本地人为触发并观察到触发过程。
4. HTTP 与 gRPC 之间的 trace 上下文不断链，关键内部操作有手写 span。
5. 日志里的 `trace_id` 与追踪系统里的同一次请求能互相跳转。

## 2. 需求（必须项）

**M1 本地可观测性编排。** `compose.yaml` 中增加 `prometheus`、`grafana`、`otel-collector` 三个 service，配置文件放仓库内并挂载。镜像与配置项以各自官方文档为准，不得写死不确定的镜像标签含义。README 给出「打开哪个地址能看到什么」的接线说明。

**M2 指标暴露。** 用 `github.com/prometheus/client_golang`（模块路径，以官方最新稳定版为准，见 <https://pkg.go.dev/github.com/prometheus/client_golang>）暴露 `/metrics`。指标必须显式注册，禁止依赖隐式全局注册表；测试中使用独立注册表，避免测试之间互相污染。

**M3 请求指标。** HTTP 与 gRPC 请求总数按方法、路由、状态码分类；`route` 标签必须取路由模板而不是原始 `URL.Path`。给出至少一个反例，说明取原始路径会让时间序列数量如何增长。

**M4 延迟直方图。** 延迟用直方图记录，分桶边界必须写出依据（例如依据 SLO 阈值与观测到的分布确定），并说明为什么不用平均值与 Summary。

**M5 饱和度指标。** 暴露在途请求数、连接池占用、队列长度等饱和度信号；这些是 Gauge，禁止用 Counter 代替。

**M6 依赖指标。** 对数据库、缓存、MQ 三类依赖分别记录调用总数、失败数与耗时，标签区分依赖名与操作类型；调用方超时与对端返回错误要能区分开。

**M7 业务指标。** 至少两个业务指标，例如文章发布数与消息消费失败数。业务指标必须能从指标名直接对应到业务健康度，标签做归类处理，不得直接用业务主键。

**M8 标签基数控制。** README 中明确列出「禁止进入指标标签」的清单并给出理由，至少覆盖：用户标识、文章标识、原始 URL、错误全文、不受控的调用方自定义头。给出改造前后时间序列数量的对比数据。

**M9 告警规则。** 至少 4 条 PromQL 告警规则，分别覆盖错误率、P99 延迟、消费积压、依赖不可用。每条都要写出阈值推导理由、`for` 的取值理由，以及为什么不用平均值告警。规则文件纳入版本控制。

**M10 Grafana 看板。** 交付看板 JSON 或等价的配置化看板，至少包含服务概览（QPS / 错误率 / 延迟分位 / 饱和度）、依赖视图、业务视图三组面板。README 用表格说明每个面板「回答什么问题」，超过 12 个面板要拆分为多个看板。

**M11 Traces 接入。** 接入 `go.opentelemetry.io/otel` 与 `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp`（模块路径，均以官方最新稳定版为准）。具体构造函数与配置项名称以官方文档为准，见 <https://opentelemetry.io/docs/languages/go/>。

**M12 上下文传播。** 实现 HTTP 入站、HTTP 出站、gRPC 出站、gRPC 入站四个边界的提取与注入，保证「HTTP 网关 → `post-service` → `user-service`」是一条完整链路。传播格式用 W3C Trace Context；漏掉任一方向都要在 README 中指出会导致什么现象。

**M13 手写 span。** 为关键内部操作手写 span：数据库查询、缓存读写、MQ 投递。span 命名使用业务语义，`client` / `server` / `producer` / `consumer` 的配对要正确，不能全部标成 `internal`。

**M14 日志与链路关联。** `log/slog` 输出的每条请求日志都带 `trace_id` 与 `span_id`；从日志能跳到链路，从链路能跳回日志。可用 Go 1.25 的 `slog.GroupAttrs` 组织字段，或多路输出时用 Go 1.26 新增的 `log/slog.NewMultiHandler`（注意它会调用所有 handler）。

**M15 采样策略。** 明确说明采样比率与「错误与慢请求如何被保留」，并解释为什么不能只靠极低比率的头部采样。采样决策位置（SDK 还是 Collector）要写明。

**M16 测试与文档。** 至少两项测试：指标中间件单测（请求经过后计数与直方图有增量，且用了独立注册表）；trace 上下文传播单测（注入后提取出的 trace 标识与注入的一致）。README 含架构图与「一次请求的三支柱对照」：同一次请求的指标、日志、链路分别在哪看。

## 3. 需求（加分项）

- **B1** 用 Collector 的处理器实现尾部采样，让错误与慢请求接近全量保留，并给出对存储量的估算。
- **B2** 把告警规则本地跑通一次并保存触发与恢复的截图或原始输出。
- **B3** 用 `slog.Record.Source`（Go 1.25 新增）在日志中标注调用位置，并说明它对日志体积的影响。
- **B4** 给 `/metrics` 与追踪端点加访问控制（仅内网或需要凭据），并说明为什么不应公网暴露。
- **B5** 用 Go 1.27 的 `net/http/httptest.NewTestServer`（配套 `testing/synctest` 的内存假网络）写一个不占真实端口的端到端传播测试。

## 4. 技术约束

| 约束 | 要求 |
| --- | --- |
| 语言版本 | Go 1.25 基线；使用 1.26 / 1.27 的能力必须标注版本 |
| 指标库 | `github.com/prometheus/client_golang`（以官方最新稳定版为准） |
| 追踪库 | `go.opentelemetry.io/otel`、`go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp` |
| 日志 | `log/slog`，禁止在日志里打印完整凭证与个人信息 |
| 命名 | 计数类指标以 `_total` 结尾，单位放名字末尾（`_seconds`、`_bytes`），同一指标不得混用毫秒与秒 |
| 禁止 | 把用户标识、业务主键、原始路径、错误全文作为标签；在业务代码里散落 `fmt.Println` 式调试输出 |

## 5. 交付物清单

| 交付物 | 说明 |
| --- | --- |
| `internal/metrics/` | 指标定义、显式注册、HTTP 与 gRPC 中间件 |
| `internal/tracing/` | SDK 初始化、传播器、gRPC 拦截器、手写 span 辅助函数 |
| `deploy/prometheus/` | 抓取配置 + 告警规则文件 |
| `deploy/grafana/` | 数据源配置 + 看板 JSON |
| `deploy/otel/` | Collector 配置（接收、处理、导出） |
| `compose.yaml` | 应用 + Prometheus + Grafana + OTel Collector |
| `README.md` | 接线说明、基数清单、告警推导、面板问答表 |
| `docs/trace-walkthrough.md` | 一次请求的链路截图或 span 树文本 |

## 6. 验收标准（可执行命令）

```bash
# 1. 一键起全套
docker compose up -d

# 2. 测试必须检出数据竞争
go test -race ./...

# 3. 打一次流量，然后确认指标有增量
curl -sS http://127.0.0.1:8080/api/v1/posts | head
curl -sS http://127.0.0.1:8080/metrics | grep -E 'http_requests_total|http_request_duration_seconds_count'

# 4. 校验告警规则文件语法（用 Prometheus 自带的规则检查工具）
docker compose exec prometheus promtool check rules /etc/prometheus/rules.yml
```

PowerShell 下第 3 条用 `curl.exe` 或 `Invoke-RestMethod`，避免 `curl` 别名差异。

验收判定：

| 编号 | 判定方式 |
| --- | --- |
| V1 | Prometheus 的 Targets 页面中应用实例为 `UP` |
| V2 | `promtool check rules` 退出码为 0 |
| V3 | 一次请求的日志 `trace_id` 能在追踪系统中检索到同一条链路 |
| V4 | 人为让下游超时后，链路中能一眼看出是下游那一跳变慢 |
| V5 | 指标标签中不存在禁止清单里的维度（导出 `/metrics` 后逐个核对） |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| Metrics 覆盖度 | 20 | 五类指标齐全、类型选对（Counter / Gauge / Histogram）、命名规范 |
| 标签基数控制 | 15 | 有禁止清单、有改造前后数据、路由用模板 |
| 告警设计 | 20 | 4 条规则可触发、阈值有推导、有 `for`、有最小流量门槛 |
| Grafana 看板 | 15 | 三组视图齐全、每个面板能回答问题、无面板膨胀 |
| Traces | 20 | 四边界传播完整、span 种类正确、关键操作有手写 span |
| 日志关联与文档 | 10 | 日志带 trace 标识、接线说明可复现、采样策略有依据 |

## 8. 提示与思路

1. **先定义问题再选指标。** 写下「值班的人半夜被叫醒时第一个想看什么」，再反推指标；从指标出发堆看板一定会得到一堆没人看的面板。
2. **分桶要对着 SLO 设。** 若 SLO 是「P99 小于 500ms」，桶边界就应在 100ms / 250ms / 500ms / 1s 附近加密，让 `histogram_quantile` 在阈值附近的误差可控。
3. **告警从症状出发，不从原因出发。** 错误率与延迟是症状，CPU 高是原因；症状告警更稳定，原因类指标放在看板里用于定位。
4. **traces 的断点大多在两个地方**：自己 `http.NewRequest` 而没包装 Transport 的出站调用，以及中途用 `context.Background()` 新建的上下文。
5. **本地验证优先用「一次请求对照三支柱」**：同一个 `trace_id` 在日志、链路各查一次，在 Grafana 看总量是否 +1，比逐个功能测试更能暴露接线错误。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 用原始 `URL.Path` 当标签 | 时间序列数量随请求参数爆炸，采集端内存暴涨 | 标签基数失控 | 用路由模板作为 `route` 标签 |
| 用 `Summary` 汇总多实例分位数 | 全局 P99 与真实值偏差大 | 分位数不可跨实例相加 | 用直方图并在服务端聚合计算 |
| 延迟指标不区分成功与失败 | 看不出故障请求的真实耗时 | 缺少结果维度 | 延迟指标加结果维度或在 SLO 中区分 |
| 告警没有 `for` 且无流量门槛 | 抖动触发告警风暴，随后被整体静音 | 瞬时尖峰当成故障 | 设置 `for` 与最小流量门槛 |
| 全部 span 标成 `internal` | 拓扑图上是一堆孤立的点 | span 种类未按语义设置 | `client` / `server` 正确配对 |
| 出站调用不设超时 | 一个慢依赖拖满所有 goroutine | 使用默认无超时客户端 | 每个出站调用显式设置超时并透传 `ctx` |
| 自己做一份运行时指标 | 与默认注册表重复，序列冲突 | 重复定义 `go_*` / `process_*` | 运行时指标交给库的默认注册表 |
| 测试共用全局注册表 | 单元测试之间指标互相干扰 | 全局状态共享 | 测试使用独立注册表 |
| 采样比率极低且无错误保留 | 出事时没有任何错误链路可查 | 采样与排障需求冲突 | 头部粗筛 + 尾部对错误与慢请求保留 |

## 10. 参考实现要点

**指标中间件：路由模板在匹配之后取。** 这是 M3/M8 的关键点，取错位置就等于放弃基数控制。

```go
func instrument(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		inFlight.Inc()
		defer inFlight.Dec()

		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)

		route := routePattern(r) // 必须是路由模板，绝不能是 r.URL.Path
		requests.WithLabelValues(r.Method, route, statusClass(rec.status)).Inc()
		duration.WithLabelValues(r.Method, route).Observe(time.Since(start).Seconds())
	})
}
```

**告警规则的四个方向。** 只给形态，阈值由自己的观测数据推导。

```yaml
groups:
  - name: service-slo
    rules:
      # 1. 错误率：5xx 占比超过阈值并持续一段时间，且有最小流量门槛
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{code="5xx"}[5m])) > 1
            and
          sum(rate(http_requests_total{code="5xx"}[5m]))
            / sum(rate(http_requests_total[5m])) > 0.01
        for: 10m

      # 2. P99 延迟：直方图聚合后计算，按路由维度定位
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)) > 0.5
        for: 5m

      # 3. 消费积压：hw13 留出的 outbox_pending 埋点在此接上
      - alert: ConsumerBacklog
        expr: max(outbox_pending) > 1000
        for: 15m

      # 4. 依赖不可用：下游调用失败率
      - alert: DependencyFailing
        expr: |
          sum(rate(dependency_calls_total{result="error"}[5m])) by (dependency)
            / sum(rate(dependency_calls_total[5m])) by (dependency) > 0.05
        for: 5m
```

为什么不用平均值告警：平均值会被大量快请求稀释，1% 的慢请求可以把 P99 抬到秒级而平均值几乎不动；延迟类告警要看分位数，且要区分失败请求的耗时。

| 决策点 | 选择 | 理由 |
| --- | --- | --- |
| 分位数计算位置 | 服务端聚合直方图 | 多实例可聚合，跨实例求分位数无意义 |
| 采样位置 | Collector 侧做尾部采样 | 只有请求结束后才能知道它是否出错或超时 |
| 日志与链路关联字段 | `trace_id` + `span_id` | 两个方向都能互跳，定位成本最低 |
| 看板组织 | 一个看板回答一类问题 | 超过 12 个面板就拆分，避免值班时找不到图 |
