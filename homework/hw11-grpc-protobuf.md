# 作业 11：gRPC 与 Protobuf 服务

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W11 |
| 发放日期 | 2026-12-14 |
| 交付日期 | 2026-12-20 |
| 预计工时 | 20 小时 |
| 难度 | 挑战 |
| 对应讲义 | L15 |
| 前置作业 | hw03、hw04 |

## 1. 背景与目标

把 hw03 / hw04 定义的用户领域能力暴露为 gRPC 服务，并建立 proto 契约、生成工具链、拦截器体系、错误模型与端到端测试。本作业的核心是「契约先行」：proto 是唯一接口真相，服务端与客户端都由它生成，任何手改生成代码或复用字段编号的行为都视为不合格。

| 目标 | 可观测结果 |
| --- | --- |
| 契约可演进 | `buf lint` 通过，`buf breaking` 对基线无破坏性变更 |
| 生成可复现 | 重新生成后 `git diff` 为空 |
| 形态覆盖 | 一元与服务端流（或双向流）方法各有可跑通的调用 |
| 错误可映射 | gRPC 状态码与业务错误码一一对应，每类有触发用例 |
| 调试可用 | `grpcurl` 能列出服务并调用方法 |

## 2. 需求（必须项）

**M1 proto 契约。** 编写 `user.proto`：`syntax = "proto3"`，显式声明 `package` 与 `go_package`。定义 `UserService`，四个方法覆盖四种调用形态中的至少三种：`GetUser`（一元）、`CreateUser`（一元）、`ListUsers`（服务端流）、`WatchUsers`（服务端流或双向流任选）。每个方法的请求与响应消息必须有独立类型，不复用同一个 message 承担多种语义。

**M2 字段编号规范。** 产出一份「字段编号分配表」，列固定为 `消息 / 编号区间 / 用途 / 状态`，并预留区间（`reserved`）给后续扩展。演示一次兼容演进：新增字段、新增枚举值、用 `reserved` 废弃字段。必须解释为什么废弃字段的编号与名字不能被复用——旧版本客户端仍可能发送这些编号，复用会导致语义错位。

**M3 生成工具链与 CI。** 用 `buf`（模块路径 `github.com/bufbuild/buf`，以官方最新稳定版为准）或 `protoc` 配合 `protoc-gen-go` / `protoc-gen-go-grpc`（以官方最新稳定版为准）生成代码。提交 `buf.yaml` 与 `buf.gen.yaml`（或等价生成脚本）；CI 中必须包含 `buf lint`、`buf breaking` 与「重新生成后无 diff」三项检查。生成代码必须提交进仓库。

**M4 服务端实现。** 用 `google.golang.org/grpc`（以官方最新稳定版为准）实现一元与流式方法。配置 `KeepaliveParams`、最大消息大小、健康检查与 reflection（便于 `grpcurl` 调试；`grpcurl` 用法以官方说明为准）。服务端必须尊重 `stream.Context()`，所有阻塞点都要有取消分支。

**M5 拦截器。** 实现一元与流式两套拦截器（同一职责两种签名都要有），至少覆盖：请求 ID 注入、`log/slog` 结构化日志（含方法名、耗时、状态码）、panic 恢复、鉴权（从 metadata 读 token 并校验）、`context` 超时兜底。必须给出拦截器执行顺序表（`位置 / 拦截器 / 作用 / 顺序理由`），并说明鉴权必须在业务逻辑之前、恢复必须包住其余拦截器。

**M6 错误模型。** 给出 gRPC 状态码到业务错误码的完整映射表，列固定为 `业务错误 / gRPC 状态码 / 触发条件 / 客户端应如何处理`。用 `status.Error` 返回结构化错误。对 `InvalidArgument`、`NotFound`、`PermissionDenied`、`ResourceExhausted`、`Unavailable`、`DeadlineExceeded` 各写一个可触发用例，并断言客户端收到的状态码。

**M7 客户端。** 实现客户端并满足：`context` deadline 传播到服务端；连接状态可观察；元数据注入；对幂等接口配置至少一次重试，并说明为什么非幂等接口不能重试。必须说明 `grpc.WithBlock` 已不推荐的原因（阻塞拨号把连接可用性与启动流程耦合，且不利于故障时快速失败与重连），并给出替代做法。

**M8 与 REST 并存。** 用 gRPC-Gateway 或手写适配层，暴露至少一个 HTTP 接口，且与 gRPC 入口复用同一个 service 层——不得复制业务逻辑。给出「两个入口如何调用同一 service 方法」的调用关系说明。

