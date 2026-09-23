# 阶段四：微服务与云原生（W11–W16）

- 时间：**2026-12-14（周一）至 2027-01-24（周日）**，共 6 周（含 2027-01-01 元旦）
- 对应原始规划：阶段 4「微服务 / 云原生」，建议周期 6–8 周，本计划取 6 周
- 作业：hw11（12-20）、hw12（12-27）、hw13（2027-01-03）、hw14（01-10）、hw15（01-17）、hw16（01-24）
- 阶段门：G4

---

## 1. 这一阶段要解决什么

从「一个能跑的服务」到「一组能被观测、能被部署、能在依赖故障时活下来的服务」。

| 目标 | 判定标准 |
| --- | --- |
| 契约先行 | 先写 `.proto`，再生成代码；字段编号有分配表与 `reserved`；CI 校验生成结果无 diff |
| 服务能互相找到 | 注册中心可见实例列表；实例上下线能热更新到调用方；L4 负载均衡不足的原因能讲清 |
| 异步不丢数据 | 落库与发消息不产生「落库成功但消息丢失」；Outbox 与幂等消费都有实现与测试 |
| 故障能被看见 | 一次跨服务请求能用 trace 串起来；有指标、有告警、有看板，且标签基数受控 |
| 故障能被挡住 | 限流、熔断、降级、隔离四件套至少三件有实现与验证 |
| 能部署能回滚 | `helm lint`/`template` 通过；有探针、有资源参数推导、有零 5xx 的滚动更新验证与回滚演练 |

---

## 2. W11：gRPC 与 Protobuf（2026-12-14 ~ 12-20）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 12-14 | 一 | RPC 基础与 proto3 | REST/JSON 与 gRPC/Protobuf 对比；proto3 语法（字段编号、`repeated`、`map`、`oneof`、`optional`、枚举、`reserved`）；为什么编号绝不可复用 | `user.proto` 初版 |
| 12-15 | 二 | 生成工具链 | `buf`（`github.com/bufbuild/buf`）或 `protoc` + 插件；`buf lint` / `buf breaking`；生成代码提交进仓库，并写 CI 检查「重新生成无 diff」 | 可复现的生成流程 |
| 12-16 | 三 | 服务端四种方法 | 一元、服务端流、客户端流、双向流；`stream.Context()`、`Send` 错误检查、`KeepaliveParams`、最大消息大小、健康检查、reflection | 服务端实现 |
| 12-17 | 四 | 拦截器与错误模型 | 一元 + 流式两套拦截器（请求 ID、slog、recover、鉴权）；gRPC 状态码 → 业务错误码映射表；结构化错误 | 拦截器套件 + 映射表 |
| 12-18 | 五 | 客户端与 REST 并存 | deadline 传播、元数据注入、幂等接口重试与非幂等接口不重试；用 gRPC-Gateway 或手写适配暴露一个 HTTP 接口，复用同一 service 层 | 客户端 + HTTP 适配 |
| 12-19 | 六 | 作业主体 | `bufconn` 或真实端口端到端测试：成功、超时、鉴权失败、流式中断 | hw11 主体完成 |
| 12-20 | 日 | 收口 | 兼容演进演示（加字段、加枚举值、`reserved` 废弃）；自评打分 | **hw11 交付** |

配套讲义：[L15 gRPC 与 Protobuf](../lectures/15-grpc-and-protobuf.md)

铁律：**不要手改生成代码。** 需要扩展行为就写包装层，而不是往 `*.pb.go` 里加方法。

---

## 3. W12：服务拆分、注册发现与配置中心（2026-12-21 ~ 12-27）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 12-21 | 一 | 拆分与契约 | 拆出 `user-service` 与 `post-service`；共享数据库反模式的识别；服务粒度判断；先用静态地址跑通 | 两个服务 + 调用链路 |
| 12-22 | 二 | 服务注册 | 启动注册（服务名/地址/端口/版本/健康）、优雅关闭注销、租约与心跳续期；「`kill -9` 后的脏节点清理」 | 注册模块 |
| 12-23 | 三 | 服务发现与负载均衡 | 实例列表拉取 + watch 热更新；轮询与最少连接策略；**为什么 gRPC 长连接下 L4 负载均衡不够用**（连接级均衡 vs 请求级均衡） | 客户端发现 + 策略单测 |
| 12-24 | 四 | 健康检查 | `grpc_health_v1` 或自定义端点；注册中心健康状态与应用就绪状态的联动；不健康实例立刻摘除 | 健康检查链路 |
| 12-25 | 五 | 配置中心与降级 | 启动拉取 + watch 热更新；本地缓存；配置中心不可用时用上次成功配置启动；配置白名单（只允许热更新可安全热更新的参数） | 配置中心接入 |
| 12-26 | 六 | 作业主体与故障演练 | 停掉一个实例验证请求全部成功；停掉注册中心验证已启动实例继续工作 | hw12 主体完成 |
| 12-27 | 日 | 收口 | 架构图 + 一次调用的完整链路说明；自评打分 | **hw12 交付** |

