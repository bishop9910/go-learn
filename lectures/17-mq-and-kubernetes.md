# L17 消息队列、容器化与 Kubernetes 部署

（本篇定位：把「服务之间怎么异步协作」与「服务怎么被部署和运维」连成一条线。对应 W13（消息队列与最终一致性）与 W16（容器化与 Kubernetes 部署），前置为 [L16](16-microservices-and-observability.md) 的拆分方法论与可观测性。语言基线 Go 1.25，容器感知 `GOMAXPROCS` 等版本差异单独标注。）

## 1. 为什么用 MQ，以及它的代价

| 动机 | 得到什么 | 引入的代价 |
| --- | --- | --- |
| 解耦 | 生产者不需要知道消费者是谁、有几个 | 调用链从显式变成隐式，出问题时要额外查消费端 |
| 削峰 | 突发流量进入队列，消费者按自身能力处理 | 队列积压成为新的故障点，需要积压监控与扩容 |
| 异步 | 主链路只做必要工作，慢操作后台完成 | 用户看不到最终结果，需要状态查询与补偿 |
| 广播 | 一份事件多个下游各自消费 | 事件 schema 成为多方契约，变更成本上升 |

额外代价：一致性从「本地事务」退化为「最终一致」；顺序保证需要显式设计；重复投递必须由消费端处理；MQ 自身要部署、监控、升级；排障从单进程调试变成跨进程追踪。**如果一次本地调用 + 一个事务就能解决，不要引入 MQ。**

## 2. 选型对比

| 项目 | 吞吐与延迟 | 顺序保证 | 投递语义 | 生态与运维复杂度 |
| --- | --- | --- | --- | --- |
| `Kafka` | 极高吞吐，毫秒级延迟 | 单分区内严格有序 | 至少一次，配合事务可做到幂等写入 | 生态最丰富，运维最重（分区、副本、再平衡） |
| `RabbitMQ` | 中等吞吐，微秒到毫秒 | 单队列有序，多消费者竞争消费 | 至少一次，支持消息确认与死信 | 路由灵活（exchange/queue/binding），运维中等 |
| `NATS`（含 JetStream） | 高吞吐，极低延迟 | 单 subject / stream 内有序 | 核心 NATS 至多一次，JetStream 可持久化与重放 | 部署最轻，功能相对精简 |
| `Redis Stream` | 高吞吐，低延迟 | 单 stream 内有序 | 至少一次，靠消费者组与确认 | 复用已有 Redis，持久化与容量受 Redis 限制 |

Go 客户端模块路径（均以官方最新稳定版为准）：

| 目标 | 模块路径 |
| --- | --- |
| Kafka | `github.com/segmentio/kafka-go`、`github.com/twmb/franz-go`、`github.com/IBM/sarama` |
| RabbitMQ | `github.com/rabbitmq/amqp091-go` |
| NATS | `github.com/nats-io/nats.go` |

选型判据不是「谁的基准测试更快」，而是：需要长期保留与重放日志选 `Kafka`；需要复杂路由、优先级、延迟队列选 `RabbitMQ`；需要极轻量、低延迟的请求-响应与广播选 `NATS`；已经在用 Redis 且数据量可控时用 `Redis Stream`。

## 3. 投递语义与幂等消费

| 语义 | 真实含义 | 什么时候出现 |
| --- | --- | --- |
| at-most-once | 消息可能丢，不会重复 | 先提交位移再处理，或不做确认 |
| at-least-once | 消息不会丢，可能重复 | 先处理再提交位移；MQ 重试、再平衡、网络超时都会造成重复 |
| exactly-once | **通常靠消费端幂等实现**，不是协议天然保证 | 生产端事务写入 + 消费端去重键，端到端效果等价于一次 |

位移提交时机直接决定重复窗口：**先处理后提交** → 处理成功但提交前崩溃会重复消费（at-least-once）；**先提交后处理** → 崩溃时消息丢失（at-most-once）。因此工程上的标准答案是「按 at-least-once 设计 + 消费端幂等」。

