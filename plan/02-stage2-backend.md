# 阶段二：后端项目落地（W3–W6）

- 时间：**2026-10-19（周一）至 2026-11-15（周日）**，共 4 周
- 对应原始规划：阶段 2「后端项目落地」，建议周期 4–6 周，本计划取 4 周
- 作业：hw03（10-25）、hw04（11-01）、hw05（11-08）、hw06（11-15）
- 阶段门：G2

---

## 1. 这一阶段要解决什么

从「会写 Go 程序」到「能交付一个后端服务」。判定标准不是「接口能通」，而是下面五条同时成立：

| 目标 | 判定标准 |
| --- | --- |
| 数据层靠得住 | 有迁移、有索引、有连接池推导、事务能正确回滚、错误能映射为领域错误 |
| 接口层成体系 | 路由清晰、中间件有序、统一错误响应、参数校验完整、401 与 403 语义正确 |
| 安全不靠运气 | 密码自适应哈希、JWT 算法固定且校验 `exp`、资源级授权防 IDOR、CORS 白名单 |
| 缓存有设计 | 有 key 设计表、有 TTL 策略、能说清一致性方案与三大缓存问题的处理位置 |
| 交付可复现 | `docker compose up` 一条命令起全栈、接口文档与实现一致、集成测试覆盖核心链路 |

---

## 2. W3：数据访问层（2026-10-19 ~ 10-25）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 10-19 | 一 | 表设计与迁移 | 设计 `users` 表；写可重复执行、可回滚的迁移文件；理解「先加列后删列」的兼容原则 | 迁移文件 + ER 说明 |
| 10-20 | 二 | `database/sql` 与连接池 | `DB`/`Stmt`/`Tx`/`Rows` 职责；连接池四个参数的推导；所有查询带 `ctx` | 仓储层雏形 |
| 10-21 | 三 | 仓储分层与错误映射 | 仓储接口 + MySQL 实现；`sql.ErrNoRows` 判定；唯一键冲突 → 领域错误 `ErrEmailExists` | 依赖倒置的仓储层 |
| 10-22 | 四 | 事务 | `BeginTx` + `defer tx.Rollback()`；写一个「失败整体回滚」的测试；理解长事务与事务内网络调用的危害 | 事务用例 + 测试 |
| 10-23 | 五 | 索引与分页 | 联合索引最左前缀、覆盖索引、深分页问题；`EXPLAIN` 验证；游标分页 | 索引说明 + `EXPLAIN` 记录 |
| 10-24 | 六 | 作业主体 | 用 `net/http` 写一个薄接口层驱动 CRUD；补集成测试 | hw03 主体完成 |
| 10-25 | 日 | 收口 | 补文档（连接池推导、索引理由）；`go test -race`；自评打分 | **hw03 交付** |

配套讲义：[L08 数据访问](../lectures/08-database.md)、[L02 语言精要](../lectures/02-language-essentials.md)（接口与方法集部分）

---

## 3. W4：Web 服务与认证授权（2026-10-26 ~ 11-01）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 10-26 | 一 | 路由与中间件 | `http.Handler` 链式组合；**Go 1.22 起的增强 `ServeMux` 路由**（方法与通配符、`r.PathValue`）；中间件顺序为什么重要 | 路由表 + 中间件骨架 |
| 10-27 | 二 | 请求解析与统一错误 | `http.MaxBytesReader` 限制体积；JSON 严格解码；统一错误响应 `{code, message, request_id}`；错误码表 | 错误模型 + 校验层 |
| 10-28 | 三 | 超时与优雅关闭 | 服务器各超时字段；`signal.NotifyContext` + `Shutdown(ctx)`；请求 ID 中间件 | 可优雅退出的服务 |
| 10-29 | 四 | 密码与注册登录 | `golang.org/x/crypto/bcrypt` 自适应哈希（模块路径，以官方最新稳定版为准）；登录失败信息不区分「用户不存在」与「密码错误」 | 注册/登录接口 |
| 10-30 | 五 | JWT 与鉴权中间件 | `github.com/golang-jwt/jwt/v5` 模块路径；固定算法、校验 `exp`/`iss`/`aud`；access + refresh 双令牌与轮换；身份写入 `context`（自定义 key 类型） | 鉴权中间件 + 刷新流程 |
| 10-31 | 六 | 作业主体与安全加固 | 覆盖「过期 token / 篡改签名 / 算法混淆」负向用例；**Go 1.25 的 `net/http.CrossOriginProtection`** 在一个路由组启用；CORS 白名单 | hw04 主体完成 |
| 11-01 | 日 | 收口 | 补齐 curl 端到端脚本与 README；自评打分 | **hw04 交付** |

