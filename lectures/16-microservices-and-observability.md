# L16 微服务架构与可观测性

（本篇定位：回答「什么时候该拆、按什么拆、拆完怎么知道它是否健康」。对应 W12 与 W14，前置为 L13 gRPC、L11 结构化日志与错误处理、W7–W10 的 HTTP 服务与数据层。语言基线 Go 1.25，涉及 1.26 / 1.27 的能力单独标注版本。）

## 1. 什么时候不该上微服务

微服务不是架构默认值，而是组织规模与伸缩压力共同作用下的结果。判断依据是下表的具体信号，不是技术偏好。

| 维度 | 留在模块化单体的信号 | 倾向拆分的信号 |
| --- | --- | --- |
| 团队规模 | 2–8 人团队能读懂全部模块 | 多团队并行改同一份代码，合并冲突常态化 |
| 部署频率 | 全量发布几分钟，回滚就是切回上一个二进制 | 某模块的发布被其他在制品阻塞，一天要发多次 |
| 独立伸缩需求 | 各部分负载比例稳定，整体扩缩容成本可接受 | 只有一个模块吃 CPU/内存，其他模块被迫陪跑 |
| 数据边界清晰度 | 外键密集、事务跨表、边界仍在变 | 限界上下文稳定，各自拥有数据，跨边界只需接口或事件 |
| 故障隔离要求 | 单进程崩溃可整体重启 | 一个模块的内存泄漏不能拖垮支付链路 |
| 技术异构需求 | 单一语言栈够用 | 某模块必须用另一种运行时或模型 |

「模块化单体」不等于把代码堆进一个 main 包，而是**单一部署单元 + 强制模块边界**：每个限界上下文一个顶层包，对外只暴露接口与 DTO；模块间禁止 import 对方内部实现、禁止跨模块读写对方的表；跨模块调用统一走接口，日后把实现换成 RPC 客户端即可。拆分的代价必须与收益对账：

| 代价 | 具体表现 | 缓解手段 |
| --- | --- | --- |
| 分布式事务 | 原本一个 `BEGIN ... COMMIT` 变成跨进程一致性问题 | Outbox + 最终一致性，见 [L17](17-mq-and-kubernetes.md) |
| 网络故障 | 超时、重试、半开连接、DNS 抖动变成业务逻辑的一部分 | 超时必设、退避重试、幂等键、熔断，见 [L19](19-distributed-systems.md) |
| 调试复杂度 | 一次请求跨多进程，本地断点无法还原全貌 | 全链路 `trace_id` + 结构化日志 + 分布式追踪 |
| 运维成本 | 每个服务都要 CI、镜像、探针、看板、告警、值班手册 | Helm Chart 模板化 + 统一可观测性接入 |
| 版本兼容 | 上下游不同版本长期共存 | 契约优先（proto 先行）+ 只做向后兼容演进 |
| 延迟叠加 | 一次用户请求产生 N 次内部往返，尾延迟被放大 | 合并调用、并发调用、缓存、减少跳数 |

## 2. 拆分方法论

### 2.1 按业务能力与限界上下文拆

从业务语言出发，不从技术分层出发。`user-service` / `db-service` / `log-service` 这类技术维度拆分只是把单体按层切碎，不带来独立部署收益。

| 拆分依据 | 提问方式 | 产物 |
| --- | --- | --- |
| 业务能力 | 这个能力单独对外提供是否成立 | 订单、库存、结算、通知 |
| 限界上下文（DDD） | 同一个词在不同上下文里含义是否不同 | 独立领域模型与术语表 |
| 变更频率 | 哪些代码总是一起改 | 高频变更收敛到少数服务 |

### 2.2 按数据边界拆与共享数据库反模式

服务边界最终由数据所有权决定：**一个数据集合只能有一个写入方**。读可以冗余（读模型、缓存、物化视图），写必须单一。链路成立的标准是：写入方唯一 → 其他服务经接口或事件取得数据 → 不存在两个服务同时 `UPDATE` 同一张表。