| 幂等手段 | 实现 | 适用 |
| --- | --- | --- |
| 业务唯一键 + 去重表 | 用消息 id 或业务主键做唯一索引，插入冲突即已处理 | 通用，需要一次额外写入 |
| 状态机 + 版本号 | 只允许 `pending → paid → shipped` 的合法迁移，重复事件被拒绝 | 有明确状态的业务实体 |
| 条件更新 | `UPDATE ... WHERE status = ?`，受影响行数为 0 说明已被处理 | 库存扣减、余额变更 |

```sql
-- 去重表：唯一索引是幂等的核心，不能只靠应用层先查后写
CREATE TABLE consumed_message (
  message_id  VARCHAR(64) NOT NULL,
  consumer    VARCHAR(64) NOT NULL,
  consumed_at TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (message_id, consumer)
);

-- 库存扣减用条件更新，避免读出再写回造成的超卖
UPDATE inventory SET available = available - ?
WHERE sku = ? AND available >= ?;
```

## 4. 顺序性

| 问题 | 结论 |
| --- | --- |
| 顺序边界在哪 | 分区（Kafka partition）/ 队列 / stream 内部有序，**跨分区不保证** |
| 如何保证同一实体的顺序 | 用实体键（如 `order_id`）做分区键，同一实体的消息落到同一分区 |
| 消费并发与顺序的冲突 | 单分区多消费者会乱序；要并行又要顺序，就按 key 路由到固定 worker |
| 再平衡的影响 | 分区重新分配会让同一 key 短暂由不同消费者处理，需要处理交接期的重复 |

```go
// 示意：消费端按 key 路由到固定 worker，保证同一 key 串行、不同 key 并行
type keyRouter struct {
	workers []chan Message
}

func (r *keyRouter) dispatch(m Message) {
	idx := int(hashKey(m.Key) % uint64(len(r.workers)))
	r.workers[idx] <- m // 同一 key 永远落进同一个 channel
}
```

代价是吞吐受最热的 key 限制（倾斜）。若某个 key 的消息量远大于其他，说明分区键选错了，或该实体本身就应该是独立主题。

## 5. 可靠性

| 环节 | 手段 | 注意 |
| --- | --- | --- |
| 生产者 | 确认级别（acks）：全副本确认最安全但最慢；开启生产端重试 | 重试会放大重复，必须配合幂等消费 |
| 存储 | 持久化 + 多副本，副本数至少 3 | 单副本写入成功不等于消息安全 |
| 消费失败 | 有限次重试（指数退避 + 抖动）后进入死信队列 | 死信队列必须有人看，否则等于丢弃 |
| 毒消息 | 解析失败或永远失败的消息立即入死信，不参与重试计数 | 否则会卡住整个分区 |
| 积压 | 监控消费延迟（lag）与队列长度，按 lag 而非 CPU 扩容 | 积压告警要有「持续增长」条件，避免瞬时抖动 |
| 扩容上限 | 消费者数不能超过分区数，多的消费者会空转 | 扩容前先确认分区数够 |

重试与退避的细节：退避基数随重试次数指数增长，并加入随机抖动避免所有消费者同时重试造成二次峰值；重试队列与主队列分离，避免重试消息阻塞正常消息。

## 6. 最终一致性：Outbox 与 Saga

「分布式事务」多数是伪需求：业务真正需要的是「本地操作与消息投递不会只成功一半」。**Outbox 模式**把这两件事放回同一个本地事务：业务表与 outbox 表在同一事务内写入，再由独立的中继把 outbox 记录投递出去。

