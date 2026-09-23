# Go 后端进阶学习计划（2026-09-22 起）

本仓库是一套可直接执行的 Go 学习方案，包含**周计划**、**作业**与**讲义**三部分。

- 起点：**2026-09-22（周二）**，即开营日
- 终点：**2027-03-28（周日）**，作品集收口
- 总跨度：**25 个学习周**（含开营周 W0 与收口周 W25），约 6 个月
- 语言基线：**Go 1.25**（本机已安装 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`），覆盖到 **Go 1.27**
- 假期：**2026-09-25 至 2026-10-07 不安排任何任务**（用户指定）；其余法定假日按「降强度 / 机动」处理

---

## 一、这个计划解决什么问题

原始规划（见 `go-learn..md`）给出了「先快速上手 → 再项目落地 → 再进阶工程化 → 再底层原理 / 云原生 / 微服务」的五阶段路线，但它是一份**主题清单**，没有落到日期，也没有作业与讲义。

本仓库把它落成三件东西：

| 层 | 位置 | 作用 |
| --- | --- | --- |
| 计划 | `plan/` | 把五阶段拆成 25 个学习周，每周有日期、主题、每日任务、产出与验收 |
| 作业 | `homework/` | 21 份可独立交付的作业，每份有需求编号、验收命令与评分表 |
| 讲义 | `lectures/` | 19 篇配套讲义，覆盖计划中出现的全部技术主题，代码基线为 Go 1.25+ |

---

## 二、目录结构

```text
go-learn/
├── README.md                      本文件：总入口
├── go-learn..md                    原始规划（输入资料，未改动）
├── _meta/
│   └── authoring-spec.md           编写规范与 Go 1.25/1.26/1.27 事实基线
├── plan/
│   ├── 00-overview.md              总览：完整周历、里程碑、作业节奏、假期处理
│   ├── 01-stage1-foundation.md     阶段一：基础补全（W1–W2）
│   ├── 02-stage2-backend.md        阶段二：后端项目落地（W3–W6）
│   ├── 03-stage3-concurrency.md    阶段三：并发与工程化（W7–W10）
│   ├── 04-stage4-microservices.md  阶段四：微服务与云原生（W11–W16）
│   └── 05-stage5-runtime.md        阶段五：底层原理与性能优化（W17–W25）
├── homework/
│   ├── 00-index.md                 作业总览、评分标准、提交规范
│   ├── hw00-environment.md         W0  环境、工具链与基线自测
│   ├── hw01-cli-skeleton.md        W1  多包 CLI 骨架
│   ├── hw02-stdlib-cli.md          W1–W2 标准库综合 CLI
│   ├── hw03-sql-crud.md            W2–W3 database/sql CRUD 与迁移
│   ├── hw04-gin-user-api.md        W3–W4 用户系统 API 与 JWT
│   ├── hw05-blog-redis.md          W5  博客系统与 Redis 缓存
│   ├── hw06-backend-hardening.md   W6  Docker、接口文档、集成测试
│   ├── hw07-concurrency-patterns.md W7  并发模式
│   ├── hw08-context-refactor.md    W8  context 治理与并发安全
│   ├── hw09-testing-engineering.md W9  测试工程
│   ├── hw10-pprof-optimization.md  W10 pprof 性能调优与工程化收口
│   ├── hw11-grpc-protobuf.md       W11 gRPC 与 Protobuf
│   ├── hw12-discovery-config.md    W12 服务发现与配置中心
│   ├── hw13-mq-consistency.md      W13 消息队列与最终一致性
│   ├── hw14-observability.md       W14 可观测性三支柱
│   ├── hw15-ratelimit-gateway.md   W15 限流熔断与网关
│   ├── hw16-k8s-deploy.md          W16 Kubernetes 与 CI/CD
│   ├── hw17-scheduler-gc.md        W17–W19 调度器与 GC 实验报告
│   ├── hw18-source-reading.md      W20–W21 源码阅读报告
│   ├── hw19-distributed-practice.md W22–W24 分布式实战
│   └── hw20-portfolio.md           W23–W25 作品集收口
└── lectures/
    ├── 00-index.md                 讲义索引与阅读顺序
    ├── 01-toolchain-and-modules.md
    ├── 02-language-essentials.md
    ├── 03-errors-and-logging.md
    ├── 04-stdlib-core.md
    ├── 05-serialization-json.md
    ├── 06-testing.md
    ├── 07-net-http.md
    ├── 08-database.md
    ├── 09-redis-and-caching.md
    ├── 10-auth-and-jwt.md
    ├── 11-concurrency-and-context.md
    ├── 12-concurrency-patterns.md
    ├── 13-engineering.md
    ├── 14-performance.md
    ├── 15-grpc-and-protobuf.md
    ├── 16-microservices-and-observability.md
    ├── 17-mq-and-kubernetes.md
    ├── 18-runtime-internals.md
    └── 19-distributed-systems.md
