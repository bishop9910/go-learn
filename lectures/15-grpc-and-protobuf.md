# L15 gRPC 与 Protobuf

> 本篇定位：解决「服务拆分后进程间如何可靠、可演进地互相调用」的问题：契约怎么定义、代码怎么生成、错误与超时怎么表达、连接怎么复用。
> 对应周次：W11（阶段四：微服务与云原生起步）。
> 前置要求：熟悉 `net/http`、`context`、接口与方法集，能读懂 `go.mod` 与构建流程。
> 语言基线：Go 1.25。第三方模块只给模块路径，版本以官方最新稳定版为准；gRPC-Go 的具体函数签名与选项名以
> <https://pkg.go.dev/google.golang.org/grpc> 为准，本篇对不确定的 API 只描述用途并给出链接。

---

## 1. 为什么需要 RPC

### 1.1 进程内调用与跨进程调用的差异

把一次进程内的方法调用改造成跨进程调用，不是「加一层网络」那么简单，语义会系统性地变化：

| 维度 | 进程内调用 | 跨进程调用 |
| --- | --- | --- |
| 失败模型 | 只有返回值与 panic | 请求可能已发出但结果未知（部分失败）；需要超时、重试与幂等设计 |
| 延迟 | 纳秒到微秒 | 毫秒级且抖动大，受网络与对端负载影响 |
| 参数传递 | 共享内存，可传指针与引用 | 必须序列化，是拷贝语义，不能传指针 |
| 版本演进 | 同一二进制一起发布 | 服务端与客户端独立升级，必须同时向前与向后兼容 |
| 观测 | 调用栈即可定位 | 需要 request_id / trace_id 贯穿整条链路 |
| 资源 | 没有连接概念 | 连接池、并发上限、背压、消息大小上限 |

「部分失败」是最容易被忽略的一条：客户端超时并不代表服务端没有执行，这在写操作上会直接产生重复数据。设计接口时要把「哪些操作可安全重试」当成契约的一部分。

### 1.2 REST/JSON 与 gRPC/Protobuf 对比

| 维度 | REST/JSON | gRPC/Protobuf |
| --- | --- | --- |
| 编码 | 文本 JSON，体积大、解析有反射成本 | 二进制 Protobuf，体积小、按字段号解析 |
| 契约 | OpenAPI 等文档描述，属于「约定」 | `.proto` 是强类型契约，代码由它生成 |
| 流式 | 需要 SSE / WebSocket 变通 | 内置四种方法类型，含双向流 |
| 浏览器支持 | 原生支持 | 需要 gRPC-Web 或网关转换 |
| 调试便利性 | `curl` 即可，肉眼可读 | 需要支持反射的工具或预先拿到 proto |
| 生态与治理 | 网关、缓存、CDN、鉴权中间件成熟 | 拦截器、负载均衡、重试策略、健康检查成套 |
| 典型位置 | 对外 API、第三方集成 | 对内服务间调用、高吞吐低延迟链路 |

两者不是替代关系。常见做法是对外保留 REST/JSON，对内用 gRPC；如果需要一套契约同时提供两种入口，用第 11 节的 gRPC-Gateway。

---

## 2. Protobuf

### 2.1 `.proto` 语法

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/svc/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

enum Status {
  STATUS_UNSPECIFIED = 0; // 枚举必须有 0 值，作为默认值
  STATUS_ACTIVE = 1;
  STATUS_DISABLED = 2;
}