```sql
CREATE TABLE outbox (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  aggregate_id VARCHAR(64)     NOT NULL,      -- 聚合根 id，用于顺序性
  event_type   VARCHAR(64)     NOT NULL,
  payload      JSON            NOT NULL,
  created_at   TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
  published_at TIMESTAMP       NULL,          -- NULL 表示待投递
  PRIMARY KEY (id),
  KEY idx_unpublished (published_at, id)      -- 中继扫描用
);

-- 业务写入与 outbox 写入必须同一事务
BEGIN;
INSERT INTO orders (id, sku, amount_cents, status) VALUES (?, ?, ?, 'created');
INSERT INTO outbox (aggregate_id, event_type, payload)
VALUES (?, 'order.created', JSON_OBJECT('order_id', ?, 'amount_cents', ?));
COMMIT;
```

```go
// 中继 goroutine 骨架：批量取未投递记录，投递成功后标记
func (r *Relay) Run(ctx context.Context) {
	ticker := time.NewTicker(r.Interval)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			batch, err := r.store.FetchUnpublished(ctx, r.BatchSize) // WHERE published_at IS NULL ORDER BY id LIMIT ?
			if err != nil {
				r.log.Error("fetch outbox failed", slog.String("err", err.Error()))
				continue
			}
			for _, rec := range batch {
				// 以 id 作为消息 key，保证同一聚合根的事件顺序
				if err := r.producer.Publish(ctx, rec.AggregateID, rec.EventType, rec.Payload); err != nil {
					r.log.Warn("publish failed, will retry", slog.Int64("id", rec.ID))
					break // 保持顺序：失败即停止本轮，下轮从同一条继续
				}
				if err := r.store.MarkPublished(ctx, rec.ID); err != nil {
					// 投递成功但标记失败 → 下轮会重复投递，由消费端幂等兜住
					r.log.Error("mark published failed", slog.Int64("id", rec.ID))
				}
			}
		}
	}
}
```

中继本身必须容忍「至少一次」：投递成功但标记失败会导致重复投递，这正是第 3 节幂等消费要解决的问题。**Saga** 用于跨服务的长事务：编排式（orchestration）由一个协调者按顺序调用各服务并在失败时调用补偿接口；协同式（choreography）由各服务监听事件自行决定下一步。补偿事务同样必须幂等，否则重试补偿会造成重复退款一类的事故。

## 7. 事件设计

| 主题 | 约定 |
| --- | --- |
| 命名 | 用「已发生的事实」的过去式：`order.created`、`payment.settled`；不要用 `createOrder` 这种命令式命名 |
| 事件与命令的区别 | 事件是广播事实、可以有多个消费者；命令是点对点、有明确接收方与失败语义 |
| schema 演进 | 只加字段、不删字段、不改字段语义、不复用字段编号；消费端忽略未知字段 |
| payload 只放 id 还是放快照 | 只放 id → 消费端要回查，耦合且放大读压力；放快照 → 解耦但有数据陈旧风险。折中：放 id + 事件发生时的关键字段，消费端以快照为准、必要时回查 |
| 版本 | 消息头带 `event_version` 与 `occurred_at`，便于回溯与重放 |

## 8. Docker：分层、多阶段与镜像选择

镜像分层与缓存命中：Dockerfile 中每条指令生成一层，任一层变化会让其后所有层缓存失效。因此顺序是「先拷贝依赖描述文件 → 下载依赖 → 再拷贝源码 → 编译」，这样改业务代码不会重新下载依赖。`.dockerignore` 必须排除 `vendor`（如果不使用）、`.git`、构建产物与本地配置，否则镜像上下文巨大或把密钥打进镜像。