配套讲义：[L16 微服务架构与可观测性](../lectures/16-microservices-and-observability.md)

---

## 4. W13：消息队列与最终一致性（2026-12-28 ~ 2027-01-03）

本周包含 **2027-01-01（周五，元旦）**。处理方式：01-01 只做复盘与休整，不推进新内容；hw13 交付日仍为 01-03，若进度紧张可顺延到 01-05 并记入日志。

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 12-28 | 一 | MQ 选型与投递语义 | Kafka / RabbitMQ / NATS / Redis Stream 对比；at-most-once / at-least-once / exactly-once 的真实含义（exactly-once 通常靠幂等实现） | 选型说明 + MQ 接入 |
| 12-29 | 二 | Outbox 模式 | 同一事务内写业务表与 `outbox` 表；中继轮询投递并标记；outbox 表结构与索引；**避免「落库成功但消息丢失」** | Outbox 迁移 + 中继代码 |
| 12-30 | 三 | 幂等消费 | 业务唯一键 + 去重表，或状态机 + 版本号乐观并发；写「同一消息投递两次，业务只生效一次」的测试 | 幂等消费者 |
| 12-31 | 四 | 顺序、重试与死信 | 分区键与顺序边界；按 key 路由到固定 worker；手动提交位移的时机；指数退避与死信队列 | 可靠性机制 |
| 01-01 | 五 | 元旦 | 休整与复盘，只做回顾不推新内容 | 复盘记录 |
| 01-02 | 六 | 作业主体 | 制造消费者失败 → 重试 → 死信 → 重放的演示脚本 | hw13 主体完成 |
| 01-03 | 日 | 收口 | 消息流时序图、投递语义说明、失败处理策略表；自评打分 | **hw13 交付** |

配套讲义：[L17 消息队列、容器化与 Kubernetes 部署](../lectures/17-mq-and-kubernetes.md)

Outbox 的价值在于把一个分布式问题变回单机事务问题。判断标准很简单：**数据库提交成功的那一刻，消息一定不会被丢**。做不到这一点，方案就是错的。

---

## 5. W14：可观测性（2027-01-04 ~ 01-10）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 01-04 | 一 | 三支柱与指标体系 | Metrics / Logs / Traces 各自适合回答什么；四黄金信号（延迟、流量、错误、饱和度）与 RED/USE；为什么不用平均值告警 | 指标设计表 |
| 01-05 | 二 | Prometheus 指标 | `github.com/prometheus/client_golang`（模块路径，以官方最新稳定版为准）；`Counter`/`Gauge`/`Histogram`/`Summary` 的选择；**标签基数灾难**与「禁止进指标的标签」清单 | `/metrics` 埋点 |
| 01-06 | 三 | 告警与看板 | 至少 4 条 PromQL 告警（错误率、P99、消费积压、依赖不可用）与阈值推导理由；Grafana 看板每个面板「回答什么问题」 | 告警规则 + 看板 |
| 01-07 | 四 | OpenTelemetry trace | `go.opentelemetry.io/otel` 与 `otelhttp`（模块路径，以官方最新稳定版为准）；W3C tracecontext 传播；HTTP 与 gRPC 之间的上下文传递；手写 span 覆盖 DB/缓存/MQ | 跨服务链路 |
| 01-08 | 五 | 日志关联与 Collector | 把 `trace_id` 注入 `slog` 字段，实现「日志 ↔ trace」互跳；OTLP 导出器对接 Collector 的架构；采样策略与成本权衡 | 三支柱打通 |
| 01-09 | 六 | 作业主体 | `compose.yaml` 起 Prometheus + Grafana + OTel Collector，端到端验证 | hw14 主体完成 |
| 01-10 | 日 | 收口 | 一次跨服务请求的完整证据链（trace + 日志 + 指标）；自评打分 | **hw14 交付** |

配套讲义：[L16](../lectures/16-microservices-and-observability.md)

---