message User {
  reserved 7, 9 to 11;         // 冻结已删除字段的编号
  reserved "old_nickname";     // 冻结已删除字段的名字

  string id = 1;
  string name = 2;
  repeated string tags = 3;    // 有序列表
  map<string, string> labels = 4; // 无序键值对
  Status status = 5;
  optional string bio = 6;     // 显式存在性：未设置与空字符串可区分
  google.protobuf.Timestamp created_at = 12;

  oneof contact {              // 同一时刻只有一个字段被设置
    string email = 13;
    string phone = 14;
  }

  message Address {            // 嵌套类型
    string city = 1;
    string country = 2;
  }
  Address address = 15;
}
```

proto 类型到 Go 类型的映射（最终以生成代码为准）：

| proto | Go |
| --- | --- |
| `int32` / `sint32` / `sfixed32` | `int32` |
| `int64` / `sint64` / `sfixed64` | `int64` |
| `uint32` / `fixed32` | `uint32` |
| `uint64` / `fixed64` | `uint64` |
| `float` / `double` | `float32` / `float64` |
| `bool` | `bool` |
| `string` | `string` |
| `bytes` | `[]byte` |
| `message` | 指针类型 `*T` |
| `enum` | 具名 `int32` 派生类型 |
| `repeated T` | `[]T` |
| `map<K,V>` | `map[K]V` |
| proto3 `optional` 标量 | 指针 `*T`，用于区分「未设置」与零值 |

### 2.2 字段编号规则

- 编号取值 1 到 536870911，其中 19000 到 19999 被 Protobuf 保留，不可使用。
- 1 到 15 编码时占一个字节，应留给最高频的字段；16 及以上占两个字节。
- 编号是字段在二进制线格式中的唯一身份。字段名不参与编码（只影响生成的标识符），改名字不影响兼容性，改编号则直接破坏兼容性。
- 删除字段时必须用 `reserved` 同时冻结编号和名字。复用编号的后果是：旧客户端会把新字段按旧类型解析，得到类型正确但内容错误的数据——静默的数据损坏，比报错更难发现。

### 2.3 向后与向前兼容的演进规则

| 可以做 | 不可以做 |
| --- | --- |
| 新增字段（旧客户端忽略未知字段，新客户端看到零值） | 修改已有字段的编号 |
| 新增 `optional` 字段并显式表达存在性 | 修改已有字段的类型（例如 `int32` 改 `string`） |
| 新增枚举值（旧客户端会落到 `UNSPECIFIED`，需要有兜底分支） | 修改已有枚举值的编号 |
| 新增服务或方法 | 修改 `package` 或服务名（改名等于换了一组完整方法名） |
| 用 `reserved` 冻结废弃编号 | 删除字段后把编号让给别的字段 |
| 把 `repeated` 改造成「新增一个 message 字段承载」 | 把单值字段直接改成 `repeated`（线格式不兼容） |

实践经验：接口从第一版起就为将来的扩展留 `optional` 字段，并且所有 `enum` 都要有 `UNSPECIFIED = 0` 加上一层默认分支处理。

### 2.4 `bytes` 与 `string`

- `string` 在 proto3 中语义上要求是合法 UTF-8 文本；放入非法字节序列属于协议违规。
- `bytes` 承载任意字节，适合哈希、加密结果、压缩数据与二进制载荷。
- 不要把二进制数据编码进 `string`，也不要用 `string` 承载可能非 UTF-8 的内容：语言实现与中间件可能对 `string` 做 UTF-8 校验或转码。
- `bytes` 与 `string` 之间的类型改变同样属于不兼容变更（这正是 2.3 节「不换类型」的典型场景）。

---

## 3. 生成工具链

### 3.1 插件与模块路径

| 名称 | 模块路径 | 用途 |
| --- | --- | --- |
| `protoc` | 由 Protocol Buffers 项目发布 | 编译器本体，读取 `.proto` 并调用插件 |
| `protoc-gen-go` | `google.golang.org/protobuf/cmd/protoc-gen-go` | 生成消息类型与序列化代码 |
| `protoc-gen-go-grpc` | `google.golang.org/grpc/cmd/protoc-gen-go-grpc` | 生成服务接口、客户端桩与注册函数 |
| `buf` | `github.com/bufbuild/buf` | 统一的 lint / breaking / 代码生成工具，替代手写 `protoc` 命令行 |

以官方最新稳定版为准。直接使用 `protoc` 时形如：

```bash
protoc --go_out=. --go-grpc_out=. --go_opt=paths=source_relative \
  --go-grpc_opt=paths=source_relative user/v1/user.proto
```

各插件的选项名以插件官方文档为准。无论用哪条路径，**生成代码是派生物，不要手写、也不要手工修改**。

### 3.2 `buf` 配置

`buf.yaml`（模块定义与检查规则）：

```yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
  except:
    - PACKAGE_VERSION_SUFFIX
breaking:
  use:
    - FILE
```

`buf.gen.yaml`（生成配置）：

```yaml
version: v2
managed:
  enabled: true
plugins:
  - local: protoc-gen-go
    out: gen
    opt: paths=source_relative
  - local: protoc-gen-go-grpc
    out: gen
    opt: paths=source_relative