```dockerfile
# 构建阶段
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download            # 依赖层：go.mod/go.sum 不变则命中缓存
COPY . .
# CGO_ENABLED=0 得到静态二进制，才能放进 scratch/distroless
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

# 运行阶段
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

| 基础镜像 | 体积 | 攻击面 | 注意 |
| --- | --- | --- | --- |
| `scratch` | 最小 | 最小 | 无 CA 证书、无时区库、无 shell，需自己拷贝证书与 `/usr/share/zoneinfo` |
| `distroless` | 小 | 很小 | 无 shell，调试只能用 `kubectl debug` 的临时容器；`static` 变体适合纯 Go 静态二进制 |
| `alpine` | 较小 | 较小 | 用 musl libc：启用 CGO 时可能遇到 DNS 解析与 `getaddrinfo` 行为差异；需要 `ca-certificates` 与 `tzdata` |

CGO 相关判断：用了 `net`、`os/user` 等包并且 `CGO_ENABLED=1` 时，二进制依赖 glibc/musl，不能直接放进 `scratch`。要么 `CGO_ENABLED=0` 静态编译，要么让运行镜像与构建镜像的 libc 一致。时区问题的典型症状是日志时间全是 UTC——拷贝 `tzdata` 或显式设置 `TZ`。非 root 运行与资源限制是默认要求，不是可选项。

## 9. `compose.yaml` 编排

```yaml
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: devroot
      MYSQL_DATABASE: appdb
    volumes: ["mysql-data:/var/lib/mysql"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

  app:
    build: .
    environment:
      DB_DSN: app:app@tcp(mysql:3306)/appdb
      REDIS_ADDR: redis:6379
    ports: ["8080:8080"]
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }

  prometheus:
    image: prom/prometheus:latest
    volumes: ["./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro"]
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]

volumes:
  mysql-data:
```

`depends_on` 只保证**启动顺序**，不保证依赖可用：容器进程起来了但数据库还没接收连接是常态。因此必须配 `healthcheck` 并用 `condition: service_healthy`；即便如此，应用侧仍要有连接重试与启动探针，不能假设依赖永远就绪。

## 10. Kubernetes 核心对象

| 对象 | 作用 | 常见误用 |
| --- | --- | --- |
| `Pod` | 最小调度单元，一个或多个容器 | 直接创建裸 Pod，重启后不重建 |
| `Deployment` | 无状态副本集与滚动更新 | 用它管理有状态服务 |
| `Service` | 稳定的虚拟 IP 与负载均衡 | 类型选错（该用 Headless 时用 ClusterIP） |
| `Ingress` | 七层入口与 TLS 终止 | 把鉴权、限流全部塞进 Ingress 注解 |
| `ConfigMap` | 非敏感配置 | 把密钥放进去 |
| `Secret` | 敏感数据 | 以为 Secret 默认加密（默认只做 base64 编码） |
| `Job` / `CronJob` | 一次性与定时任务 | 用 Deployment 跑批处理，导致任务被反复重启 |
| `HPA` | 按指标水平扩缩容 | 没有配 `requests` 就设 CPU 目标，导致扩容失效 |
| `StatefulSet` | 稳定网络标识与有序部署 | 无状态服务也用它，白增复杂度 |
| `PVC` | 持久卷声明 | 忘记设置 `storageClassName` 与回收策略，数据随 Pod 一起消失 |

### 10.1 探针

| 探针 | 回答的问题 | 失败后果 | 正确设置 |
| --- | --- | --- | --- |
| `livenessProbe` | 进程是否还活着、需不需要重启 | 连续失败即重启容器 | 只检查进程自身（如内部健康端点），**绝不检查外部依赖** |
| `readinessProbe` | 现在能不能接流量 | 从 Service 端点摘除，不重启 | 检查本地就绪状态（依赖连接是否建立、配置是否加载完成） |
| `startupProbe` | 启动是否完成 | 未通过前不执行另两个探针 | 给慢启动服务设较长的 `failureThreshold`，避免启动期被误杀 |

最常见的雪崩是 **readiness 配错**：把 readiness 指向「数据库是否可写」。数据库抖动时所有副本同时被摘除，Service 端点为 0，流量瞬间全部失败；恢复后所有副本又同时回来，形成惊群。正确做法是 readiness 只反映本进程能否处理请求（连接池是否已建好、是否处于优雅退出），把依赖故障交给熔断与降级处理。另一个高频错误是把 liveness 写成依赖检查，导致数据库抖动时全量 Pod 被反复重启。

### 10.2 资源限制与 Go 运行时

必须同时设置 `requests` 与 `limits`：`requests` 决定调度与 HPA 基准，`limits` 决定被限流/被杀的上限。Go 服务有两类经典不匹配：

| 问题 | 症状 | 解决 |
| --- | --- | --- |
| `GOMAXPROCS` 与 CPU limit 不匹配 | 容器 limit 为 2 核，但 Go 看到宿主机 64 核 → 64 个 P 争抢 2 核配额 → cgroup CPU throttling → P99 延迟尖刺 | **Go 1.25 起容器感知 `GOMAXPROCS`**：Linux 上会考虑 cgroup CPU bandwidth limit，并在运行期周期性更新；可用 `GODEBUG=containermaxprocs=0` 与 `GODEBUG=updatemaxprocs=0` 分别关闭「容器感知」与「周期更新」，也可用 1.25 新增的 `runtime.SetDefaultGOMAXPROCS` 显式设定 |
| 内存 limit 与 GC 目标不匹配 | 内存持续增长直到 `OOMKilled`（退出码 137） | 用 1.19 起的 `GOMEMLIMIT`（或 `runtime/debug.SetMemoryLimit`）设置软上限，通常取 limit 的 70%–80%，给运行时与栈留余量 |

注意 `GOMEMLIMIT` 是**软上限**：它让 GC 更早启动，不阻止分配超过上限，因此仍需按实际堆峰值设 limit。`OOMKilled` 的排查顺序：`kubectl describe pod` 看 `Last State` 与退出码 → 看容器内存工作集曲线是否贴近 limit → 用 `GODEBUG=gctrace=1` 或运行时指标看 GC 是否跟不上分配速度。

### 10.3 优雅退出与配置注入

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]   # 等端点摘除生效，再让应用停止
      readinessProbe:
        httpGet: { path: /readyz, port: 8080 }
        periodSeconds: 5
        failureThreshold: 2
```