| 反模式 | 现象 | 为什么坏 | 替代 |
| --- | --- | --- | --- |
| 多服务同库同表 | 两个服务都能改 `orders` | 隐式耦合，改表要同步发版 | 表归属单一服务，其他服务走接口 |
| 跨库 JOIN | 查询里 join 别人的表 | 数据库成为共享单点，无法独立演进 | 读模型 / 冗余字段 / 事件同步 |
| 跨库事务 | 一个事务写两个库 | 本地事务无法跨实例保证 | Outbox + 中继，或接受最终一致 |
| 「只读」共用 | 认为只读就安全 | 表结构变更直接打挂读方 SQL | 视图或接口固化，按需物化 |

### 2.3 服务粒度与契约先行

粒度自检：能否被一个团队独立部署回滚；接口是否小到能写在一页纸上；拆开后原本的本地调用是否变成多次串行 RPC；是否出现「分布式单体」（所有服务必须同时发布才工作）。最后一条成立就说明边界错了，先合并回去。接口一律**契约先行**：先写 `.proto` → 评审字段与语义 → 生成代码 → 各自实现。

```proto
syntax = "proto3";

package order.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
}

message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  string order_id = 1;
  int64 amount_cents = 2;   // 金额用整数分，避免浮点误差
  string status = 3;
}
```

兼容规则：只加字段、不删字段、不改字段语义、不复用已删除的字段编号；破坏性变更走新版本包名（`order.v2`）并行一段时间。

## 3. 服务注册与发现

| 模式 | 负载均衡决策方 | 代表 | 优点 | 缺点 |
| --- | --- | --- | --- | --- |
| 客户端发现 | 调用方进程 | gRPC + 注册中心 | 少一跳、策略可定制、延迟低 | 每种语言都要实现，客户端需感知注册中心 |
| 服务端发现 | 中间代理 / 平台 | k8s Service、网关 | 客户端零逻辑、语言无关 | 多一跳，策略受平台限制 |
| DNS 发现 | 解析结果 | k8s Headless Service | 无需额外组件 | 受 TTL 缓存影响，摘除不及时，需配合就绪探针 |

| 项目 | 一致性模型 | 健康检查 | 配置能力 | 典型生态 |
| --- | --- | --- | --- | --- |
| `etcd` | Raft 强一致，支持线性化读 | 靠客户端租约（lease）续约，自身不主动探活 | KV + watch，原语化，治理层需自建 | Kubernetes、gRPC 生态 |
| `Consul` | Raft 强一致，支持多数据中心 | 内置主动检查（HTTP/TCP/TTL/脚本） | KV + watch，配套完整 | 传统服务发现、多机房 |
| `Nacos` | 支持 AP/CP 切换 | 内置心跳与主动探测 | 命名空间 + 分组 + 灰度，注册与配置一体 | Spring Cloud / Java 生态 |

选型看已有技术栈：已在 k8s 就用平台 Service；需要 gRPC 原生解析、愿意自写治理逻辑用 `etcd`；跨机房、要开箱健康检查用 `Consul`。gRPC 接入注册中心的概念链路（具体 API 以 gRPC-Go 官方文档为准）：

```text
注册中心 ──watch 地址变化──► resolver ──► Address 列表（附属性：zone/权重/版本）
        ──► balancer（round_robin 或自定义）──► SubConn
        ──► 连接状态机：IDLE → CONNECTING → READY / TRANSIENT_FAILURE
```

要点：resolver 只产出地址列表，健康由连接状态与就绪探针体现；地址变更要增量推送，全量重建会造成发布期错误尖峰；灰度把版本写进地址属性交给 balancer 过滤；开发期可直连地址，生产期不要写死 IP。

| k8s 场景 | 结论 |
| --- | --- |
| 同集群内互调、无特殊路由 | `Service` + kube-dns 够用，交给平台做服务端发现 |
| 需要版本分流、zone 亲和、权重灰度 | 引入 resolver + balancer，或不走 `Service` |
| 有状态服务需稳定网络标识 | 用 Headless Service，按 DNS 多 A 记录自行处理 |
| 跨集群、跨云、混合部署 | `Service` 不够用，需要独立注册中心或服务网格 |

## 4. 配置中心

能力要求：集中配置（单一来源 + 多环境隔离，用命名空间或键前缀）；灰度（按实例标签或分组订阅不同的键，先给少量实例生效）；热更新（watch 变更 → 回调 → 原子替换内存快照）；版本与回滚（每次变更写一条不可变记录，当前生效版本用一个指针标识，回滚即指回旧版本号并触发一次 watch 事件，复用同一条热更新通路）。正确形态是「启动拉全量 + 运行期 watch + 内存缓存 + 变更回调」，读路径完全不依赖网络：