```

`.proto` 中涉及标准错误细节（例如第 6.3 节的 `errdetails`）时，需要在模块依赖里引入对应的 Protobuf 模块，具体命令以 buf 官方文档为准。

### 3.3 `buf lint` 与 `buf breaking`

```bash
buf lint
buf breaking --against '.git#branch=main'
```

- `buf lint` 检查命名、包版本、字段编号风格等约定，属于「提交前必修」。
- `buf breaking` 用当前工作区与指定基线比对，检测不兼容变更：删除字段、改编号、改类型都会在这里被拦住。
- 两者都适合放进 CI 的必过门禁，与第 3.4 节的生成一致性校验一起构成 proto 的守卫。

### 3.4 生成代码纳入版本管理与 CI 校验

生成代码必须提交到版本库。理由：使用者不需要安装 `protoc` 与插件即可编译；CI 与本地看到同一份产物；代码评审能看到契约变化对生成代码的影响。

CI 中校验「生成结果与 proto 一致」的做法：

```bash
buf generate
git diff --exit-code   # 有任何差异说明有人改了 proto 没重新生成，或手工改了生成代码
```

流程固定为四步：本地改 `.proto` → `buf generate` → 提交生成代码 → CI 重跑生成并比对，有差异即失败并提示执行 `buf generate`。

---

## 4. 四种方法类型

```proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);            // 一元
  rpc ListUsers(ListUsersRequest) returns (stream User);            // 服务端流
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse); // 客户端流
  rpc Sync(stream Event) returns (stream Event);                    // 双向流
}
```

生成的服务接口方法名与签名完全由插件决定，不要照抄本篇骨架去手写接口；下面是「数据往哪个方向流动、`Send`/`Recv` 出现在哪里」的示意：

```go
// 一元：直接用返回值和 error 表达结果
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error)

// 服务端流：在循环里 Send，次数由服务端决定
func (s *userServer) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
	for _, u := range users {
		if err := stream.Send(u); err != nil { // 客户端断开或超时都会在这里报错
			return err
		}
	}
	return nil
}

// 客户端流：循环 Recv，读完后一次性返回
func (s *userServer) CreateUsers(stream pb.UserService_CreateUsersServer) error {
	for {
		req, err := stream.Recv()
		if err == io.EOF { // 客户端关闭发送方向，可以汇总返回
			return stream.SendAndClose(&pb.CreateUsersResponse{})
		}
		if err != nil {
			return err
		}
		_ = req
	}
}

// 双向流：两端各自收发。读和写必须在不同 goroutine 里进行，
// 否则一边阻塞会让另一边无法推进；约定通常是一个 goroutine 负责 Recv，
// 另一个负责 Send，并在结束条件上收敛。
```

| 方法类型 | 适用场景 | 注意点 |
| --- | --- | --- |
| 一元 | 绝大多数查询与写入 | 消息大小有上限，大载荷应走流式或对象存储 |
| 服务端流 | 列表分页推送、订阅、日志与事件下发 | 客户端必须能主动取消，服务端要处理 `Send` 返回的错误 |
| 客户端流 | 批量上传、聚合上报、会话式写入 | 服务端要有累积上限，防止客户端只发不收压垮内存 |
| 双向流 | 实时协作、代理转发、需要请求-响应交错的协议 | 并发读写要分离，超时与半关闭语义要事先约定 |

四种方法类型的官方入口：<https://pkg.go.dev/google.golang.org/grpc>

---

## 5. 服务端与客户端基础

### 5.1 服务端

```go
import (
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"

	pb "example.com/svc/gen/user/v1"
)

func serve() error {
	srv := grpc.NewServer(
		grpc.ChainUnaryInterceptor(loggingUnary, authUnary),
		grpc.ChainStreamInterceptor(loggingStream),
		grpc.KeepaliveParams(keepaliveParams),
		grpc.MaxRecvMsgSize(4<<20),
	)

	// 注册函数由 protoc-gen-go-grpc 生成，名字形如 RegisterXxxServiceServer，
	// 签名以生成结果为准，不要手写
	pb.RegisterUserServiceServer(srv, &userServer{})

	// 反射让支持反射的调试工具可以动态发现服务定义
	reflection.Register(srv)

	// 健康检查服务定义在 google.golang.org/grpc/health/grpc_health_v1
	healthpb.RegisterHealthServer(srv, healthServer{})

	lis, err := net.Listen("tcp", ":9000")
	if err != nil {
		return err
	}
	return srv.Serve(lis)
}
```

选项与拦截器类型的具体名称以 <https://pkg.go.dev/google.golang.org/grpc> 为准。

### 5.2 拦截器：一元与流式两套

拦截器是 gRPC 的横切机制，日志、鉴权、链路注入、指标都在这里做。一元和流式是两套独立的函数类型，必须分别注册：

```go
// 一元：拿到请求、调用 handler、拿到响应
func loggingUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (any, error) {
	start := time.Now()
	resp, err := handler(ctx, req)
	slog.Info("grpc",
		"method", info.FullMethod,
		"code", status.Code(err).String(),
		"dur", time.Since(start),
	)
	return resp, err
}