退出顺序是：Pod 被标记 Terminating → 端点摘除（与下一步并发）→ `preStop` 执行 → 发送 `SIGTERM` → 应用停止接收新请求并等待在途请求结束 → 超过 `terminationGracePeriodSeconds` 则 `SIGKILL`。应用内必须实现「收到信号后 Shutdown」；`preStop` 的 sleep 是为了错开「端点摘除」与「进程开始拒绝请求」这两个动作的竞态。

| 配置注入方式 | 优点 | 缺点 |
| --- | --- | --- |
| 环境变量（`envFrom`） | 简单、无需改代码 | 变更需重启 Pod；不适合长配置 |
| 挂载 `ConfigMap` 为文件 | 可挂载完整配置文件，支持热更新（kubelet 同步有延迟） | 应用要自己监听文件变化；subPath 挂载不会热更新 |
| 挂载 `Secret` 为文件 | 权限可控、不进入环境变量 | 同样有同步延迟 |
| 外部配置中心 | 版本、灰度、回滚能力完整 | 多一个依赖，需处理启动期不可用 |

## 11. Helm

Helm 把一组 Kubernetes 清单参数化成一个可复用、可版本化的 Chart：`Chart.yaml`（元信息与版本）、`values.yaml`（默认值）、`templates/`（带模板语法的清单）。环境差异通过多份 values 文件覆盖（`values-dev.yaml`、`values-prod.yaml`），而不是复制 Chart。

```text
mychart/
  Chart.yaml
  values.yaml
  values-prod.yaml
  templates/
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    _helpers.tpl
```

```yaml
# Chart.yaml
apiVersion: v2
name: quote-api
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: registry.example.com/quote-api
  tag: ""                    # 留空则用 .Chart.AppVersion，CI 中由 --set 覆盖为 git sha
  pullPolicy: IfNotPresent
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 1,    memory: 512Mi }
config:
  logLevel: info
ingress:
  enabled: true
  host: quote.example.com
```

