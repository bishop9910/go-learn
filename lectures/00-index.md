# 讲义索引

- 讲义总数：**19 篇**
- 代码基线：**Go 1.25**（本机 `go1.25.7`）；标注「Go 1.26+」「Go 1.27+」的写法在 1.25 上不可编译
- 事实来源：所有版本相关陈述以 [`_meta/authoring-spec.md`](../_meta/authoring-spec.md) 收录的官方 Release Notes 为准

---

## 1. 全部讲义

| 编号                                           | 标题                              | 主要对应周次      | 解决的问题                                                                        |
| -------------------------------------------- | ------------------------------- | ----------- | ---------------------------------------------------------------------------- |
| [L01](01-toolchain-and-modules.md)           | 工具链与 Go Modules                 | W0–W1       | `go.mod` 的 `go` 行与 `toolchain` 行、依赖与代理、工作区、工具依赖、构建选项                         |
| [L02](02-language-essentials.md)             | 语言精要与易错点                        | W1–W2       | 切片别名、map 零值、接口 nil、方法集、泛型、循环变量等高频坑                                           |
| [L03](03-errors-and-logging.md)              | 错误处理与结构化日志                      | W1–W2       | 包装 / 判定 / 翻译三层、`panic` 边界、`log/slog` 用法与字段规范                                 |
| [L04](04-stdlib-core.md)                     | 标准库核心                           | W2          | `io` / `os`（含 `os.Root`）/ `strings` / `bytes` / `strconv` / `bufio` / `time` |
| [L05](05-serialization-json.md)              | 序列化：`encoding/json` 与 `json/v2` | W2、W5       | v1 的坑、可选字段建模、Go 1.27 起 v2 的严格默认与 tag 变化                                      |
| [L06](06-testing.md)                         | 测试工程                            | W2–W3、W9    | 表格驱动、`httptest`、benchmark、fuzz、`testing/synctest`                            |
| [L07](07-net-http.md)                        | `net/http` 与 Web 服务             | W3–W4       | 路由、中间件、超时、优雅关闭、客户端连接池、CSRF 防护                                                |
| [L08](08-database.md)                        | 数据访问                            | W3–W5       | `database/sql`、连接池、事务、迁移、`sqlc` vs GORM、索引与分页                                |
| [L09](09-redis-and-caching.md)               | Redis 与缓存设计                     | W5          | key 设计、TTL、Cache-Aside、穿透 / 击穿 / 雪崩、降级                                       |
| [L10](10-auth-and-jwt.md)                    | 认证授权与 JWT                       | W4          | 密码哈希、JWT 安全、双令牌、RBAC、越权防护                                                    |
| [L11](11-concurrency-and-context.md)         | 并发原语与 `context`                 | W7–W8       | channel / select / sync 原语、内存模型、context 纪律与泄漏                                |
| [L12](12-concurrency-patterns.md)            | 并发模式与并发安全                       | W7–W8       | Worker Pool、Pipeline、Fan-in/out、背压、限流熔断、并发数据结构                               |
| [L13](13-engineering.md)                     | 工程化                             | W4–W6、W10   | 项目结构、依赖注入、配置、`golangci-lint` v2、CI/CD、Docker                                 |
| [L14](14-performance.md)                     | 性能分析                            | W10、W22     | pprof、trace、`FlightRecorder`、逃逸分析、降分配、GC 调参                                  |
| [L15](15-grpc-and-protobuf.md)               | gRPC 与 Protobuf                 | W11         | proto3、`buf` 工具链、四种方法、拦截器、状态码与错误模型                                           |
| [L16](16-microservices-and-observability.md) | 微服务架构与可观测性                      | W12、W14     | 拆分方法、注册发现、配置中心、Prometheus / Grafana / OpenTelemetry                          |
| [L17](17-mq-and-kubernetes.md)               | 消息队列、容器化与 Kubernetes            | W13、W16     | MQ 选型与投递语义、Outbox、Docker、K8s 对象与探针、Helm                                      |
| [L18](18-runtime-internals.md)               | 运行时原理                           | W17–W24     | GMP 调度、栈与堆、GC、`sync.Pool`、内存模型                                               |
| [L19](19-distributed-systems.md)             | 分布式基础                           | W15、W22–W24 | CAP、Raft、分布式锁、一致性哈希、限流算法、分布式 ID                                              |

---

## 2. 推荐阅读顺序

按周计划走即可。如果只想补某一块，按下面的依赖关系选：

```text
L01 工具链 ─┬─ L02 语言精要 ─┬─ L04 标准库 ─┬─ L05 序列化
            │                │              │
            │                └─ L03 错误与日志 ─┬─ L13 工程化
            │                                   │
            └─ L06 测试 ────────────────────────┘
                        │
      L07 net/http ─────┼──── L08 数据访问 ──── L09 Redis 缓存
                        │            │
                        └─ L10 认证授权 ─┘
                                     │
                     L11 并发与 context ─── L12 并发模式
                                     │
                     L14 性能分析 ────┤
                                     │
                     L15 gRPC ────────┤
                                     │
                     L16 微服务与可观测性 ─── L17 MQ 与 K8s
                                     │
                     L18 运行时原理 ─── L19 分布式基础
```

---

## 3. 每篇讲义的固定结构

| 章节 | 内容 |
| --- | --- |
| 本篇定位 | 3–5 行：解决什么问题、适用哪些学习周、前置要求 |
| 正文分节 | 原理 + 可编译代码示例 + 版本差异标注 |
| 常见错误与反模式 | 表格：错误写法 / 现象 / 根因 / 正确做法 |
| 动手练习 | 3–5 个小练习，用于巩固 |
| 自检清单 | 可勾选项，全部打勾才算掌握 |
| 延伸阅读 | 官方链接为主；书籍只给书名、作者、出版社 |

---

## 4. 阅读方式建议

1. **不要通读。** 先看「本篇定位」和「自检清单」，判断是否已经掌握；已掌握的跳过。
2. **代码必须敲一遍。** 讲义中的示例以 Go 1.25 为基线，多数可以直接 `go run`。只读不敲等于没读。
3. **遇到版本标注要当真。** 标注「Go 1.27+」的写法在本机 1.25 上**编译不过**，这是预期行为，不是讲义写错了。
4. **把「常见错误与反模式」当检查表用。** 写完代码后对照一遍，能挡掉大部分低质量实现。
5. **讲义是参照，不是教程。** 主线是 `plan/` 里的周计划与 `homework/` 里的作业；讲义是卡住时查的。

---

## 5. 关于书籍

原始规划推荐的书（《Go 程序设计语言》《Go 语言实战》《Go 语言高级编程》《Go 语言设计与实现》《Go 语言学习笔记》《Concurrency in Go》《Cloud Native Go》）仍然有效，各阶段计划文件里给出了对应的阅读章节建议。

本仓库的讲义不替代这些书，而是做两件事：

1. 把书里的知识**对齐到 Go 1.25/1.26/1.27 的现状**（很多书的版本停留在 1.18–1.22，例如循环变量语义、`time.Timer` 通道、`ReverseProxy.Director`、`encoding/json` 实现都已变化）。
2. 把书里的知识**转成可交付的作业要求**，让你必须写出来而不是读过去。

书籍只写书名、作者、出版社，不提供任何下载链接。