// 流式：没有单一请求与响应，只能包装 ServerStream
func loggingStream(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
	handler grpc.StreamHandler) error {
	return handler(srv, ss)
}
```

顺序规则：`ChainUnaryInterceptor` 里的拦截器按注册顺序进入、按逆序返回，鉴权和日志的相对位置要按这个规则设计（通常日志放最外层，鉴权放内层）。客户端侧的拦截器配置方式以 grpc 包文档为准。

### 5.3 keepalive 与消息大小

| 配置 | 关注点 |
| --- | --- |
| 服务端 keepalive | 探测空闲连接与判断对端存活；间隔设得过激会让客户端被判定为不守规矩 |
| 客户端 keepalive | 穿越 NAT 与负载均衡的空闲超时；间隔过小会触发服务端的最小间隔限制 |
| `MaxRecvMsgSize` | 单条消息上限，防止单个请求耗尽内存；超过上限的请求会以 `ResourceExhausted` 失败 |

keepalive 的参数语义与默认值见 <https://pkg.go.dev/google.golang.org/grpc/keepalive>。调整前先确认问题确实是连接被中间设备断开，而不是服务端本身慢。

### 5.4 反射与健康检查

| 能力 | 包 | 用途 |
| --- | --- | --- |
| 反射 | `google.golang.org/grpc/reflection` | 让调试工具在不持有 proto 的情况下动态发现服务与方法，仅在开发与内网开启 |
| 健康检查 | `google.golang.org/grpc/health` 与 `google.golang.org/grpc/health/grpc_health_v1` | 供负载均衡与编排系统探活；服务定义见包的文档 |

健康检查要接真实依赖判断（数据库、缓存等），而不是永远返回健康，否则编排系统会把故障实例继续放进流量。

### 5.5 客户端

```go
import (
	"context"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"

	pb "example.com/svc/gen/user/v1"
)

func newClient(creds credentials.TransportCredentials) (pb.UserServiceClient, error) {
	// NewClient 是当前的构造函数；grpc.Dial 已不推荐使用
	conn, err := grpc.NewClient("dns:///user-svc:9000",
		grpc.WithTransportCredentials(creds),
		grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(4<<20)),
	)
	if err != nil {
		return nil, err
	}
	return pb.NewUserServiceClient(conn), nil
}

func (c *client) getUser(ctx context.Context, id string) (*pb.User, error) {
	ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()
	return c.conn.GetUser(ctx, &pb.GetUserRequest{Id: id})
}
```

客户端要点：

- **`WithBlock` 已不推荐**：它让 `NewClient` 阻塞到连接就绪，既掩盖了连接失败的原因，也把启动流程与后端可用性强耦合，还会延长启动时间。改为让第一个 RPC 去暴露连接问题，并配好重试与超时。
- 连接是懒建立的：构造函数返回成功只代表参数合法，不代表后端可达。
- 连接对象应当长期复用：它内部维护子连接、解析器与均衡器状态。每个请求新建连接会带来握手开销并耗尽端口。
- 状态与重连由 gRPC 内部处理，应用层不要自己写「检测断开再重连」的循环。
- `WithDefaultCallOptions` 设置默认调用选项（例如最大接收消息大小）；单次调用可以覆盖默认值，具体 API 见 grpc 包文档。
- 每个 RPC 都必须带 `context`，deadline 会沿调用链传播。

---

## 6. 错误模型

### 6.1 状态码与 HTTP 映射

| 状态码 | 语义 | 对应 HTTP | 可否重试 |
| --- | --- | --- | --- |
| `codes.OK` | 成功 | 200 | — |
| `codes.InvalidArgument` | 参数非法 | 400 | 否 |
| `codes.Unauthenticated` | 未提供或凭据无效 | 401 | 否（先刷新凭据） |
| `codes.PermissionDenied` | 无权限 | 403 | 否 |
| `codes.NotFound` | 资源不存在 | 404 | 否 |
| `codes.AlreadyExists` | 已存在 | 409 | 否 |
| `codes.Aborted` | 并发冲突（如事务冲突） | 409 | 视幂等性 |
| `codes.FailedPrecondition` | 前置状态不满足 | 400 | 否 |
| `codes.ResourceExhausted` | 配额或限流耗尽 | 429 | 是（带退避） |
| `codes.Unimplemented` | 方法未实现 | 501 | 否 |
| `codes.Internal` | 服务端内部错误 | 500 | 视幂等性 |
| `codes.Unavailable` | 服务不可用或连接失败 | 503 | 是 |
| `codes.DeadlineExceeded` | 超时 | 504 | 视幂等性 |
| `codes.DataLoss` | 不可恢复的数据损坏 | 500 | 否 |

选码原则：调用方需要据此决定动作的语义才值得区分。把「参数错误」报成 `Internal` 会让客户端重试，把「后端不可达」报成 `InvalidArgument` 会让客户端永不重试。

### 6.2 `status.Error` 与 `status.FromError`

```go
// 服务端：用状态码加消息返回
return nil, status.Error(codes.NotFound, "user not found")