配套讲义：[L07 net/http](../lectures/07-net-http.md)、[L10 认证授权与 JWT](../lectures/10-auth-and-jwt.md)

安全红线（本阶段起长期有效）：

1. 密码只存自适应哈希，禁止明文与可逆加密。
2. **密码不允许经明文 HTTP 传输**：本地开发也要明确标注「仅限 localhost」，生产必须 TLS。
3. Token 不得写入日志；日志中的敏感字段必须脱敏。
4. 每个「按 id 取资源」的接口都要做资源级归属校验，防水平越权（IDOR）。

---

## 4. W5：Redis 与缓存设计（2026-11-02 ~ 11-08）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 11-02 | 一 | Redis 基础 | 五种基础结构 + ZSet / Bitmap / HyperLogLog / Stream 的后端用途对照；`github.com/redis/go-redis/v9` 接入（模块路径，以官方最新稳定版为准） | Redis 接入 + 连通性测试 |
| 11-03 | 二 | key 设计与 TTL | 命名空间分层、长度控制、避免大 key 与热 key；TTL 必须显式设置；**TTL 随机抖动**避免同时过期 | key 设计表 |
| 11-04 | 三 | Cache-Aside 与一致性 | 完整时序图；「先更新 DB 再删缓存」vs 反向的对比分析；不一致窗口的兜底 | 缓存层实现 + 时序图 |
| 11-05 | 四 | 三大缓存问题 | 穿透（空值缓存 + 短 TTL）、击穿（`golang.org/x/sync/singleflight` 互斥重建）、雪崩（抖动 + 多级 + 限流） | 三类问题的实现位置说明 |
| 11-06 | 五 | 降级与可观测 | Redis 不可用时回源而不是 500；命中率、慢查询的关注方式 | 降级实现 + 故障测试 |
| 11-07 | 六 | 作业主体 | 文章 CRUD + 标签多对多 + 分页；把缓存接到列表与详情 | hw05 主体完成 |
| 11-08 | 日 | 收口 | 补 README（key 表、时序图、`EXPLAIN`、降级策略）；自评打分 | **hw05 交付** |

配套讲义：[L08](../lectures/08-database.md)、[L09 Redis 与缓存设计](../lectures/09-redis-and-caching.md)

必须能回答的三个问题（面试常问，也是设计自检）：

1. 缓存和数据库不一致的窗口有多大？兜底是什么？
2. Redis 挂了，接口会怎样？为什么不会 500？
3. 热点文章被高频访问时，回源请求会被放大吗？怎么抑制？

---

## 5. W6：工程化与部署（2026-11-09 ~ 11-15）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 11-09 | 一 | 结构与配置收口 | 统一 `Config` + `Validate()` 启动即校验（fail fast）；环境变量优先级表；手写构造函数注入，去掉包级全局单例 | 可测试的装配入口 |
| 11-10 | 二 | 质量门禁 | `gofmt` / `go vet` / **`golangci-lint`（v2 配置以 `version: "2"` 开头，见官方文档）** / `govulncheck`（`golang.org/x/vuln/cmd/govulncheck`）全部清零 | `.golangci.yml` + 门禁命令 |
| 11-11 | 三 | 集成测试 | `httptest` + 真实 MySQL/Redis；覆盖率 ≥60%；说明未覆盖部分的理由 | 集成测试 + 覆盖率报告 |
| 11-12 | 四 | 容器化 | 多阶段 `Dockerfile`；`CGO_ENABLED=0`、`-trimpath -ldflags "-s -w"`、非 root、`.dockerignore`；记录镜像体积对比 | 可运行的镜像 |
| 11-13 | 五 | 编排与接口文档 | `compose.yaml` 编排 app + MySQL + Redis + 迁移；OpenAPI/Swagger 文档（标注鉴权的接口才写鉴权） | 一键起栈 + 文档 |
| 11-14 | 六 | CI 与作业主体 | GitHub Actions：lint → vet → `test -race` → govulncheck → 构建镜像（`go-version-file: go.mod`）；注意 `GOTOOLCHAIN` 在 CI 中可能尝试联网下载工具链 | CI 工作流 |
| 11-15 | 日 | 收口与阶段门 | 按 G2 清单逐条自评；写阶段复盘 | **hw06 交付 + G2 通过** |