```go
type Config struct {
	TimeoutMS   int
	LogLevel    string
	FeatureFlag map[string]bool
}

type Store struct{ snap atomic.Pointer[Config] } // atomic.Pointer 见 sync/atomic

func (s *Store) Load() *Config { return s.snap.Load() }

// watch 回调：解析 → 校验 → 原子替换 → 通知订阅者
func (s *Store) apply(raw []byte) error {
	var c Config
	if err := parseConfig(raw, &c); err != nil {
		return err // 解析或校验失败必须丢弃，保留旧快照
	}
	s.snap.Store(&c)
	return nil
}
```

watch 连接会断，必须有重连并重拉全量；解析失败时保留旧配置，不能把空配置写进去。热更新只允许「读取幂等、无副作用、无需重建资源」的参数：

| 可安全热更新 | 必须重启 |
| --- | --- |
| 日志级别、采样率、超时时间 | 监听端口、TLS 证书链（除非实现证书热加载） |
| 限流阈值、熔断参数 | 数据库连接串（涉及连接池重建） |
| 功能开关、灰度白名单 | 序列化格式、分片数量 |

## 5. 可观测性三支柱与信号

| 支柱 | 数据形态 | 成本 | 擅长回答 | 不擅长 |
| --- | --- | --- | --- | --- |
| Metrics | 聚合数值时间序列 | 低，与请求量基本无关 | 是否异常、持续多久、影响多大 → 告警 | 单次请求为什么失败 |
| Logs | 离散事件文本 | 高，与请求量成正比 | 这一次具体发生了什么、上下文细节 | 跨服务整体趋势 |
| Traces | 跨服务因果链 | 中，可采样 | 慢在哪一跳、哪个依赖拖累尾延迟 | 长期趋势与聚合统计 |

使用顺序是**告警看 Metrics → 定位看 Traces → 归因看 Logs**。没有 Metrics 的可观测性是「出事才知道」，没有 Traces 的排查是「逐个服务加日志重发」。

| 方法 | 适用对象 | 三个维度 |
| --- | --- | --- |
| 四个黄金信号 | 所有服务 | 延迟、流量、错误、饱和度 |
| USE | 资源（CPU、内存、磁盘、网卡、连接池） | 使用率、饱和度、错误 |
| RED | 服务（面向请求） | 速率、错误、耗时 |

延迟要区分成功与失败，失败请求的耗时分布与成功请求不同；饱和度看最受限的那一种资源，而不是所有资源的平均值。资源看板用 USE，服务看板用 RED，RED 变差时回到 USE 找饱和点。

## 6. Prometheus + Go

模块路径 `github.com/prometheus/client_golang`（以官方最新稳定版为准，见 <https://pkg.go.dev/github.com/prometheus/client_golang>）。

| 类型 | 语义 | 适用 |
| --- | --- | --- |
| `Counter` | 只增不减的累计值 | 请求总数、错误总数、处理字节数 |
| `Gauge` | 可增可减的瞬时值 | 在途请求数、连接池占用、队列长度 |
| `Histogram` | 分桶计数 + `_sum` / `_count` | 延迟、请求体大小 → 服务端聚合算分位数 |
| `Summary` | 客户端预计算分位数 | 单实例本地分位数，无法跨实例聚合 |

### 6.1 Histogram 优先于 Summary

| 维度 | `Histogram` | `Summary` |
| --- | --- | --- |
| 分位数计算位置 | 服务端，PromQL 聚合后计算 | 客户端，每实例各算各的 |
| 多实例聚合 | 可以：先对桶求和再 `histogram_quantile` | 不可以：跨实例求分位数数学上无意义 |
| 配置成本 | 需预设桶边界，选错精度就差 | 需预设目标分位数与误差 |
| 序列数量 | 桶数 × 标签组合 | 分位数数量 × 标签组合 |

`Summary` 的分位数是每个实例在本地滑窗算出来的，把 10 个实例的 P99 平均一次得到的不是全局 P99——这就是「分位数在服务端聚合的问题」。默认选 `Histogram`。

### 6.2 标签基数与业务指标