// 客户端：按状态码分支，不要比较错误字符串
if st, ok := status.FromError(err); ok {
	switch st.Code() {
	case codes.NotFound:
		return nil, ErrUserNotFound
	case codes.Unavailable, codes.DeadlineExceeded:
		return nil, ErrUpstreamUnavailable
	}
}
```

`status.FromError` 对非 gRPC 错误返回 `codes.Unknown`。不要用 `err.Error()` 做分支判断：文案会变，包装层会改写它。

### 6.3 `details` 承载结构化错误

状态码只能表达「哪一类错误」，字段级信息用 `details` 传递。标准错误细节定义在 `google.golang.org/genproto/googleapis/rpc/errdetails`：

```go
st := status.New(codes.InvalidArgument, "invalid request")
st, err := st.WithDetails(&errdetails.BadRequest{
	FieldViolations: []*errdetails.BadRequest_FieldViolation{
		{Field: "email", Description: "must be a valid address"},
		{Field: "name", Description: "must not be empty"},
	},
})
if err != nil {
	return nil, status.Error(codes.Internal, "failed to attach details")
}
return nil, st.Err()
```

客户端用 `st.Details()` 取回并做类型断言，逐字段展示校验结果。要点：

- `details` 里的结构必须在客户端与服务端都能解析，因此标准错误细节所在的模块要作为依赖引入到 `buf.yaml` 中。
- 不要把所有调试信息塞进 `details`：它随错误一起回传，可能被日志或监控记录，注意敏感信息。

### 6.4 重试与幂等前提

| 前提 | 说明 |
| --- | --- |
| 操作幂等 | 只有查询、幂等的更新（带版本号或 CAS）才可无条件重试 |
| 有幂等键 | 非幂等操作（创建订单、扣款）要带客户端生成的幂等键，服务端按它去重 |
| 只重试可重试码 | `Unavailable`、部分 `ResourceExhausted` 等；`InvalidArgument`、`NotFound` 重试无意义 |
| 有退避与上限 | 指数退避加抖动，并限制最大尝试次数 |
| 有总预算 | 所有重试共享同一个 deadline，否则重试会成倍放大下游压力 |

gRPC 客户端支持由服务配置下发的重试策略，其格式与限制见官方文档 <https://github.com/grpc/grpc/blob/master/doc/service_config.md>。策略只在服务端返回可重试状态码时生效，且要求调用是幂等的。

---

## 7. 元数据与链路追踪

元数据是随请求传输的 key-value（键为小写），适合放凭据、request_id、trace_id 这类控制信息，不适合放业务数据。

```go
// 客户端：注入出站元数据
md := metadata.Pairs("x-request-id", reqID)
ctx = metadata.NewOutgoingContext(ctx, md)

// 服务端：读取入站元数据
md, ok := metadata.FromIncomingContext(ctx)
if ok {
	ids := md.Get("x-request-id")
	_ = ids
}
```

函数名与用法以 <https://pkg.go.dev/google.golang.org/grpc/metadata> 为准。约定与注意点：

| 事项 | 做法 |
| --- | --- |
| 注入位置 | 客户端一元拦截器统一生成 request_id，避免每个调用点各写一遍 |
| 提取位置 | 服务端一元拦截器提取并放进 `context`，日志从中取值 |
| 传播 | 服务端作为客户端调用下游时，把同一个 request_id 继续注入 |
| 大小限制 | 元数据进入 HTTP/2 header，有大小上限；超限会导致请求失败 |
| 敏感信息 | 认证令牌不要写进日志；元数据在中间代理处可能被记录 |

链路追踪同样以「拦截器注入 + 拦截器提取」为骨架：trace_id 与 span 上下文放在元数据里跨进程传递，具体实现交给 OpenTelemetry 之类的库，不要在业务代码里手工拼 header。

---

## 8. 超时、重试、负载均衡与解析器

- **resolver（解析器）**：把客户端配置的 target 字符串解析成一组后端地址，并持续更新。
- **balancer（均衡器）**：拿到地址列表后决定如何选择子连接。

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| `pick_first` | 连上第一个可用地址后一直用它，其余地址保留作为故障切换 | 单实例、后端前面已有独立负载均衡器 |
| `round_robin` | 在已就绪的子连接之间轮询 | 多实例、需要客户端侧分摊流量 |

长连接场景下的关键事实：TCP 连接一旦建立，客户端不会因为后端扩缩容而重新分配流量。如果 target 是一个 VIP（例如 Kubernetes 的 ClusterIP Service），解析结果只有一个地址，客户端侧均衡无从谈起——所有流量仍由那一跳转发。解决办法有两个方向：

1. 让解析器能拿到全部实例地址（例如 headless service 的 DNS 多 A 记录，或接入服务注册中心），并配置 `round_robin` 之类的均衡策略。
2. 保留 VIP，把均衡交给外部组件（Service Mesh 边车或云负载均衡）。

第 1 条需要服务发现的支持，见 [L16 微服务与可观测性](16-microservices-and-observability.md)。当前阶段先记住结论：**gRPC 的长连接特性决定了「客户端侧负载均衡 + 服务发现」是微服务的必需品，而不是可选项**。

超时也要分层设置：客户端 deadline 必须大于下游各跳之和，同时给每一跳留下递减的余量；只在最外层设一个很大的超时，等于没有超时。

---

## 9. 安全

| 项 | 要点 |
| --- | --- |
| TLS | 生产环境必须启用；凭据类型在 `google.golang.org/grpc/credentials` 包（见 <https://pkg.go.dev/google.golang.org/grpc/credentials>） |
| Server 名称 | 客户端必须校验服务端证书里的名称与它实际连接的目标一致，否则中间人可以用任意合法证书 |
| mTLS | 服务端要求并校验客户端证书，客户端同时加载自己的证书链与 CA；适合零信任内网 |
| 凭据挂载 | 证书与私钥从文件或密钥管理服务读取，不要写进镜像或代码 |
| 鉴权 | 放在拦截器：从元数据取令牌并校验，失败返回 `codes.Unauthenticated` |
| 授权 | TLS 只证明「谁在说话」，不代表「能做什么」；权限判断仍需业务层或策略层 |
| 明文凭据 | 只允许在本地开发使用，并且要在代码和配置里显式标注，避免被复制到生产 |

把鉴权写在每个 handler 里是常见的越权来源：新增一个方法时忘记加校验即可绕过。统一放拦截器，并对「不需要鉴权的白名单方法」做显式列举。

---

## 10. 完整示例：user 服务

### 10.1 `user.proto`

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/svc/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  google.protobuf.Timestamp created_at = 4;
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}
```