```

> 路径一律使用 ASCII，目的是避免跨平台编码问题与 Markdown 链接失效；标题与正文全部为简体中文。

---

## 三、怎么用

### 每天

1. 打开 `plan/00-overview.md`，找到当前日期所在的周次。
2. 打开该阶段的周计划文件，看当天的「主题 / 任务 / 产出」。
3. 需要补背景时打开对应讲义（每周计划的「配套讲义」列已给出编号）。
4. 在 `LEARNING-LOG.md` 里写三行：学了什么、卡在哪、明天做什么。

### 每周日

1. 对照本周的「验收清单」逐条打勾，没过的项当天补齐或明确顺延到下周（顺延必须写进日志，不允许静默跳过）。
2. 若有作业到期，按 `homework/00-index.md` 的提交规范提交。

### 遇到作业

作业不是「做完就行」，而是按 `homework/00-index.md` 的评分表自评。**低于 70 分视为未通过**，需要重做未达标项。每份作业都给出了「参考实现要点」，但那是给卡住时看的，先自己写。

---

## 四、假设与调整方式

计划建立在一组明确假设上。如果实际情况不同，按下表调整，不要重写整个计划。

| 假设 | 取值 | 如果不符合 |
| --- | --- | --- |
| 每日投入 | 工作日 2–3 小时，周末 4–6 小时 | 只有工作日晚上：把每周的「周末整合」任务平摊到工作日，作业交付期整体后移 50% |
| 起点基础 | 已会 Go 基础语法与少量标准库，能读能改，没系统做过项目 | 若语法不熟，先用 W1–W2 并行补 `lectures/02-language-essentials.md`，作业 01 放宽到 W2 交付 |
| 开发环境 | Windows + PowerShell，可装 Docker | 若不能装 Docker，阶段二的 MySQL/Redis 改用云端免费实例；阶段四的 K8s 用 kind 或跳过 hw16 的集群部分保留 Helm 渲染验收 |
| 网络 | 可访问 `proxy.golang.org` 或配置了国内代理 | 若代理受限，配置 `GOPROXY` 并**固定 `GOTOOLCHAIN=local`**，避免 go 命令尝试联网下载工具链而失败 |
| 目标方向 | Go 后端求职 / 工程能力补齐（非云原生专精） | 若专攻云原生，把 W11–W16 扩展为 10 周，压缩 W17–W24 为 4 周专题 |

**进度落后时的取舍顺序**（从先砍到后砍）：加分项 → 阶段五的第二个实验 → 阶段四的 hw16 集群部分 → 阶段三的 benchmark 深度。**不要砍**：错误处理、测试、pprof、context 生命周期——这四项是阶段三之后所有内容的公共基础。

---

## 五、版本策略

| 时间 | Go 版本 | 本计划中的处理 |
| --- | --- | --- |
| 现在（2026-09） | **1.25.7（本机已装）** | 基线。所有讲义示例以 1.25 语法为准 |
| 2026-10 ~ 2026-12 | 1.25.x / 1.26.x | 阶段一至三。可在 W6 前后升级到 1.26，用 `go fix` 的 modernizers 批量现代化代码 |
| 2027-01 ~ 2027-03 | 1.26.x / 1.27.x | 阶段四、五。建议升级到 1.27，用上泛型方法与标准库 `uuid` |

要点：

- **基线是 1.25，凡是标注「Go 1.26+」「Go 1.27+」的写法在 1.25 上不可编译**，讲义中已逐处标注。
- 升级前先读对应的 Release Notes（`https://go.dev/doc/go1.26`、`https://go.dev/doc/go1.27`），重点看 Go 1 兼容性承诺之外的**行为变化**：Go 1.25 修复了 1.21 引入的 nil 检查延迟 bug（原本能跑的错误代码现在会 panic）；Go 1.26 起 `ServeMux` 尾斜杠重定向由 301 改为 307、`net/http/httputil.ReverseProxy.Director` 废弃；Go 1.27 起 `encoding/json` 由 v2 实现支撑、错误文本可能不同且默认更严格。
- `go.mod` 的 `go` 行是**语言版本**而不是工具链版本。Go 1.26 起 `go mod init` 默认写 `go 1.(N-1).0`，这是有意为之（鼓励生成对当前受支持版本兼容的模块）。
- 网络受限时固定 `GOTOOLCHAIN=local`，否则 `go.mod` 要求更高版本时 go 命令会尝试下载工具链并卡住。

---

## 六、快速导航

| 我想…… | 去看 |
| --- | --- |
| 知道今天该学什么 | [`plan/00-overview.md`](plan/00-overview.md) |
| 看某个阶段的完整安排 | [`plan/`](plan/) 下的阶段文件 |
| 领作业、看评分标准 | [`homework/00-index.md`](homework/00-index.md) |
| 补某个知识点的背景 | [`lectures/00-index.md`](lectures/00-index.md) |
| 核对某个 Go 特性属于哪个版本 | [`_meta/authoring-spec.md`](_meta/authoring-spec.md) |
| 回到最初的规划文本 | [`go-learn..md`](go-learn..md) |

---

## 七、产出与验收总账

| 阶段 | 周次 | 日期 | 作业 | 阶段验收 |
| --- | --- | --- | --- | --- |
| 开营 | W0 | 09-22 ~ 09-24 | hw00 | 环境可复现，基线自测全过 |
| 一 基础补全 | W1–W2 | 10-08 ~ 10-18 | hw01、hw02 | 能写出多包、有测试、有结构化日志、能 `go build` 出二进制的 CLI |
| 二 后端项目落地 | W3–W6 | 10-19 ~ 11-15 | hw03–hw06 | 一个可部署的后端服务：MySQL + Redis + JWT + 文档 + 集成测试 + Docker |
| 三 并发与工程化 | W7–W10 | 11-16 ~ 12-13 | hw07–hw10 | 并发可控、`-race` 全绿、有 benchmark 与 pprof 优化报告、质量门禁全绿 |
| 四 微服务与云原生 | W11–W16 | 12-14 ~ 2027-01-24 | hw11–hw16 | 2–3 个服务、gRPC + 服务发现 + MQ + 可观测 + K8s 部署 |
| 五 底层原理与优化 | W17–W24 | 01-25 ~ 03-21 | hw17–hw19 | 能解释 GMP / GC、能读关键源码、能定位泄漏、能实现分布式基础组件 |
| 收口 | W23–W25 | 03-08 ~ 03-28 | hw20 | 一个能放进简历、一条命令可起的作品集项目 |