| 反模式 | 后果 |
| --- | --- |
| `user_id`、`order_id`、原始 URL 作标签 | 时间序列数量爆炸，采集端与应用内存同时崩溃 |
| 错误全文作标签 | 熵极高，等价于每个错误一条序列 |
| `method`+`route`+`code`+`host`+`version`+`region` 全维度组合 | 组合爆炸，实际查询只用得上两三个 |
| 用标签区分同一语义的多种取值 | 应拆成多个指标或做归类聚合 |

标签只放低基数、可枚举、查询必需的维度：路由用**路由模板**（`/orders/:id`）而不是真实路径，错误用类型枚举而不是消息文本。业务指标直接对应业务健康度：

| 指标 | 类型 | 标签 | 用途 |
| --- | --- | --- | --- |
| `orders_created_total` | Counter | `channel`、`result` | 下单成功率趋势与渠道对比 |
| `payment_amount_cents_total` | Counter | `currency` | 成交额累积（整数分避免浮点误差） |
| `inventory_available` | Gauge | `sku_class` | 库存水位，触发补货告警 |
| `queue_backlog` | Gauge | `topic` | 消费积压，扩容依据 |

### 6.3 暴露 `/metrics` 与运行时指标

```go
// 具体构造函数签名以官方文档为准
var httpRequests = prometheus.NewCounterVec( /* http_requests_total{method,route,code} */ )
var httpDuration = prometheus.NewHistogramVec( /* http_request_duration_seconds{method,route} */ )
var httpInFlight = prometheus.NewGauge( /* http_in_flight_requests */ )

func init() { prometheus.MustRegister(httpRequests, httpDuration, httpInFlight) }
```

中间件里的用法见第 10 节的 `handleQuote`：进入时 `httpInFlight.Inc()`、退出时 `Dec()`，处理完按路由模板与状态码 `WithLabelValues(...).Inc()`，并用 `Observe(...)` 记录耗时。

`/metrics` 由 `promhttp` 提供处理器（例如 `mux.Handle("/metrics", promhttp.Handler())`）。Go 运行时指标（goroutine 数、GC 次数与停顿、堆内存等）由默认注册表自动暴露，常见的 `go_*` / `process_*` 指标名以官方文档与实际输出为准。**不需要自己重写这些指标**，也不要重复定义一份。

### 6.4 告警规则

```yaml
groups:
  - name: service-slo
    rules:
      - alert: HighErrorRate          # 要求最小流量，避免低流量噪音
        expr: |
          sum(rate(http_requests_total[5m])) > 1
            and sum(rate(http_requests_total{code="5xx"}[5m]))
                / sum(rate(http_requests_total[5m])) > 0.01
        for: 10m
      - alert: HighLatencyP99         # 直方图先按 le 聚合再算分位数
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)) > 0.5
        for: 5m
      - alert: SaturationHigh         # 饱和度：在途请求逼近并发上限
        expr: sum(http_in_flight_requests) > 0.8 * 200
        for: 5m
```

`for` 必须存在以避免抖动误报；错误率告警要配最小流量门槛；分位数告警看趋势而非单点。

## 7. Grafana 看板组织

| 看板 | 回答的问题 | 主要内容 | 变量 |
| --- | --- | --- | --- |
| 服务概览 | 现在服务健康吗 | RED 指标 + 四个黄金信号 + SLO 达成率 | `service`、`env` |
| 依赖拓扑 | 谁在拖累谁 | 各下游延迟/错误率、连接池、重试次数 | `service`、`upstream` |
| 资源使用 | 哪里先饱和 | USE：CPU、内存、GC、goroutine 数、连接池、fd | `instance`、`pod` |
| 业务指标 | 业务是否正常 | 订单量、支付成功率、库存水位、活跃用户 | `channel`、`region` |

数据源统一指向 Prometheus，追踪由 Grafana 的追踪数据源关联。避免看板膨胀：一个看板只回答一类问题，超过 12 个面板就拆；顶层只放异常时才需要看的图，明细下钻到子看板；变量默认值要直接反映生产状态；告警面板与告警规则一一对应，避免「告警响了却找不到对应图」。

## 8. OpenTelemetry

模块路径（均以官方最新稳定版为准，见 <https://pkg.go.dev/go.opentelemetry.io/otel>；具体构造函数与配置项名称以官方文档为准）：