### 10.2 目录布局与生成

```text
svc/
  proto/user/v1/user.proto     # 唯一契约来源
  buf.yaml
  buf.gen.yaml
  gen/user/v1/                 # 生成代码，纳入版本管理，禁止手改
  internal/server/             # 服务端实现
  internal/client/             # 客户端封装
  cmd/server/                  # 可执行入口
```

### 10.3 服务端实现

```go
type userServer struct {
	pb.UnimplementedUserServiceServer // 生成的前向兼容占位，未实现的方法返回 Unimplemented
	repo                              // 数据访问，省略定义
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
	if req.GetId() == "" {
		return nil, status.Error(codes.InvalidArgument, "id is required")
	}
	u, err := s.repo.Find(ctx, req.GetId())
	if errors.Is(err, ErrNotFound) {
		return nil, status.Error(codes.NotFound, "user not found")
	}
	if err != nil {
		return nil, status.Error(codes.Internal, "query failed")
	}
	return &pb.GetUserResponse{User: toProto(u)}, nil
}

func (s *userServer) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
	if req.GetPageSize() <= 0 || req.GetPageSize() > 1000 {
		return status.Error(codes.InvalidArgument, "page_size must be in (0,1000]")
	}
	it, err := s.repo.Iterate(stream.Context(), req.GetPageToken(), int(req.GetPageSize()))
	if err != nil {
		return status.Error(codes.Internal, "iterate failed")
	}
	for it.Next() {
		if err := stream.Send(toProto(it.Value())); err != nil {
			return err // 客户端取消或断开
		}
	}
	return it.Err()
}

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
	if req.GetEmail() == "" {
		st := status.New(codes.InvalidArgument, "invalid request")
		st, _ = st.WithDetails(&errdetails.BadRequest{
			FieldViolations: []*errdetails.BadRequest_FieldViolation{
				{Field: "email", Description: "must not be empty"},
			},
		})
		return nil, st.Err()
	}
	u, err := s.repo.Create(ctx, req.GetName(), req.GetEmail())
	if err != nil {
		return nil, status.Error(codes.Internal, "create failed")
	}
	return &pb.CreateUserResponse{User: toProto(u)}, nil
}
```

服务端实现必须内嵌生成的前向兼容类型，否则将来新增方法会让旧实现编译失败。`UnimplementedUserServiceServer` 的具体名字以生成结果为准。

### 10.4 拦截器：日志与鉴权

```go
func authUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (any, error) {
	if isPublic(info.FullMethod) {
		return handler(ctx, req)
	}
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "missing metadata")
	}
	tokens := md.Get("authorization")
	if len(tokens) == 0 || !verifyToken(tokens[0]) {
		return nil, status.Error(codes.Unauthenticated, "invalid token")
	}
	return handler(ctx, req)
}

var publicMethods = map[string]bool{
	"/user.v1.UserService/GetUser": true, // 白名单显式列举
}

func isPublic(fullMethod string) bool { return publicMethods[fullMethod] }
```