**M9 端到端测试。** 用 `bufconn` 或真实端口做端到端测试，覆盖成功、超时、鉴权失败、流式中断四类场景；`go test -race ./...` 全绿。

## 3. 需求（加分项）

- **A1 双向流。** 把 `WatchUsers` 实现为双向流，补充客户端按批发送订阅条件、服务端按条件推送的场景，并写出背压处理策略。
- **A2 breaking 基线固化。** 把 `buf breaking` 的基线固化为 git 标签（例如 `proto-baseline`），并在 CI 中作为门禁；说明基线更新流程。
- **A3 冒烟脚本。** 写 `scripts/smoke.sh`，用 `grpcurl` 依次调用四个方法，输出可复现的调试记录并纳入 README。
- **A4 限流与熔断。** 在拦截器链中增加限流或熔断，用持续压测触发 `ResourceExhausted`，给出触发阈值与压测证据。

## 4. 技术约束

| 约束 | 说明 |
| --- | --- |
| 语言基线 | 本仓库基线 Go 1.25；如需使用 1.26 / 1.27 能力必须标注版本并给出 1.25 的替代写法 |
| 第三方库 | 只给模块路径，版本以官方最新稳定版为准；不得编造版本号 |
| 生成代码 | 禁止手改生成文件；生成命令写入 README，CI 校验无 diff |
| 字段编号 | 禁止复用已废弃编号；禁止把 `reserved` 区间重新启用 |
| 分层 | proto 消息不得直接作为数据库层领域模型；必须经 service 层转换 |
| 测试 | 端到端测试必须能在 `-race` 下通过；流式测试必须显式关闭流 |

## 5. 交付物清单

| 编号 | 交付物 | 形式 |
| --- | --- | --- |
| D1 | proto 定义 | `proto/user/v1/user.proto` |
| D2 | 生成配置与生成代码 | `buf.yaml`、`buf.gen.yaml`、`gen/...` |
| D3 | 字段编号分配表 | `docs/hw11-field-numbers.md` |
| D4 | 服务端与客户端 | `internal/grpcserver/...`、`internal/grpcclient/...` |
| D5 | 拦截器 | `internal/grpcserver/interceptors/...`（一元 + 流式各一套） |
| D6 | 状态码映射表 | `docs/hw11-error-mapping.md` |
| D7 | REST 适配层 | `internal/gateway/...` 或 gateway 配置 |
| D8 | 测试 | `internal/grpcserver/*_test.go`、`test/e2e/*_test.go` |
| D9 | README | 含生成命令、`grpcurl` 调试示例、启动步骤 |

## 6. 验收标准（可执行命令）

```bash
buf lint
buf breaking --against '.git#branch=main'
go vet ./...
go test -race ./...
go build ./...
```

| 命令 | 通过条件 |
| --- | --- |
| `buf lint` | 无告警，退出码 0 |
| `buf breaking --against <baseline>` | 相对基线无破坏性变更；本作业内的兼容演进不得触发失败 |
| `go build ./...` | 全部编译通过，生成代码与 proto 一致 |
| `go vet ./...` | 无输出 |
| `go test -race ./...` | 全部通过，无 `DATA RACE` |
| 重新生成检查 | 执行生成命令后 `git status` 无未提交改动 |
| 冒烟调试 | `grpcurl` 能列出 `UserService` 并成功调用 `GetUser` |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| proto 设计与兼容性 | 25 | 消息职责单一；编号分配表完整且预留区间；演进演示正确；`reserved` 用法规范 |
| 生成工具链与 CI | 15 | 生成可复现、无 diff；`buf lint` 与 `buf breaking` 接入 CI；生成代码已提交 |
| 服务端与四种方法 | 20 | 至少三种形态可跑通；`stream.Context()` 被尊重；keepalive / 消息大小 / 健康检查 / reflection 配置到位 |
| 拦截器与错误模型 | 20 | 一元与流式两套齐全；顺序有理由；映射表完整；六类状态码各有触发用例 |
| 测试 | 10 | 覆盖成功 / 超时 / 鉴权失败 / 流式中断；`-race` 通过 |
| 文档 | 10 | README 含生成命令与 `grpcurl` 示例；REST 与 gRPC 复用 service 层说明清晰 |

## 8. 提示与思路

