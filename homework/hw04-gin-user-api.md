# 作业 04：用户系统 API——注册、登录、JWT 与中间件

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W3–W4 |
| 发放日期 | 2026-10-19 |
| 交付日期 | 2026-11-01 |
| 预计工时 | 20 小时 |
| 难度 | 进阶 |
| 对应讲义 | L07、L10 |
| 前置作业 | hw03 |

## 1. 背景与目标

阶段二的目标是交付「能被别人调用的服务」。本作业实现一个最小但完整的用户系统 API：注册、登录、签发与刷新令牌、注销、受保护资源，以及支撑它们的中间件与统一错误模型。完成本作业后你应能：

- 独立给出路由表、请求/响应 schema 与错误码表，并保证文档与实现一致。
- 严格区分认证（你是谁 → 401）与授权（你能做什么 → 403）。
- 说清 JWT 的可校验面（算法、签发者、受众、有效期）与 access/refresh 双令牌轮换的取舍。
- 用中间件把请求 ID、日志、recover、限流、认证等横切关注点从业务代码中剥离。

交付形态：可 `go build` 的 HTTP 服务 + README + 自动化测试 + 一份 `curl` 端到端脚本。

## 2. 需求（必须项）

**路线选择**

- **M1** 从下表三条路线中选择一条，并在 README 说明理由（依赖体积、可控性、团队熟悉度、后续作业复用成本）。第三方模块版本一律以官方最新稳定版为准。

| 路线 | 模块 / 包 | 优点 | 代价 |
| --- | --- | --- | --- |
| A | `github.com/gin-gonic/gin` | 生态大、中间件多、参数绑定方便 | 依赖较重，隐式行为多 |
| B | `github.com/labstack/echo` | API 直观、中间件模型清晰 | 生态小于路线 A |
| C | 标准库 `net/http` + Go 1.22 起的增强 `ServeMux` 路由（方法与通配符） | 零第三方依赖、行为完全可控 | 分组、绑定、校验需自己写 |

路线 C 依赖 Go 1.22 起的增强路由，见 <https://pkg.go.dev/net/http>。

**接口清单**

- **M2** 必须实现下列端点，路径、方法与认证语义不得改动：