| 用途 | 模块路径 |
| --- | --- |
| 核心 API | `go.opentelemetry.io/otel` |
| SDK | `go.opentelemetry.io/otel/sdk` |
| Trace | `go.opentelemetry.io/otel/trace` |
| Metric | `go.opentelemetry.io/otel/metric`；导出器 `go.opentelemetry.io/otel/exporters`（OTLP 子模块在此下） |
| HTTP 自动埋点 | `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp` |

Trace 是一次业务请求的因果链，由 `trace_id` 标识；Span 是链路上的一个操作单元，由 `span_id` 标识，记录起止时间、名称、属性与事件，有 parent 则构成层级；`context.Context` 是 Go 里传递当前 span 的载体；Baggage 承载跨服务透传的业务键值，会进入每个请求的头，谨慎使用。跨进程传播默认用 `W3C tracecontext`（`traceparent` / `tracestate`），HTTP 走 header、gRPC 走 metadata，底层是同一组头。`SpanKind` 取 `server`（服务端入口）、`client`（调用下游，与下游 `server` 配对）、`producer` / `consumer`（消息收发配对）、`internal`（进程内操作）；`client` / `server` 配对正确链路才会连成一条线，全部标成 `internal` 会得到一堆孤立的点。

采样策略分两类：**head sampling** 在请求开始时由 SDK 决定，成本可控但无法预知是否出错，可能错过关键样本；**tail sampling** 在请求结束后由 Collector 决定，可对错误与慢请求全量保留，代价是 Collector 必须缓存完整 trace，内存与复杂度都高。常见做法是头部按比率粗筛、尾部对错误与慢请求提权重。

采样率是成本与信噪比的旋钮：全量采样在高峰可能让 Collector 与存储先于业务崩溃，1% 采样在大流量下足够看 P99 趋势但无法还原单个用户投诉。常见配置是生产 1%–10% 加「错误与慢请求强制保留」。导出链路：

```text
应用（OTel SDK）──OTLP（gRPC 4317 / HTTP 4318）──► Collector(Agent)
  ──processors: batch、memory_limiter、tail_sampling──► Collector(Gateway，可选)
  ──exporters──► 后端（Tempo / Jaeger / 托管服务）
```

应用只面向 OTLP 输出，换后端不改应用；`batch` 与 `memory_limiter` 处理器几乎必备；tail sampling 只能在 Collector 做。自动埋点与手动 span 的分工：

| 位置 | 由谁负责 |
| --- | --- |
| HTTP 入站 / 出站 span | `otelhttp` 中间件与 Transport 自动创建 |
| gRPC 出站 span | gRPC 拦截器 |
| 数据库查询 span | 手动或驱动侧拦截器 |
| 关键业务步骤（校验、扣库存、发消息） | 手动 span，命名用业务语义 |

```go
// 手动 span：继承上游上下文，命名用业务语义，错误显式记录
ctx, span := startSpan(ctx, "order.create")
defer span.End()
setAttr(span, "order.channel", in.Channel)
if err := validate(in); err != nil {
	recordError(span, err) // 是否标记 span 失败按业务语义决定
	return "", err
}

// 日志关联：把 trace_id / span_id 注入 slog 字段，日志与链路可双向跳转
func withTrace(ctx context.Context, logger *slog.Logger) *slog.Logger {
	if sc := spanContextFrom(ctx); sc.IsValid() {
		return logger.With(slog.String("trace_id", sc.TraceID().String()))
	}
	return logger
}
```

不要把业务语义全塞进自动 span，否则链路里全是无差别的 `GET /api/v1/orders`。

日志来源位置可用 1.25 的 `slog.Record.Source` 标记，成组字段可用 `slog.GroupAttrs`；需要同时输出到控制台与采集端时用 1.26 新增的 `log/slog.NewMultiHandler`，注意它会调用所有 handler，同一份记录会被处理多次。

## 9. HTTP 与 gRPC 之间的链路传播

| 边界 | 载体 | 注入方 | 提取方 |
| --- | --- | --- | --- |
| HTTP 入站 | `traceparent` 请求头 | 上游客户端 / 网关 | 服务端中间件 |
| HTTP 出站 | `traceparent` 请求头 | 出站 Transport / 中间件 | 下游服务端 |
| gRPC 出入站 | metadata（键名同 `traceparent`） | 客户端拦截器 | 服务端拦截器 |
| 消息队列 | 消息 header / 属性 | 生产者 | 消费者 |