## 6. W15：限流、熔断与网关（2027-01-11 ~ 01-17）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 01-11 | 一 | 反向代理 | `net/http/httputil.ReverseProxy`；**必须用 `Rewrite` 而不是已废弃的 `Director`**（Go 1.26 起废弃，原因是恶意客户端可用 hop-by-hop 头删除 Director 添加的头）；路由表配置化 | 网关骨架 |
| 01-12 | 二 | 限流算法 | 固定窗口的临界问题、滑动窗口（日志/计数两种）、漏桶、令牌桶（允许突发）；`golang.org/x/time/rate`（模块路径）；限流键的选择与 429 + `Retry-After` | 本地限流 |
| 01-13 | 三 | 分布式限流 | Redis + Lua 原子令牌桶；本地限流 + 分布式限流的分层组合；为什么不能只靠单机限流 | 分布式限流 |
| 01-14 | 四 | 熔断 | 三态机（closed / open / half-open）+ 滑动窗口统计失败率与慢调用比例；恢复试探；**熔断与重试的相互作用（重试会放大故障）** | 熔断器实现 |
| 01-15 | 五 | 降级与隔离 | 降级路径定义与测试；按下游隔离连接池与并发上限，说明「一个慢下游拖垮整个网关」的机制与阻断方式；网关层统一鉴权、请求 ID 与信任边界 | 降级与隔离 |
| 01-16 | 六 | 作业主体 | 压测报告（阈值下的实际 QPS 与 429 比例）；**用 Go 1.25 的 `testing/synctest` 做限流算法的虚拟时间测试** | hw15 主体完成 |
| 01-17 | 日 | 收口 | 演示脚本：正常 → 超阈值 429 → 下游故障熔断 → 恢复；自评打分 | **hw15 交付** |

配套讲义：[L16](../lectures/16-microservices-and-observability.md)、[L19 分布式基础](../lectures/19-distributed-systems.md)（限流与熔断部分）

---

## 7. W16：Kubernetes 与 CI/CD（2027-01-18 ~ 01-24）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 01-18 | 一 | 容器化收口 | 多阶段构建、`distroless`/`alpine` 取舍（CGO、时区、CA 证书）、非 root、镜像 tag 策略（禁止 `latest`） | 生产级镜像 |
| 01-19 | 二 | K8s 核心对象 | Pod / Deployment / Service / Ingress / ConfigMap / Secret / Job / HPA / StatefulSet / PVC 的职责表；本地集群用 kind 或 minikube（以官方文档为准） | 可部署的清单 |
| 01-20 | 三 | 探针与运行参数 | `startup` / `readiness` / `liveness` 的参数推导；复现并修复「readiness 配错导致滚动更新期间 502」；**Go 1.25 起容器感知 `GOMAXPROCS`**（Linux 上考虑 cgroup CPU bandwidth limit，对应 k8s 的 CPU limit）与「CPU 限流」排查；`GOMEMLIMIT` 与内存 limit 配合避免 OOMKilled | 参数推导表 + 故障复现记录 |
| 01-21 | 四 | 优雅退出 | `terminationGracePeriodSeconds` + `preStop` + 应用内 `Shutdown` 的顺序；MQ 消费者收尾；验证「滚动更新期间零 5xx」 | 零 5xx 验证脚本 |
| 01-22 | 五 | Helm 与 CI/CD | Chart 结构（`Chart.yaml` / `values.yaml` / `templates`）；多环境 values；`helm lint`/`template`/`upgrade --atomic`；CI 流水线（lint → vet → test -race → 构建推送 → 部署）与回滚步骤 | Chart + CI 工作流 |
| 01-23 | 六 | 作业主体 | Secret 注入方式对比（环境变量 vs 挂载，含热更新差异）；HPA 与 PDB；Prometheus 抓取配置 | hw16 主体完成 |
| 01-24 | 日 | 收口与阶段门 | 回滚演练记录；按 G4 清单自评；阶段复盘 | **hw16 交付 + G4 通过** |

配套讲义：[L17](../lectures/17-mq-and-kubernetes.md)

---

## 8. 本阶段的版本注意事项