| 方法 | 路径 | 认证 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/v1/users` | 否 | 注册，返回用户公开信息 |
| POST | `/api/v1/sessions` | 否 | 登录，返回 access + refresh |
| GET | `/api/v1/users/me` | 是 | 返回当前身份对应的用户 |
| POST | `/api/v1/sessions/refresh` | 否（凭 refresh） | 轮换并返回新 access + 新 refresh |
| DELETE | `/api/v1/sessions/current` | 是 | 注销当前会话 |
| GET | `/healthz` | 否 | 存活探测，不查业务依赖 |

- **M3** README 必须给出每个端点的请求与响应示例（`json` 代码块），字段名与类型和实现一致；错误响应统一为 `{code, message, request_id}` 三段结构。

**密码与令牌**

- **M4** 密码使用自适应哈希：`golang.org/x/crypto/bcrypt`（以官方最新稳定版为准）。禁止明文存储、禁止可逆加密、禁止自研哈希。
- **M5** 登录失败时，状态码、响应体与响应耗时的量级都不得暴露「用户不存在」与「密码错误」的区别，统一返回同一错误码。
- **M6** JWT 使用 `github.com/golang-jwt/jwt/v5`（以官方最新稳定版为准）。解析入口、选项名与 claims 结构以 <https://pkg.go.dev/github.com/golang-jwt/jwt/v5> 为准，本作业不锁定具体函数名。
- **M7** 校验必须显式限定允许的签名算法白名单（拒绝 `none`，不允许 HMAC 与 RSA 算法混用），并校验 `exp`、`iat`、`iss`、`aud` 四项。
- **M8** access 短有效期（分钟级）、refresh 长有效期（天级）；每次刷新都必须轮换 refresh，旧 refresh 立即失效。
- **M9** 注销必须真正生效：黑名单（带剩余 TTL）或用户令牌版本号二选一，README 写明方案与失效延迟。

**中间件**

- **M10** 必须实现并装配：请求 ID、结构化日志（`log/slog`）、recover 兜底（panic 转 500 并记录堆栈）、认证、限流。限流可使用 `golang.org/x/time/rate`（以官方最新稳定版为准）。
- **M11** 认证中间件把身份写入 `context.Context`，key 必须是自定义的非导出类型，不得使用字符串字面量作 key。
- **M12** README 必须画出中间件顺序（文本或 mermaid）并解释原因，至少说明：recover 在最外层、日志要能看到最终状态码、限流排在认证之前以免为攻击流量做哈希校验。

**错误模型与安全**

- **M13** 内部错误（数据库、哈希、网络）不得出现在响应体；响应只给稳定错误码与人类可读 `message`。错误码表至少覆盖：参数校验失败、凭证缺失、凭证过期、签名无效、无权访问、资源不存在、冲突、限流、内部错误。
- **M14** 401 与 403 严格区分：无凭证 / 过期 / 签名无效 / 算法不符 → 401；已认证但无权访问 → 403。
- **M15** 生产路径必须走 HTTPS；若只在本地开发，README 必须显式声明「仅本地开发，禁止明文传输密码」，并说明正式环境如何强制 TLS。
- **M16** 至少在**一个路由组**上启用 Go 1.25 新增的 `net/http.CrossOriginProtection`（基于 Fetch metadata 的 CSRF 防护，无需 token），README 说明其作用与不适用场景。该 API 自 Go 1.25 提供。
- **M17** CORS 使用来源白名单；禁止 `Access-Control-Allow-Origin: *`，禁止反射任意 `Origin`。
- **M18** 用 `net/http/httptest`（见 <https://pkg.go.dev/net/http/httptest>）覆盖完整链路：注册 → 登录 → 访问 `/api/v1/users/me` → 刷新 → 注销 → 再次访问被拒。
- **M19** 必须包含三类负向用例：过期令牌、篡改签名、算法混淆（改算法头部或伪造 `none`）。
- **M20** 测试必须不依赖外部数据库即可运行（内存实现或测试替身）。

## 3. 需求（加分项）

- **B1** 用 `testing/synctest`（Go 1.25 转正，见 <https://pkg.go.dev/testing/synctest>）测试令牌过期与刷新窗口，避免真实等待。
- **B2** 注册与登录增加「按 IP + 按账号」双维度限流，并在被限流时返回 `Retry-After`。
- **B3** refresh 重放检测：同一 refresh 被第二次使用时，使该用户全部 refresh 失效。
- **B4** 用 `runtime/metrics` 的 `/sched/goroutines` 系列指标（Go 1.26+，见 <https://pkg.go.dev/runtime/metrics>）暴露一个只读诊断端点，并说明其访问控制。

## 4. 技术约束

| 编号 | 约束 |
| --- | --- |
| C1 | Go 1.25 工具链；`go.mod` 的 `go` 行与实现所用的最低版本一致 |
| C2 | 不得把密码、令牌、`Authorization` 头原文写入任何日志 |
| C3 | 不得使用包级可变全局变量保存用户、会话或配置 |
| C4 | 分层清晰：handler / service / repository，handler 不做 SQL、repository 不写 HTTP |
| C5 | 所有外部输入（JSON、查询参数、头）必须显式校验后才进入业务逻辑 |
| C6 | 超时与取消贯穿：所有请求处理链路接受 `context.Context`，数据库与下游调用带超时 |
| C7 | 文档与实现一致：README 中出现的错误码必须在代码中存在且可达 |

## 5. 交付物清单

| 交付物 | 要求 |
| --- | --- |
| 源代码 | 可 `go build ./...` 的完整模块，含分层目录 |
| README | 路线选择理由、路由表、中间件顺序图、错误码表、请求/响应示例、环境变量表 |
| 测试 | `*_test.go`，覆盖 M18 链路与 M19 三类负向用例 |
| 端到端脚本 | `scripts/e2e.sh`（或 `.ps1`）：`curl` 走通注册 → 登录 → me → 刷新 → 注销 |

## 6. 验收标准（可执行命令）

| 命令 | 通过标准 |
| --- | --- |
| `go build ./...` | 无错误 |
| `go vet ./...` | 无输出（Go 1.25 起 `go vet` 新增 `waitgroup` 分析器，见 <https://go.dev/doc/go1.25>） |
| `go test ./...` | 全部通过，包含 M18 与 M19 用例 |
| `go test -race ./...` | 全部通过，无数据竞争报告 |
| `gofmt -l .` | 输出为空 |
| `bash scripts/e2e.sh` | 链路全部返回预期状态码；注销后 `GET /api/v1/users/me` 必须为 401 |

| 手工：过期 access 访问受保护接口 | 401，响应体含 `code` 与 `request_id` |
| 手工：算法改 `none` 或改一字签名 | 401，且日志中不出现令牌原文 |
| 手工：同一 refresh 连用两次 | 第二次 401 |
| 手工：连续触发限流 | 429（若实现 B2） |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 功能 | 30 | 六个端点语义正确；注册/登录/刷新/注销链路完整；错误码表覆盖且可达 |
| 密码与 JWT 安全 | 25 | bcrypt 使用正确；算法白名单与四项 claims 校验；双令牌轮换与注销失效；无明文/可逆存储 |
| 中间件与错误模型 | 20 | 五类中间件齐备且顺序有据；context 传身份用自定义 key；统一错误结构；401/403 不混用 |
| 测试覆盖 | 15 | 正向链路 + 三类负向用例；无外部依赖；`-race` 干净 |
| 文档 | 10 | 路由表、中间件顺序图、错误码表、curl 脚本与实现一致 |

## 8. 提示与思路

- **先定契约再写代码**：把路由表、错误码表、请求/响应示例写进 README，再逐条实现，最后用 e2e 脚本回验。
- **认证与授权分两个中间件**：认证中间件只解析身份并写入 context，授权判断放在 handler 或服务层，便于 `/api/v1/users/me` 与后续作业的资源级授权复用。
- **令牌状态集中管理**：把「签发 / 校验 / 吊销」收敛到一个接口，让测试用内存实现替换，避免测试依赖真实时间与外部存储。
- **负向用例优先写**：算法混淆与篡改签名是最容易被漏掉、也最容易被追问的两类。
- **观测性用在排障上**：让每个响应都带 `request_id`，把它写进 `log/slog` 的每条记录，出错时按 ID 串起整条链路。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| JWT 未固定算法 | 攻击者用其它算法伪造令牌通过校验 | 只解析不限定算法白名单 | 解析时显式限定允许的算法集合，拒绝 `none` |
| `exp` 未校验 | 过期令牌长期可用 | 依赖库默认或忘记开启校验 | 显式校验 `exp`/`iat`/`iss`/`aud` 并加过期负向用例 |
| 把 token 写进日志 | 日志泄露可直接复用凭证 | 打印整个请求头或 claims | 日志只留 `request_id`、用户 ID、路径、状态码 |
| context key 用字符串 | 键冲突、`go vet` 告警 | 用 `"user"` 之类字面量 | 定义非导出 struct 类型的 key |
| 全局变量存用户 | 并发请求互相覆盖，测试串味 | 用包级 map/变量当会话存储 | 通过依赖注入持有存储，请求态只走 context |
| 并发写 map | `-race` 报错，进程可能崩溃 | 多 goroutine 无锁写同一 map | 加锁、改用同步原语或换成存储层 |
| bcrypt cost 过低 | 离线爆破成本低 | 为了测试速度把 cost 调到最低 | 生产用合理 cost，测试单独降低并在文档说明 |
| 把密码字段返回给客户端 | 响应体里出现哈希或明文 | 直接序列化数据模型结构体 | 定义独立的响应 DTO，只暴露公开字段 |

## 10. 参考实现要点

- **目录结构**（分层不变，命名可调）：`cmd/server`（组装依赖、启动、优雅关闭）、`internal/config`（配置读取与启动校验）、`internal/handler`（绑定、校验、错误映射）、`internal/service`（注册、登录、刷新、注销等业务规则）、`internal/repository`（存储接口 + 内存与数据库实现）、`internal/middleware`（请求 ID、日志、recover、认证、限流）、`internal/auth`（令牌签发与校验）。
- **统一错误映射**：定义内部错误类型 + 错误码常量，handler 层用一处映射函数把内部错误转成 `{code, message, request_id}`；请求 ID 优先取客户端传入的 `X-Request-ID`（校验格式），否则本地生成，并写回响应头与日志上下文。
- **关键片段（认证中间件骨架，示意用）**：

```go
type ctxKey struct{ name string }

var userIDKey = ctxKey{name: "user_id"}

func Auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims, err := verifier.Verify(r.Context(), bearerToken(r)) // 取 Authorization 头
		if err != nil {
			writeErr(w, r, ErrCodeUnauthorized) // 对外不区分过期与签名错误
			return
		}
		next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), userIDKey, claims.Subject)))
	})
}
```

- **令牌校验清单**：算法白名单 → 签名 → `exp` → `iat` → `iss` → `aud` → 吊销状态（黑名单或版本号），逐项写成表驱动测试；README 的中间件顺序按此基准并解释理由：请求 ID → 日志 → recover → 限流 → 认证 → 业务 handler。
- **自检**：错误码表里的每个码都能被至少一个测试触发；401 与 403 各至少有一个用例；日志中 grep 不到任何 token 或密码字样。