### 10.5 客户端调用

```go
client := pb.NewUserServiceClient(conn)

// 一元
ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
defer cancel()
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: id})
if st, ok := status.FromError(err); ok && st.Code() == codes.NotFound {
	// 业务侧处理
}

// 服务端流：循环 Recv 直到 io.EOF
stream, err := client.ListUsers(ctx, &pb.ListUsersRequest{PageSize: 100})
if err != nil {
	return err
}
for {
	u, err := stream.Recv()
	if err == io.EOF {
		break
	}
	if err != nil {
		return err
	}
	_ = u
}
```

### 10.6 错误映射表

| 业务情形 | 状态码 | 附加信息 | 客户端动作 |
| --- | --- | --- | --- |
| id 为空 | `InvalidArgument` | `details` 列出字段 | 提示用户，不重试 |
| 邮箱格式错误 | `InvalidArgument` | `errdetails.BadRequest` | 提示用户，不重试 |
| 用户不存在 | `NotFound` | 无 | 展示「不存在」 |
| 令牌缺失或失效 | `Unauthenticated` | 无 | 刷新凭据后重试一次 |
| 无权限访问他人数据 | `PermissionDenied` | 无 | 提示无权限 |
| 下游限流 | `ResourceExhausted` | 可含重试间隔 | 退避后重试 |
| 后端实例不可达 | `Unavailable` | 无 | 退避后重试（幂等前提下） |
| 请求超时 | `DeadlineExceeded` | 无 | 幂等则重试，否则报错并给出 request_id |
| 依赖内部异常 | `Internal` | 仅记录日志，不回传细节 | 报错并提示联系支持（带 request_id） |

---

## 11. 从 REST 迁移到 gRPC 的渐进路径

不要一次性重写。可行路径：

1. 选定一个内部调用链（例如网关到用户服务），保留现有 REST 对外接口不动。
2. 为这条链编写 `.proto`，用 `buf lint` 与 `buf breaking` 纳入 CI。
3. 服务端先实现 gRPC 接口，与现有 REST handler 共用同一层业务逻辑，避免两套实现。
4. 客户端（调用方服务）切到 gRPC，保留 REST 入口作为回退开关，用配置控制。
5. 稳定后再考虑用 gRPC-Gateway 从同一份 proto 生成 HTTP/JSON 网关，把对外接口也统一到契约上。
6. 迁移过程中所有新增字段都按第 2.3 节的兼容规则演进，保证新旧调用方可以并存。

gRPC-Gateway（模块路径 `github.com/grpc-ecosystem/grpc-gateway`，以官方最新稳定版为准）的用途是：根据 proto 中的 HTTP 注解生成一个反向代理，把 REST/JSON 请求翻译成 gRPC 调用，从而用一份契约同时服务浏览器与内部服务。局限在于流式方法的 HTTP 映射能力有限，需要按官方文档确认支持范围。

---

## 12. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 复用已删除字段的编号 | 旧客户端读到类型正确但内容错误的数据 | 编号是线格式契约 | 删除字段后用 `reserved` 冻结编号与名字 |
| 修改字段类型（如 `int32` 改 `string`） | 解析失败或数据错乱 | 线格式不兼容 | 新增字段替代，旧字段保留并废弃 |
| 把 proto3 的 `optional` 当必填用 | 服务端拿到零值，无法区分「未设置」 | proto3 默认值语义 | 用 `optional` 并显式校验存在性 |
| 手写或手工修改生成代码 | 下次生成被覆盖，行为不一致 | 生成物是派生物 | 只改 `.proto`，生成代码进版本管理并在 CI 校验一致 |
| 用错误字符串判断错误类型 | 文案一变逻辑就坏 | 没有使用状态码 | `status.FromError` + `codes` |
| 非幂等的 Create 无条件重试 | 重复创建数据 | 网络不确定性叠加自动重试 | 幂等键 + 只在可重试码上重试 |
| 不设 deadline | 上游变慢时连接与 goroutine 被占满 | 缺少超时传播 | 每个 RPC 都带 `context` deadline，逐跳递减 |
| 每次请求新建连接 | 握手开销大、端口耗尽、负载均衡失效 | 把连接当成了请求对象 | 进程内长期复用连接 |
| 用大消息传输文件 | 内存峰值高，失败重传代价大 | 消息大小上限与全量反序列化 | 用流式传输，或对象存储加引用传递 |
| 用 `pick_first` 却期望多实例均衡 | 只有一个后端承担流量 | 默认均衡策略是「先连上就用」 | 用 `round_robin` 并让 resolver 返回多地址，或使用 headless service |
| 把业务数据放进元数据 | header 超限，或被中间件丢弃 | 元数据不是传输体 | 业务数据放消息体 |
| 生产环境仍用明文凭据 | 全部流量可被嗅探与篡改 | 开发配置未替换 | TLS/mTLS，并在拦截器里做鉴权 |
| 在每个 handler 里手写鉴权 | 漏改一处即越权 | 横切逻辑散落各处 | 统一放拦截器，白名单显式列举 |