规则是**每个进出的边界都要有一次提取与一次注入**，漏掉任一边链路即断；常见断点是自己 `http.NewRequest` 而没包装 Transport、在 goroutine 里新建 `context.Background()`、以及直连数据库的调用。

```go
func unaryServerInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (any, error) {
	ctx = extractFromMetadata(ctx)              // 提取 traceparent，得到父 span
	ctx, span := startSpan(ctx, info.FullMethod) // 为本次 RPC 建 span
	defer span.End()

	resp, err := handler(ctx, req)
	if err != nil {
		recordError(span, err)
	}
	return resp, err
}
```

## 10. 完整示例：HTTP 服务 + gRPC 依赖

```go
type Downstream interface {
	Quote(ctx context.Context, sku string) (int64, error) // 实现由 gRPC 客户端提供
}

type Server struct {
	downstream Downstream
	logger     *slog.Logger
	requests   *prometheus.CounterVec   /* http_requests_total{method,route,code} */
	duration   *prometheus.HistogramVec /* http_request_duration_seconds{method,route} */
	inFlight   *prometheus.Gauge        /* http_in_flight_requests */
	failClosed atomic.Int64             /* 业务计数：下游失败次数 */
}

func (s *Server) handleQuote(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	s.inFlight.Inc()
	defer s.inFlight.Dec()

	logger := withTrace(r.Context(), s.logger) // 日志带 trace_id
	callCtx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
	defer cancel() // 出站调用必须显式超时，并透传上游 ctx

	status := http.StatusOK
	price, err := s.downstream.Quote(callCtx, r.URL.Query().Get("sku"))
	if err != nil {
		s.failClosed.Add(1)
		logger.Error("downstream quote failed", slog.String("err", err.Error()))
		status = http.StatusBadGateway
		http.Error(w, "upstream unavailable", status)
	} else {
		logger.Info("quote ok", slog.Int64("price_cents", price))
		w.Header().Set("Content-Type", "application/json")
		_, _ = w.Write([]byte(`{"price_cents":` + itoa(price) + `}`))
	}

	route := "/api/v1/quote" // 从路由模板取，禁止用 r.URL.Path
	s.requests.WithLabelValues(r.Method, route, statusClass(status)).Inc()
	s.duration.WithLabelValues(r.Method, route).Observe(time.Since(start).Seconds())
}

// 装配：http.NewServeMux() 上挂 "/metrics"（promhttp.Handler()）与业务处理器，
// 业务处理器用 instrument 中间件包住；优雅退出与探针见 L17。
```

这个例子同时成立四件事：中间件提取/生成 trace 上下文、三个指标覆盖请求数/延迟直方图/在途数、结构化日志带 `trace_id`、下游调用有显式超时且失败被记录。本地接线用 `compose.yaml`：

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_SERVICE_NAME: quote-api
    depends_on: [otel-collector]
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/config.yaml"]
    volumes: ["./otel/config.yaml:/etc/otel/config.yaml:ro"]
    ports: ["4317:4317", "4318:4318"]
  prometheus:
    image: prom/prometheus:latest
    volumes: ["./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro"]
    ports: ["9090:9090"]
  grafana:
    image: grafana/grafana:latest
    environment: { GF_AUTH_ANONYMOUS_ENABLED: "true" }
    ports: ["3000:3000"]