```yaml
# templates/deployment.yaml（节选）
apiVersion: apps/v1
kind: Deployment
metadata: { name: {{ include "quote-api.fullname" . }} }
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          envFrom: [{ configMapRef: { name: {{ include "quote-api.fullname" . }}-config } }]
          resources: {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe: { httpGet: { path: /readyz, port: 8080 } }
---
# templates/service.yaml
apiVersion: v1
kind: Service
metadata: { name: {{ include "quote-api.fullname" . }} }
spec:
  selector: { app.kubernetes.io/name: quote-api }
  ports: [{ name: http, port: 80, targetPort: 8080 }]
---
# templates/ingress.yaml（.Values.ingress.enabled 为假时不渲染）
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: {{ include "quote-api.fullname" . }} }
spec:
  rules: [{ host: {{ .Values.ingress.host | quote }}, http: { paths: [{ path: /, pathType: Prefix, backend: { service: { name: quote-api, port: { number: 80 } } } }] } }]
{{- end }}
---
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: {{ include "quote-api.fullname" . }}-config }
data: { LOG_LEVEL: {{ .Values.config.logLevel | quote }} }
```

工作流固定为：`helm lint` 检查模板与 values 结构 → `helm template --values values-prod.yaml` 渲染出真实清单并人工复核 → `helm upgrade --install --atomic` 发布，失败时自动回滚到上一个 revision。

## 12. CI/CD 到部署

| 环节 | 要求 |
| --- | --- |
| 镜像 tag | **不要用 `latest`**：用 git sha 或语义化版本（不可变），否则回滚无法定位到底跑了哪个提交 |
| 多架构 | 需要 arm64 节点时用 buildx 构建多平台镜像并在清单中固定 digest |
| 滚动更新 | `maxSurge` / `maxUnavailable` 与就绪探针配合，确保任何时刻都有可用副本 |
| 回滚 | `kubectl rollout undo deployment/<name>`；`helm rollback <release> <revision>` |
| 数据库变更 | 与应用解耦（先加列后写、先双写后切换），保证新旧版本可同时运行 |
| 渐进式发布 | 蓝绿：两套环境切流量，回滚是切回；金丝雀：先给少量流量，观察错误率与延迟指标后再放大 |

发布前必须能回答：这次的镜像 digest 是什么、上一个可用 revision 是哪个、回滚命令是什么、回滚需要多久。

## 13. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 用 MQ 做一次简单的方法调用 | 系统复杂度和延迟同时上升 | 为异步而异步 | 同步调用能解决就不引入 MQ |
| 先提交位移再处理消息 | 崩溃时消息丢失 | 位移提前于业务提交 | 先处理再提交，用消费端幂等兜住重复 |
| 只靠「先查后写」实现幂等 | 并发下仍然重复处理 | 检查与写入不是原子的 | 用唯一索引或条件更新 |
| 用随机分区键 | 同一订单的事件乱序 | 顺序边界被破坏 | 用实体键做分区键，按 key 路由 worker |
| 无死信队列 | 毒消息卡住整个分区 | 失败消息无限重试 | 有限重试 + 死信队列 + 有人消费死信 |
| 在业务事务外发消息 | 业务成功但消息丢失，或反之 | 双写没有原子性 | Outbox：业务表与 outbox 同事务写入 |
| 镜像里用 `latest` 且无 digest | 回滚后行为不一致 | tag 可变 | 用不可变 tag 或 digest |
| readiness 检查数据库可写 | 依赖抖动导致全量副本被摘除 | 就绪判定越界 | readiness 只反映本进程是否可服务 |
| liveness 检查外部依赖 | 依赖抖动导致 Pod 被反复重启 | 探针职责混淆 | liveness 只检查进程自身 |
| 只设 `limits` 不设 `requests` | HPA 无法计算利用率、调度不合理 | 缺少基准 | 两者都设，requests 反映常态用量 |
| CPU limit 与 `GOMAXPROCS` 不匹配 | P99 延迟尖刺、CPU throttling | Go 看到的核数多于配额 | Go 1.25 容器感知 `GOMAXPROCS`，或显式设置 |
| 用 `depends_on` 当健康检查 | 应用启动即连不上数据库并退出 | 只保证启动顺序 | 配 `healthcheck` + 应用侧连接重试 |
| 忽略 `terminationGracePeriodSeconds` | 发布时出现 502 | 进程在在途请求结束前被杀 | `preStop` + 应用内优雅退出 + 合理的宽限期 |