---

## 13. 动手练习

1. 写一个 `user.proto`，包含一元、服务端流两种方法，用 `buf lint` 与 `buf generate` 生成代码，并把生成物提交到仓库。
2. 在 CI 中加入 `buf generate && git diff --exit-code`，故意手工改一行生成代码，确认 CI 能拦住。
3. 实现服务端，为一元方法加参数校验并返回 `InvalidArgument` 与 `errdetails.BadRequest`，客户端解析 `details` 并逐字段打印。
4. 实现一元服务端拦截器，注入 `x-request-id` 并在日志中输出；再实现客户端拦截器，向下游传播同一个 id。
5. 为服务注册健康检查并接真实依赖状态，用一个脚本模拟依赖故障，确认探活结果变化。
6. 写一个 `round_robin` 的客户端并用两个后端实例验证流量分摊，再换成 `pick_first` 观察差异。
7. 给一元调用加 200 ms deadline，在服务端注入 500 ms 延迟，确认客户端拿到 `DeadlineExceeded` 且服务端能感知上下文取消。
8. 用 `buf breaking --against` 对比一次「修改字段编号」的改动，确认能被检出并说明原因。

---

## 14. 自检清单

- [ ] `.proto` 里每个字段都有明确编号，删除过的编号与名字都已 `reserved`。
- [ ] 所有 `enum` 都有 `UNSPECIFIED = 0`，客户端对未知枚举值有兜底分支。
- [ ] 生成代码已提交，且 CI 会校验「生成结果与 proto 一致」。
- [ ] `buf lint` 与 `buf breaking` 都在 CI 门禁中。
- [ ] 服务端内嵌了生成的前向兼容类型，未实现的方法返回 `Unimplemented`。
- [ ] 一元与流式拦截器都已注册，顺序符合「日志在外、鉴权在内」的预期。
- [ ] 每个 RPC 都设置了 deadline，重试有退避、上限、幂等前提与总预算。
- [ ] 错误返回使用状态码，字段级信息用 `details`，没有用错误字符串做分支。
- [ ] 客户端复用连接，没有使用 `WithBlock`，也没有每个请求新建连接。
- [ ] 多实例场景下确认过 target 能被解析成多个地址，且均衡策略符合预期。
- [ ] 生产环境启用 TLS（必要时 mTLS），凭据不写在代码或镜像里。
- [ ] 鉴权在拦截器统一实现，白名单方法显式列举。
- [ ] 反射只在开发或受控内网开启；健康检查反映真实依赖状态。

---

## 15. 延伸阅读

- gRPC-Go 主包（服务端、客户端、拦截器、选项）：<https://pkg.go.dev/google.golang.org/grpc>
- gRPC 状态码与 `status`：<https://pkg.go.dev/google.golang.org/grpc/status>
- gRPC 状态码常量（`codes`）：<https://pkg.go.dev/google.golang.org/grpc/codes>
- 元数据（`metadata`）：<https://pkg.go.dev/google.golang.org/grpc/metadata>
- 凭据（`credentials`）：<https://pkg.go.dev/google.golang.org/grpc/credentials>
- keepalive：<https://pkg.go.dev/google.golang.org/grpc/keepalive>
- 反射（`reflection`）：<https://pkg.go.dev/google.golang.org/grpc/reflection>
- 健康检查（`health` / `grpc_health_v1`）：<https://pkg.go.dev/google.golang.org/grpc/health>
- Protobuf Go 运行时与 `protoc-gen-go`：<https://pkg.go.dev/google.golang.org/protobuf>
- `protoc-gen-go-grpc`：<https://pkg.go.dev/google.golang.org/grpc/cmd/protoc-gen-go-grpc>
- `buf`（模块路径 `github.com/bufbuild/buf`，以官方最新稳定版为准）：<https://pkg.go.dev/github.com/bufbuild/buf>
- gRPC-Gateway（模块路径 `github.com/grpc-ecosystem/grpc-gateway`，以官方最新稳定版为准）：<https://pkg.go.dev/github.com/grpc-ecosystem/grpc-gateway>
- gRPC 官方文档：<https://grpc.io/docs/>
- Protocol Buffers 语言指南（proto3）：<https://protobuf.dev/programming-guides/proto3/>
- 书籍：《Go 语言高级编程》，柴树杉、曹春晖，人民邮电出版社
- 书籍：《Cloud Native Go》，Matthew A. Titmus，O'Reilly Media