```

接线检查：Prometheus 的 `scrape_configs` 指向 `app:8080/metrics`，在 Targets 页确认实例为 `UP`；应用 OTLP 端点指向 `otel-collector:4317`（gRPC）或 `:4318`（HTTP），确认 Collector 日志无导出失败；Grafana 添加 Prometheus 数据源后用 `up` 验证连通；触发一次请求，确认链路中同时存在应用 span 与下游 `client` span，且日志里的 `trace_id` 能在追踪系统中搜到。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 按技术分层拆服务 | 每次业务改动要同时发多个服务 | 拆的是层不是业务能力 | 按业务能力与限界上下文拆 |
| 多服务共用一张表 | 改表结构必须同步发版 | 数据写入方不唯一 | 表归属单一服务，其他服务走接口或事件 |
| HTTP 指标用 `r.URL.Path` 当标签 | 时间序列数随请求参数爆炸 | 标签基数失控 | 用路由模板作为 `route` 标签 |
| 用 `Summary` 汇总多实例分位数 | 全局 P99 与真实值偏差大 | 分位数不可跨实例聚合 | 用 `Histogram` + 服务端聚合 |
| 出站调用不设超时 | 一个慢依赖拖满所有 goroutine | 使用默认无超时客户端 | 每次出站显式设超时并透传 context |
| 用 `context.Background()` 发起下游调用 | 链路断开、取消信号丢失 | 未沿调用链传 ctx | 一路透传 `ctx`，不中途新建 |
| 把 `user_id` 放进指标标签 | 采集端内存暴涨、写入变慢 | 高基数标签 | 用分群等低基数维度替代 |
| 热更新不做校验直接生效 | 一次误操作推空配置，全量实例崩溃 | 缺少校验与失败保留 | 解析校验失败即丢弃，保留旧快照 |
| 告警没有 `for` | 抖动触发告警风暴后被整体静音 | 把瞬时抖动当故障 | 设置 `for` 与最小流量门槛 |
| 自动埋点承担全部业务语义 | 链路里全是同名 HTTP span | 缺少手动 span | 关键业务步骤补手动 span |
| 依赖 DNS 发现但不配就绪探针 | 流量被送到尚未就绪的 Pod | TTL 缓存内无法感知未就绪 | 正确实现就绪探针，或改用订阅式发现 |

## 12. 动手练习

1. **模块边界梳理**：按限界上下文画出模块依赖图，列出所有跨模块直接访问表的调用点并改为接口调用，记录改动前后模块间 import 数量。
2. **拆掉一个跨库 JOIN**：把一处跨模块 join 改成「接口调用 + 本地冗余字段」，补补偿任务保证最终一致，说明不一致窗口的最大时长。
3. **指标与分位数改造**：为现有 HTTP 服务加请求数、延迟直方图、在途请求三个指标，把 `route` 从真实路径改成路由模板；再同时暴露 `Histogram` 与 `Summary`，起两个实例打流量，用 PromQL 对比「各实例分位数平均」与「聚合后计算的分位数」的差异幅度。
5. **打通两跳链路与日志关联**：用上面的 compose 起 Prometheus + Grafana + OTel Collector，实现「HTTP 入口 → gRPC 下游」链路，人为制造一次下游超时，确认能从追踪里定位到是下游这一跳慢；再把 `trace_id` 注入 `slog`，用同一 `trace_id` 在日志与追踪系统各查一次。
6. **告警验证**：为服务写错误率、P99、饱和度三条告警规则，人为触发并记录 `for` 的实际生效时间。

## 13. 自检清单

- [ ] 每个服务有唯一数据写入方；对外接口由 proto 或 OpenAPI 契约定义，且能识别破坏性变更。
- [ ] 指标标签全是低基数可枚举维度；延迟分位数用 `Histogram` 在服务端聚合，运行时指标未重复定义。
- [ ] 告警规则都有 `for`，错误率类告警有最小流量门槛；四个看板都能对应到具体问题。
- [ ] 链路中 `client` / `server` span 正确配对；日志带 `trace_id` 可双向跳转。
- [ ] 配置热更新有校验且失败保留旧值；不可热更新的参数被明确排除。
- [ ] 本地能用 compose 一键起可观测性组件并验证数据流通。

## 14. 延伸阅读

- Prometheus 直方图与分位数实践：<https://prometheus.io/docs/practices/histograms/>；命名与标签实践：<https://prometheus.io/docs/practices/naming/>
- `client_golang` 文档：<https://pkg.go.dev/github.com/prometheus/client_golang>；OpenTelemetry Go 文档：<https://opentelemetry.io/docs/languages/go/>，采样概念：<https://opentelemetry.io/docs/concepts/sampling/>
- W3C Trace Context 规范：<https://www.w3.org/TR/trace-context/>
- 《Site Reliability Engineering》，作者：Betsy Beyer 等，出版社：O'Reilly Media
- 《Designing Data-Intensive Applications》，作者：Martin Kleppmann，出版社：O'Reilly Media
- Go 官方 Release Notes：<https://go.dev/doc/go1.25>、<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>
- 相关讲义：[L17 消息队列、容器化与 Kubernetes 部署](17-mq-and-kubernetes.md)、[L18 运行时原理](18-runtime-internals.md)、[L19 分布式基础](19-distributed-systems.md)