| 特性 | 版本 | 影响 |
| --- | --- | --- |
| 容器感知 `GOMAXPROCS` | 1.25 | Linux 上运行时按 cgroup CPU bandwidth limit 设置 `GOMAXPROCS`，并用 `GODEBUG=containermaxprocs=0`/`updatemaxprocs=0` 可关闭；这是排查「CPU 限流」的关键 |
| `errors.AsType` | 1.26 | 跨层错误判定可用泛型版，更安全也更快 |
| `ReverseProxy.Director` 废弃 | 1.26 | 网关必须用 `Rewrite` |
| `ServeMux` 尾斜杠重定向 307 | 1.26 | 网关与服务的重定向行为变化，注意客户端缓存语义 |
| `log/slog.NewMultiHandler` | 1.26 | 同时输出到 stdout 与文件的官方方案 |
| pprof Web UI 默认火焰图 | 1.26 | 采 profile 后直接看火焰图 |
| 泛型方法 | 1.27 | 通用工具（限流器、缓存、重试器）可以写成类型上的泛型方法而非包级函数 |
| 标准库 `uuid` | 1.27 | 生成请求 ID / 事件 ID 可不再引入第三方 UUID 库，见 <https://pkg.go.dev/uuid> |
| `go test` 默认跑 `stdversion` | 1.27 | 多服务仓库在 CI 中可能因版本标注不一致而失败 |
| `httptest.NewTestServer` | 1.27 | 配合 `testing/synctest` 测 HTTP 与超时逻辑 |

---

## 9. 阶段验收清单（G4）

- [ ] 至少 2 个服务，通过注册中心互相发现并调用（不是硬编码地址）
- [ ] 停掉 1 个实例后请求全部成功；停掉注册中心后已启动实例继续工作（有演练记录）
- [ ] `.proto` 有字段编号分配表与 `reserved`，`buf lint` 通过，CI 校验生成结果无 diff
- [ ] gRPC 状态码 → 业务错误码映射表完整，且每类错误有可触发用例
- [ ] 消息投递：数据库提交成功即不丢消息（Outbox 有实现与测试），重复投递业务只生效一次
- [ ] 一次跨服务请求能用 trace 串起，且日志中可通过 `trace_id` 定位到对应 span
- [ ] 有至少 4 条告警规则，每条能说清阈值怎么来的
- [ ] 指标标签基数受控，有「禁止进指标的标签」清单
- [ ] 限流、熔断、降级、隔离四项至少三项有实现与压测/故障验证
- [ ] 网关使用 `Rewrite` 而非 `Director`
- [ ] `helm lint` 与 `helm template` 通过；探针与资源参数有推导理由
- [ ] 滚动更新期间零 5xx，且完成过一次回滚演练
- [ ] hw11–hw16 自评均 ≥70 分

---

## 10. 资源

官方：

- gRPC 官方文档：<https://grpc.io/docs/>
- Protocol Buffers：<https://protobuf.dev/>
- Buf：<https://buf.build/docs/>
- OpenTelemetry Go：<https://opentelemetry.io/docs/languages/go/>
- Prometheus：<https://prometheus.io/docs/>
- Kubernetes：<https://kubernetes.io/docs/>
- Helm：<https://helm.sh/docs/>
- Docker 多阶段构建：<https://docs.docker.com/build/building/multi-stage/>

书籍：

- 《Cloud Native Go》——Matthew A. Titmus。云原生模式与 Go 的对应。
- 《Kubernetes 权威指南》——龚正等。中文，运维视角。
- 《Designing Data-Intensive Applications》——Martin Kleppmann。本阶段读复制、分区、一致性与流处理章节。
- 《微服务架构设计模式》——Chris Richardson。Saga 与事件驱动的参考。

---

## 11. 本阶段的典型风险

| 风险 | 表现 | 对策 |
| --- | --- | --- |
| 为了微服务而微服务 | 两个服务共用一个数据库、互相直接读表 | 先明确数据边界；共享数据库反模式必须在 hw12 的 README 里显式排除 |
| 消息丢了才发现 | 先发消息后落库，或用「尽力而为」的投递 | Outbox 是硬性要求，测试必须覆盖「消息投递失败后重试成功」 |
| 可观测性变成堆工具 | 装了 Prometheus 和 Grafana，但没有一条能定位问题的证据链 | 交付物必须是「一次跨服务请求的完整证据链」，不是截图 |
| 限流把正常用户挡了 | 限流键用 IP 且未考虑 NAT | 限流键设计要写理由；必须有「阈值如何得出」的压测数据 |
| K8s 当成黑盒 | 参数照抄，OOMKilled 与 CPU 限流反复出现 | hw16 要求每个资源参数都有推导理由与验证记录 |
| 节奏被元旦打断 | 12-28 到 01-03 这一周实际只剩 4 天 | 提前把 Outbox 的设计想清楚，01-01 只复盘不推新内容 |