- 先写 proto 再写代码。proto 里出现的每个 message 都应对应一个明确的业务概念；如果一个 message 既用于创建又用于查询响应，通常意味着建模错了。
- 字段编号一旦发布就不可更改。新增字段用新的编号，废弃字段把编号与名字一起写进 `reserved`，并保留注释说明原因。
- 兼容演进的三条底线：不改变已有字段的类型与语义、不复用编号、不改变枚举值对应的含义（枚举可以新增，不能重排）。
- 拦截器顺序建议：恢复（最外层包住一切）→ 请求 ID → 日志 → 鉴权 → 超时兜底 → 业务。理由：恢复要能接住所有后续 panic；鉴权要在业务之前；日志要能记录到鉴权失败。
- 流式方法里最容易漏的是 `Send` 的返回值：客户端提前断开时 `Send` 会返回错误，必须检查并终止循环。同时 `stream.Context()` 要在每次循环迭代里检查。
- 客户端重试只对幂等接口开放。重试策略要带退避与上限，否则会把下游的短暂抖动放大成雪崩。
- `bufconn` 适合单元级端到端测试（不占用端口、启动快）；真实端口适合验证 keepalive 与真实网络行为，两者可以并存。
- proto 消息是传输契约而非领域模型：数据库层用领域结构体，在 service 边界做显式转换，避免协议变更污染数据层。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 手改生成代码 | 下次生成后改动被覆盖，行为不稳定 | 把生成产物当源码维护 | 只改 proto；生成命令与 CI 校验无 diff |
| 复用字段编号 | 旧客户端数据被新字段错误解析 | 认为「字段已废弃所以编号可以回收」 | 废弃字段用 `reserved` 同时锁住编号与名字 |
| 误用 `repeated` / `map` | 大消息体积暴涨、字段顺序语义被误解 | 把 `repeated` 当有序列表依赖，或把 `map` 当有序结构 | `repeated` 顺序不保证业务语义时改为显式排序字段；`map` 不保证遍历顺序 |
| 忘记 `stream.Context()` | 客户端断开后服务端继续推送 | 循环里只判断业务条件 | 每轮迭代检查 `stream.Context()` 的取消状态 |
| 拦截器顺序错误 | 鉴权失败没有日志，panic 未被恢复 | 顺序靠印象拼 | 画出顺序表并写清每条的顺序理由，写测试验证 |
| 服务端流不检查 `Send` 错误 | 断开后疯狂空转刷日志 | 忽略 `Send` 返回值 | 检查 `Send` 错误并立即终止流 |
| proto 当领域模型 | 数据库层被协议细节污染，改 proto 就要改 SQL | 图省事直接透传 | service 边界做显式类型转换 |
| `grpc.WithBlock` 阻塞拨号 | 启动被下游可用性阻塞，故障时不能快速降级 | 沿用旧习惯 | 用非阻塞拨号 + 明确的调用超时与重试策略 |

## 10. 参考实现要点

只给关键设计决策与片段，不给完整成品代码。

```proto
// 决策 1：契约骨架（package 与 go_package 必须显式）
syntax = "proto3";

package user.v1;

option go_package = "example.com/blog/gen/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream ListUsersResponse);
  rpc WatchUsers(stream WatchUsersRequest) returns (stream WatchUsersResponse);
}
```

```proto
// 决策 2：编号分配与预留
message User {
  reserved 4, 5;            // 已废弃字段的编号，永不复用
  reserved "nickname";      // 已废弃字段名，防止复用名字
  int64  id         = 1;
  string name       = 2;
  string email      = 3;
  string avatar_url = 6;    // 新增字段使用新编号
}
```

```go
// 决策 3：一元拦截器与流式拦截器成对实现，职责一一对应
// 一元：func(ctx, req, info, handler) (resp, err)
// 流式：func(srv, ss, info, handler) error
// 顺序：恢复 -> 请求 ID -> 日志 -> 鉴权 -> 超时兜底 -> 业务
```

```go
// 决策 4：结构化错误（状态码由业务错误映射）
return nil, status.Error(codes.InvalidArgument, "name must not be empty")
// codes: InvalidArgument / NotFound / PermissionDenied /
//        ResourceExhausted / Unavailable / DeadlineExceeded
```

```go
// 决策 5：服务端流循环必须同时检查 ctx 与 Send 错误
for {
	select {
	case <-stream.Context().Done():
		return stream.Context().Err()
	default:
	}
	if err := stream.Send(msg); err != nil {
		return err
	}
}
```

- 字段编号分配表建议按消息分块，每块给一段预留区间（例如 `100-199` 留给实验字段），状态列用「使用中 / 已废弃 / 预留」。
- 兼容演进演示可以做成一串连续提交：`feat: add avatar_url` → `feat: add enum value` → `chore: reserve nickname`，配合 `buf breaking` 的每次输出。
- 状态码映射表建议再补一列「客户端重试建议」，明确哪些码可重试、哪些不可。
- 端到端测试建议同时保留 `bufconn` 版本（快，进 `-short` 层）与真实端口版本（慢，进重型层）。