## 14. 动手练习

1. **Outbox 落地**：为一张业务表加 outbox 表，实现中继 goroutine 与消费端去重，然后用「杀掉进程再重启」验证：不丢事件，且重复事件不会造成重复副作用。
2. **顺序性验证**：用 `order_id` 作分区键，故意让同一订单产生 100 条事件，验证消费端处理顺序与生产顺序一致；再把分区键换成随机值，观察乱序率。
3. **积压与扩容**：用限速消费者制造积压，观察 lag 曲线，按 lag 扩到分区数上限，记录吞吐提升比例与不再提升的原因。
4. **镜像瘦身**：为同一个 Go 服务分别用 `golang:1.25` 基础镜像、`CGO_ENABLED=0` + `scratch`、`distroless` 构建，记录镜像大小、是否能解析 HTTPS、日志时区是否正确。
5. **探针与优雅退出**：把 readiness 故意配成依赖检查，在依赖重启期间观察 Service 端点数量变化与错误率；再改成正确的探针并加 `preStop`，对比滚动更新期间的 5xx 数量。
6. **资源与运行时**：给容器设 `limits.cpu=1` 与 `limits.memory=256Mi`，分别在「设置 `GOMAXPROCS`」和「不设置」两种情况下压测，记录 CPU throttling 比例与 P99；再把内存 limit 压到会触发 `OOMKilled`，用 `GOMEMLIMIT` 修正后复测。
7. **Helm 与回滚**：把上面的最小 Chart 写出来，用 `helm template` 复核渲染结果，发布一个坏版本（如镜像 tag 不存在），用 `--atomic` 观察自动回滚，并手动执行一次 `kubectl rollout undo`。

## 15. 自检清单

- [ ] 能说明每个 MQ 使用点是解耦、削峰、异步还是广播需求，且同步方案确实不合适。
- [ ] 消费端实现了幂等（唯一索引或条件更新），分区键与顺序边界明确。
- [ ] 死信队列存在且有人消费；积压监控基于 lag，扩容上限受分区数约束。
- [ ] 跨服务一致性用 Outbox / Saga，不依赖跨库事务；事件命名是过去式事实，schema 只做兼容的加字段。
- [ ] 镜像 tag 不可变，回滚命令与上一个可用 revision 明确。
- [ ] 三类探针职责正确：liveness 只看进程，readiness 只看本地可服务状态。
- [ ] `requests` / `limits` 都设置，且 `GOMAXPROCS` 与 `GOMEMLIMIT` 与 limit 匹配。
- [ ] 应用实现了优雅退出，发布期间不产生可观测的 5xx；Chart 参数化完整，`helm upgrade --atomic` 可用。

## 16. 延伸阅读

- Kubernetes 探针文档：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>；资源管理与 HPA：<https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>；Helm 官方文档：<https://helm.sh/docs/>
- 《Kubernetes 权威指南》，作者：龚正等，出版社：电子工业出版社；《Designing Data-Intensive Applications》，作者：Martin Kleppmann，出版社：O'Reilly Media
- microservices.io 的 Saga 与 Outbox 模式说明：<https://microservices.io/patterns/data/saga.html>；Go 官方 Release Notes：<https://go.dev/doc/go1.25>、<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>
- 相关讲义：[L16 微服务架构与可观测性](16-microservices-and-observability.md)、[L18 运行时原理](18-runtime-internals.md)、[L19 分布式基础](19-distributed-systems.md)