配套讲义：[L13 工程化](../lectures/13-engineering.md)、[L06 测试工程](../lectures/06-testing.md)

---

## 6. 本阶段的版本注意事项

- **Go 1.26 起**：`net/http/httputil.ReverseProxy.Director` 废弃，改用 `Rewrite`；`ServeMux` 尾斜杠重定向由 301 改为 307；`errors.AsType` 可用；`io.ReadAll` 更快更省内存。若在 W6 前后升级到 1.26，可以用重写后的 `go fix`（modernizers）扫一遍代码库。
- **Go 1.27 起**：`go test` 默认执行 `stdversion` vet 检查（报告超出该文件生效 Go 版本的标准库符号）；`encoding/json` 由 v2 实现支撑，默认拒绝非法 UTF-8 与重复键，错误文本可能变化；标准库新增 `uuid` 包（见 <https://pkg.go.dev/uuid>），可以替代第三方 UUID 库。
- **`golangci-lint` v2 与 v1 的配置不兼容**，从官方模板起步而不是照抄旧博客，见 <https://golangci-lint.run>。
- 第三方库版本一律以官方最新稳定版为准，本阶段不要在文档里写死版本号。

---

## 7. 阶段验收清单（G2）

- [ ] `docker compose up -d` 一条命令起 app + MySQL + Redis，迁移自动执行
- [ ] 注册 → 登录 → 访问受保护接口 → 刷新 token → 注销全链路可跑通，且有 `httptest` 覆盖
- [ ] JWT 固定算法，校验 `exp`/`iss`/`aud`，且有「篡改签名」「算法混淆」「过期」三类负向用例
- [ ] 密码为 bcrypt 自适应哈希，数据库中不存在任何明文密码
- [ ] 每个按 id 取资源的接口都有归属校验，且有一个越权访问被拒绝的测试
- [ ] Redis 缓存有 key 设计表与 TTL 策略，三大缓存问题至少处理两个
- [ ] Redis 停机时接口回源而非 500，有测试证明
- [ ] 接口文档与实现一致，公开接口未被错误标注为需要鉴权
- [ ] `golangci-lint run`、`go vet ./...`、`go test -race ./...`、`govulncheck ./...` 全部通过
- [ ] CI 工作流跑通一次
- [ ] hw03–hw06 自评均 ≥70 分

---

## 8. 资源

官方与规范：

- `net/http` 文档：<https://pkg.go.dev/net/http>
- `database/sql` 文档：<https://pkg.go.dev/database/sql>
- RFC 7519（JWT）：<https://www.rfc-editor.org/rfc/rfc7519>
- OWASP API Security Top 10：<https://owasp.org/API-Security/>
- Docker 多阶段构建：<https://docs.docker.com/build/building/multi-stage/>

书籍：

- 《Go 语言实战》——William Kennedy 等。本阶段重点是并发与工程章节。
- 《Go Web 编程》（开源文档，作者 AstaXie）——作为框架无关的 Web 开发参照。
- 《Designing Data-Intensive Applications》——Martin Kleppmann。本阶段只读存储与索引相关章节，为阶段四打底。

---

## 9. 本阶段的典型风险

| 风险 | 表现 | 对策 |
| --- | --- | --- |
| 框架依赖症 | 只会 Gin 的 `c.JSON`，说不清 HTTP 本身 | hw04 允许选框架，但必须能画出「一次请求经过哪些中间件、在何处写响应头」 |
| ORM 依赖症 | 遇到慢查询无法优化 | 本阶段必须手写 `database/sql`（hw03），ORM/sqlc 放在后面作为工具对比 |
| 测试写在最后 | 集成测试变成验收脚本 | 每个接口先写失败用例再实现 |
| 缓存放进去就不管 | 有缓存但没有一致性方案 | hw05 要求 README 必须有时序图与不一致窗口分析 |
| 文档与实现漂移 | Swagger 注释写错鉴权要求 | hw06 要求文档与实现一致，且把「公开接口不要标鉴权」写进验收 |
| 只在本机能跑 | 缺少环境变量与依赖说明 | 一切依赖走 `compose.yaml` 与环境变量表 |
