# L05 序列化：encoding/json 与 json/v2

本篇定位：讲清 Go 标准库 JSON 序列化的实现换代（Go 1.25 的实验包到 Go 1.27 转正），以及 v1 公开 API 的稳定用法与全部高频坑。对应 W2（标准库补全）与 W5（接口与协议设计）。前置要求：能熟练读写 struct、指针、接口与 `io.Reader`/`io.Writer`，并已掌握[L03 错误处理与结构化日志](03-errors-and-logging.md)中的错误包装用法。

## 1. 版本分水岭

### 1.1 三个时间点

| 版本 | 状态 | 相关包 | 启用方式 |
| --- | --- | --- | --- |
| Go 1.25 | 实验 | `encoding/json/v2`、`encoding/json/jsontext` | `GOEXPERIMENT=jsonv2` |
| Go 1.26 | 实验 | 同上 | 同上 |
| Go 1.27 | 转正 | 同上 | 默认可用；`GOEXPERIMENT=nojsonv2` 退回旧实现 |

两条必须记住的官方事实：

| 事实 | 版本 |
| --- | --- |
| 新增实验包 `encoding/json/v2` 与 `encoding/json/jsontext`，用 `GOEXPERIMENT=jsonv2` 启用；启用后 `encoding/json` 由新实现支撑，并新增若干配置项 | Go 1.25 |
| `encoding/json/v2` 与 `encoding/json/jsontext` 转正；`encoding/json` 由 v2 实现支撑，行为保持，错误文本可能变化；`GOEXPERIMENT=nojsonv2` 可退回旧实现 | Go 1.27 |

「行为保持」的边界要说清楚：v1 公开 API 的语义不变，但**错误文本属于实现细节**。1.27 之后同一段坏 JSON 报出的字符串可能与 1.25 之前不同，因此 `err.Error() == "..."` 或对错误字符串做 `strings.Contains` 的断言都必须删掉。

### 1.2 GOEXPERIMENT 的控制层次

GOEXPERIMENT 的取值只有两个来源：构建时的环境变量，以及由它派生出的构建约束。`go.mod` 里没有 GOEXPERIMENT 专用指令。

| 层次 | 写法 | 作用范围 | 说明 |
| --- | --- | --- | --- |
| 环境变量（临时） | `$env:GOEXPERIMENT="jsonv2"; go build ./...` | 当前 shell | PowerShell 写法 |
| 环境变量（临时） | `GOEXPERIMENT=jsonv2 go build ./...` | 单条命令 | bash 写法 |
| 环境变量（持久） | `go env -w GOEXPERIMENT=jsonv2` | 该用户所有构建 | 排障时先查这里，它会让行为与项目解耦 |
| 构建约束 | `//go:build goexperiment.jsonv2` | 单个文件 | 开关打开时该文件才参与编译，反向用 `!goexperiment.jsonv2` |
| `go.mod` 的 `go` 行 | `go 1.25` | 整个模块 | 决定语言版本与「本文件生效的标准库符号」判定基准 |
| `go.mod` 的 `toolchain` 行 | `toolchain go1.27.0` | 整个模块 | 决定用哪个工具链；不同工具链的默认实验集合不同 |

`go.mod` 层面能做的只有两件事：用 `go` 行声明最低语言版本（写 `go 1.27` 会让 1.25 工具链直接拒绝构建），或用 `toolchain` 行固定工具链（Go 1.25 起，更新 `go` 行时不再自动写 `toolchain` 行）。Go 1.27 起 `go test` 默认执行 `stdversion` 检查，误用超出该文件生效版本的标准库符号会在测试阶段被报出来。

### 1.3 写「1.25 与 1.27 都能编译」的可移植代码

1. 只用 v1 `encoding/json` 的公开 API：1.25 由旧实现支撑，1.27 由 v2 实现支撑，语义一致。
2. 不依赖错误文本、`map` 序列化顺序与浮点格式化细节。
3. 需要 v2 独有行为时，把差异隔离进构建约束分文件，业务代码面向函数而非包。
4. 用两种设置各构建一次：`GOEXPERIMENT=jsonv2 go build ./...` 与 `GOEXPERIMENT=nojsonv2 go build ./...` 都要通过。

```go
// codec_v2.go —— 实验开关打开或 1.27 默认时参与编译
//go:build goexperiment.jsonv2

package codec

// v2 的函数、选项常量与配置项一律以官方文档为准：https://pkg.go.dev/encoding/json/v2
func RejectDuplicateKeys() bool { return true }
```

```go
// codec_legacy.go —— 旧实现下参与编译
//go:build !goexperiment.jsonv2

package codec

func RejectDuplicateKeys() bool { return false }
```

业务代码只调用 `codec.RejectDuplicateKeys()`。这样 `go 1.25` 的模块在 1.27 工具链下能编译，`GOEXPERIMENT=nojsonv2` 回退时也不炸。

## 2. v1 核心 API

### 2.1 Marshal / Unmarshal / MarshalIndent

| API | 输入 | 输出 | 典型场景 |
| --- | --- | --- | --- |
| `json.Marshal` | 任意值 | `[]byte` | 需要先拿到完整字节再决定状态码 |
| `json.Unmarshal` | `[]byte` + 目标指针 | `error` | 已知完整请求体、配置文件 |
| `json.MarshalIndent` | 值 + 行前缀 + 缩进 | `[]byte` | 给人看的文件、调试输出 |

```go
b, err := json.MarshalIndent(User{Name: "alice"}, "", "  ")
if err != nil {
	return err
}
fmt.Println(string(b))
```

### 2.2 Encoder / Decoder：生产路径

| API | 作用 | 注意点 |
| --- | --- | --- |
| `json.NewEncoder(w).Encode(v)` | 写一个 JSON 值并追加换行 | 写失败时 headers 已发出，无法再改状态码 |
| `json.NewDecoder(r).Decode(&v)` | 读一个 JSON 值 | 读到流末尾返回 `io.EOF`，不是「空输入」 |
| `Decoder.More()` | 当前数组/对象内是否还有元素 | 只用于逐元素流式处理，不能判断「是否还有多余顶层值」 |
| `Decode` 循环 | 处理 NDJSON | 用 `errors.Is(err, io.EOF)` 结束 |

拒绝「一个请求体塞两个 JSON 对象」的正确写法是再解一次并期望 `io.EOF`：

```go
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields() // 严格模式，见 2.5
if err := dec.Decode(&req); err != nil {
	return err
}
var trailing any
if err := dec.Decode(&trailing); !errors.Is(err, io.EOF) {
	return errors.New("请求体只能包含一个 JSON 值")
}
```

### 2.3 struct tag

| tag | 效果 | 陷阱 |
| --- | --- | --- |
| `json:"name"` | 字段名改为 `name` | 名字为空串时回退到 Go 字段名 |
| `json:"name,omitempty"` | 零值时省略该键 | 零值判定见 3.3，struct 永不省略 |
| `json:"-"` | 不参与序列化与反序列化 | 写成 `json:"-,"` 表示「键名就是 `-`」 |
| `json:"age,string"` | 以字符串形式读写数字/布尔 | 只对数字、布尔、字符串生效 |
| 无 tag | 使用 Go 字段名 | v1 解码时字段名匹配不区分大小写 |
| 非导出字段 | 完全忽略 | 不报错、静默丢弃，是「字段消失」的头号根因 |

```go
type Payload struct {
	ID     int64  `json:"id,string"`      // 输出 "id":"123"，避免前端精度丢失
	Secret string `json:"-"`              // 永不出现在 JSON 中
	Nick   string `json:"nick,omitempty"` // 空串时键消失
	token  string                         // 非导出：静默忽略
}
```

### 2.4 RawMessage / Number / any

| 类型 | 含义 | 用途 |
| --- | --- | --- |
| `json.RawMessage` | 尚未解析的原始 JSON 字节 | 延迟解析、转发第三方字段、按 `type` 二次分发 |
| `json.Number` | 数字的字面文本 | 保留大整数精度；解码到该类型字段时直接用字面量填充 |
| `any` | 通用容器 | 一次性读取未知结构 |

`any` 解码后的类型映射是固定的：object → `map[string]any`，array → `[]any`，string → `string`，number → `float64`（除非启用解码器的数字保留选项，见 <https://pkg.go.dev/encoding/json#Decoder>），bool → `bool`，null → `nil`。

```go
type Event struct {
	Type    string          `json:"type"`
	Payload json.RawMessage `json:"payload"`
}

switch ev.Type {
case "login":
	var p LoginPayload
	if err := json.Unmarshal(ev.Payload, &p); err != nil {
		return err
	}
	fmt.Println(p.UserID)
}
```

### 2.5 严格解码

v1 解码器默认忽略未知字段，要用「拒绝未知字段」选项（见 <https://pkg.go.dev/encoding/json#Decoder>）。两点注意：该选项只在流式解码器上，`json.Unmarshal` 没有对应开关；它在遇到未知字段时返回错误，但只给字段名、不给完整路径，需要自己补上下文。这就是生产代码统一走 `Decoder` 的原因之一。

## 3. v1 的坑

### 3.1 数字精度

`float64` 只有 53 位有效二进制位，无法精确表示所有 `int64`。

```go
var v any
_ = json.Unmarshal([]byte(`{"id":9007199254740993}`), &v)
fmt.Printf("%v\n", v.(map[string]any)["id"]) // 9.007199254740992e+15，末位已丢
```

| 场景 | 正确做法 |
| --- | --- |
| 传输数据库主键（`int64` / `uint64`） | 字段声明为 `json.Number`，或序列化时加 `,string` tag |
| 解码到 `any` 后判断整数 | 检查是否可转 `int64`，不要直接 `v.(float64)` |
| 金额 | 以分为单位的整数传输，禁止浮点 |

### 3.2 time.Time 的默认行为

- `Marshal` 输出 RFC 3339 格式（带纳秒与偏移量，如 `2026-10-12T09:30:00Z`）。
- `Unmarshal` 只接受 RFC 3339 及其变体，`"2026-10-12 09:30:00"` 直接报解析错误。
- 零值 `time.Time` 不是空串，而是 `"0001-01-01T00:00:00Z"`，会原样进入 JSON。
- 需要自定义格式（如 `"2026-10-12"`）时，定义专用类型并实现序列化接口（见 <https://pkg.go.dev/encoding/json#Marshaler>），不要把业务时间裸暴露成 `time.Time`。

### 3.3 omitempty 的零值判定

| 类型 | 被省略的条件 |
| --- | --- |
| `bool` | `false` |
| 整数 / 浮点 | `0` |
| `string` | `""` |
| 指针 / 接口 | `nil` |
| slice / map | `nil` 或长度 `0` |
| array | 长度 `0`（`[3]T` 永不省略） |
| struct | **永不省略**，包括零值 `time.Time` 与空的嵌套结构体 |
| 指向 `0` 的指针 | **不省略**，会输出 `"age":0` |

最后一行是「用指针区分缺失与零值」能成立的原理，也是唯一可靠的技巧。

### 3.4 区分「字段缺失」与「字段为零值」

```go
type PatchUser struct {
	// nil => 请求未提供该字段，保持原值；&"" => 请求显式要求清空
	Name *string `json:"name"`
	Age  *int    `json:"age"`
}
```

不要用 `omitempty` 做这件事：`*T` 配合 `omitempty` 在值非 `nil` 时仍会输出，语义正确但容易被误读，宁可显式写 `json:"name"`。

### 3.5 HTML 转义

v1 默认转义 `>`、`<`、`&` 以及 U+2028、U+2029，输出 `\u003c` 这类形式。这是合法 JSON，但会让纯文本 API 返回难读。返回给浏览器时保持默认（纵深防御的一部分）；返回给非 HTML 客户端且需要可读时，用流式编码器的关闭 HTML 转义选项（见 <https://pkg.go.dev/encoding/json#Encoder>）。对 `Unmarshal` 没有影响：`\u003c` 与 `<` 都能正确解析。

### 3.6 循环引用

自引用结构（`a.Next = a`）在 `Marshal` 时会被检测到环并返回错误，而不是栈溢出或死循环。实践建议：DTO 层不出现自引用类型，需要表达图结构时显式用 ID 数组建模。

### 3.7 错误信息不含字段路径

v1 的错误只给类型与原因，不给「是哪个字段」；嵌套到第三层后定位基本靠猜。缓解手段是分层解码——先解到结构体，再用显式校验函数逐字段检查并构造带字段名的错误响应，不要指望 `json` 包替你做路径标注。

## 4. v2 的差异

两处「默认更严格」从实验期到转正都成立：

| 行为 | v1 | v2 |
| --- | --- | --- |
| 字符串中的非法 UTF-8 | 替换为 U+FFFD 继续 | 拒绝，返回错误 |
| 对象中的重复键 | 后者覆盖前者 | 拒绝，返回错误 |

从实验期到转正，一批 tag 与选项被移除或改名：

| 实验期写法 | Go 1.27 状态 | 迁移动作 |
| --- | --- | --- |
| `format` tag 选项 | 移除 | 删除 tag，改由自定义类型实现序列化 |
| `unknown` tag 选项 | 移除 | 删除 tag |
| `DiscardUnknownMembers` 选项 | 移除 | 删除调用，改为解码后自行过滤 |
| `SkipFunc` 选项 | 移除 | 删除调用 |
| `inline` tag | 改名为 `embed` | tag 里的 `inline` 改成 `embed` |
| `string` tag 语义 | 有调整 | 重新对照官方文档确认边界行为 |
| `MatchCaseInsensitiveNames` | 有调整 | 重新对照官方文档确认匹配规则 |
| `jsontext` 的数值 `Token` 访问器 | 改为同时返回错误 | 接收并处理新增的错误返回值 |

上表只列出发布说明明确写到的名字。其余 v2 的函数、选项常量与配置项一律以官方文档为准：<https://pkg.go.dev/encoding/json/v2>、<https://pkg.go.dev/encoding/json/jsontext>，不要凭记忆写 v2 的选项常量名。

迁移清单：

1. 先跑 `go test ./...`，把断言错误文本的测试改成断言错误类型或错误码。
2. 重复键与非法 UTF-8 被拒会暴露上游脏数据：先在接入层加校验，再考虑用 `GOEXPERIMENT=nojsonv2` 临时回退。
3. Go 1.26 起可用 `errors.AsType` 做类型安全的错误提取，适合把解码错误映射成统一错误响应。
4. 回归时在 `GOEXPERIMENT=jsonv2` 与 `GOEXPERIMENT=nojsonv2` 下各跑一遍测试。

## 5. 协议选型对比

| 维度 | `encoding/json`（v1 API） | `encoding/json/v2` | protobuf | msgpack |
| --- | --- | --- | --- | --- |
| 性能 | 反射为主，最慢但够用 | 明显更快，内存占用更低 | 编解码最快，二进制紧凑 | 快于 JSON，体积中等 |
| 可读性 | 直接可读可编辑 | 同 v1 | 需要工具与 `.proto` 才能读 | 需要工具解码 |
| schema 演进 | 无 schema，靠约定 | 同 v1 | 字段编号与可选性内建，演进最强 | 无 schema，靠约定 |
| 默认严格性 | 宽松（重复键、非法 UTF-8 都放过） | 严格（两者都拒绝） | 未知字段可保留，编号驱动 | 宽松 |
| 浏览器友好 | 原生支持 | 原生支持 | 需 gRPC-Web 或网关 | 需要 JS 库 |
| 适用场景 | 对外 REST、配置、调试 | 同 v1，且可接受严格语义 | 服务间高频调用、强契约 | 服务间中等吞吐、要压体积 |
| 引入成本 | 标准库，零依赖 | 标准库，零依赖 | `google.golang.org/protobuf`（以官方最新稳定版为准） | `github.com/vmihailenco/msgpack`（以官方最新稳定版为准） |

选型规则：对外 API 一律 JSON，可读性与可调试性优先于最后 20% 的吞吐；内部服务间调用量大且契约稳定时上 protobuf，契约本身不稳定时上 protobuf 只会让改字段更痛苦；msgpack 只在确认 JSON 的体积或解析开销是瓶颈后才引入；不要在同一份对外 API 上同时提供两种格式，除非有明确的移动端带宽诉求。

## 6. 完整示例：用户 API 的请求/响应与严格解码

目标：可选字段用指针区分「未提供」与「零值」；严格解码拒绝未知字段；统一错误响应；演示 Go 1.26 的 `new(expr)`。

```go
package api

import (
	"encoding/json"
	"errors"
	"io"
	"net/http"
	"strings"
)

// CreateUserRequest 的可选字段都用指针：nil 表示未提供，非 nil 表示显式提供了值。
type CreateUserRequest struct {
	Name  string    `json:"name"`
	Email string    `json:"email"`
	Age   *int      `json:"age"`
	Tags  *[]string `json:"tags"`
}

type UserResponse struct {
	ID    string    `json:"id"`
	Name  string    `json:"name"`
	Email string    `json:"email"`
	Age   *int      `json:"age"`
	Tags  *[]string `json:"tags"`
}

// ErrorResponse 是所有失败路径唯一的响应形状。
type ErrorResponse struct {
	Code    string            `json:"code"`
	Message string            `json:"message"`
	Fields  map[string]string `json:"fields,omitempty"`
}

const maxBodyBytes = 1 << 20 // 1 MiB

func decodeStrict(w http.ResponseWriter, r *http.Request, dst any) error {
	r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	if err := dec.Decode(dst); err != nil {
		return err
	}
	var trailing any
	if err := dec.Decode(&trailing); !errors.Is(err, io.EOF) {
		return errors.New("请求体只能包含一个 JSON 值")
	}
	return nil
}

// 先编码到内存再写状态码，避免编码失败时 headers 已经发出。
func writeJSON(w http.ResponseWriter, status int, v any) {
	buf, err := json.Marshal(v)
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(`{"code":"internal","message":"response encoding failed"}`))
		return
	}
	w.WriteHeader(status)
	_, _ = w.Write(buf)
}

func writeError(w http.ResponseWriter, status int, code, msg string, fields map[string]string) {
	writeJSON(w, status, ErrorResponse{Code: code, Message: msg, Fields: fields})
}

func validateCreateUser(req *CreateUserRequest) map[string]string {
	fields := make(map[string]string)
	if strings.TrimSpace(req.Name) == "" {
		fields["name"] = "不能为空"
	}
	if !strings.Contains(req.Email, "@") {
		fields["email"] = "格式不合法"
	}
	if req.Age != nil && (*req.Age < 0 || *req.Age > 150) {
		fields["age"] = "必须在 0 到 150 之间"
	}
	if len(fields) == 0 {
		return nil
	}
	return fields
}

func handleCreateUser(w http.ResponseWriter, r *http.Request) {
	var req CreateUserRequest
	if err := decodeStrict(w, r, &req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", err.Error(), nil)
		return
	}
	if fields := validateCreateUser(&req); fields != nil {
		writeError(w, http.StatusUnprocessableEntity, "validation_failed", "字段校验失败", fields)
		return
	}
	// 未提供的可选字段保持 nil，序列化后键直接消失。
	writeJSON(w, http.StatusCreated, UserResponse{
		ID: "u_01H", Name: req.Name, Email: req.Email, Age: req.Age, Tags: req.Tags,
	})
}
```

调用侧演示 Go 1.26 的 `new(expr)`；在 Go 1.25 及以前必须写成「先落变量再取地址」。

```go
// Go 1.26+：内建 new 的运算对象可以是表达式。
req := CreateUserRequest{Name: "alice", Email: "alice@example.com", Age: new(26)}

// Go 1.25 及以前：先落变量再取地址，等价但多两行。
age := 26
req = CreateUserRequest{Name: "alice", Email: "alice@example.com", Age: &age}
```

握手验证：`curl -sS -X POST http://127.0.0.1:8080/users -H 'Content-Type: application/json' -d '{"name":"alice","email":"a@example.com","age":0,"nickname":"x"}'`。预期 `nickname` 触发严格模式错误返回 400；把 `age` 设为 `0` 时响应出现 `"age":0`，删除 `age` 键时响应不出现 `age`。

## 7. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `json.Unmarshal` 传值而非指针 | 编译不过 | 目标必须是 `any` 装箱的指针 | 传 `&v` |
| 断言 `err.Error()` 文本 | 升级到 Go 1.27 后测试挂 | 转正后错误文本可能变化 | 断言错误类型或自定义错误码 |
| `if err != nil { return nil }` | 空结构体进入业务层 | 吞掉解码错误 | 包装后返回统一错误响应 |
| 大整数 ID 用 `any` 或 `float64` 接 | 末位数字变化 | `float64` 精度不足 | `json.Number` 或 `,string` tag |
| 用 `omitempty` 判断「是否提供」 | 客户端传 `0` 被当成未提供 | 零值与缺失判定混同 | 可选字段用 `*T` |
| 指望 `omitempty` 省略零值 `time.Time` | JSON 出现 `0001-01-01T00:00:00Z` | struct 永不满足空值 | 用 `*time.Time` 或专用类型 |
| 把 `time.Time` 直接暴露给前端 | 前端解析带纳秒字符串失败 | 默认 RFC 3339 带纳秒与偏移 | 定义时间类型固定格式 |
| 非导出字段期望被序列化 | 字段静默消失 | 只有导出字段参与 | 首字母大写 |
| 见 `\u003c` 就以为输出坏了 | 难读但合法 | 默认 HTML 转义 | 需要可读时关闭转义 |
| 自引用结构直接 `Marshal` | 返回环检测错误 | 存在环 | DTO 用 ID 数组表达关系 |
| 先写状态码再 `Encode` | 编码失败时半截响应无法回滚 | 编码错误发生在写出之后 | 先 `Marshal` 到内存 |
| 把 `More()` 当成「是否还有多余顶层值」 | 多余 JSON 被静默忽略 | `More()` 只用于数组/对象内部 | 再解一次并期望 `io.EOF` |
| 请求体不设上限就 `Decode` | 内存被单个请求打满 | 无体积限制 | `http.MaxBytesReader` 包一层 |
| 依赖 `map[string]any` 的 key 顺序 | 输出不稳定，测试偶发失败 | map 无序 | 用 slice 或显式排序 |
| 全项目依赖开发者本机的 `GOEXPERIMENT` | 换机器行为不一致 | 环境变量是隐式的 | 用构建约束隔离差异 |

## 8. 动手练习

1. 写一个 `codec` 包，提供 `DecodeStrict`（拒绝未知字段与多余顶层值、限制体积）与 `Encode`（先编码到内存），用表驱动测试覆盖未知字段、两个顶层对象、超限体积、非法 UTF-8 四种情况。
2. 给含 `int64` 主键、`time.Time` 创建时间、可选 `*string` 昵称的 `User` DTO 写三版序列化策略（默认、`,string`、专用时间类型），对比输出并在注释里写清适用场景。
3. 造含重复键（`{"id":1,"id":2}`）与非法 UTF-8 字节的请求，分别在 `GOEXPERIMENT=jsonv2`、`GOEXPERIMENT=nojsonv2`、不设置三种情况下跑同一段测试，记录行为与错误文本差异。
4. 把一个「用 `omitempty` 判断字段是否提供」的旧 handler 重构为指针方案，并补一个「显式传 `0`」的回归测试。
5. 用 `json.RawMessage` 实现事件分发器：外层只有 `type` 与 `payload`，按 `type` 解到不同结构体，未知 `type` 返回统一错误响应。

## 9. 自检清单

- [ ] 能说出 `encoding/json/v2` 在 1.25 与 1.27 的状态及启用/回退开关。
- [ ] 能说明 `go.mod` 没有 GOEXPERIMENT 指令，只能通过 `go` 行与 `toolchain` 行间接控制。
- [ ] 写过 `//go:build goexperiment.jsonv2` 与取反版本的分文件，两种设置下都构建通过。
- [ ] 代码里没有对 JSON 错误文本的断言。
- [ ] 所有 `int64` 主键在对外 JSON 里都不会丢精度。
- [ ] 可选字段一律用 `*T`，且能解释 `*T` + `omitempty` 的实际行为。
- [ ] 知道 `time.Time` 零值不会被 `omitempty` 省略，并已用指针或专用类型处理。
- [ ] 每个写 JSON 的 handler 都走「先编码到内存再写状态码」。
- [ ] 每个读 JSON 的 handler 都限制体积并开启严格模式。
- [ ] 已删除实验期的 `format`、`unknown`、`DiscardUnknownMembers`、`SkipFunc` 用法，并把 `inline` 改成 `embed`。
- [ ] 能解释为什么对外 API 用 JSON、内部高频调用才考虑 protobuf。

## 10. 延伸阅读

- Go 1.25 Release Notes：<https://go.dev/doc/go1.25>
- Go 1.26 Release Notes：<https://go.dev/doc/go1.26>
- Go 1.27 Release Notes：<https://go.dev/doc/go1.27>
- `encoding/json`：<https://pkg.go.dev/encoding/json>
- `encoding/json/v2`：<https://pkg.go.dev/encoding/json/v2>
- `encoding/json/jsontext`：<https://pkg.go.dev/encoding/json/jsontext>
- `time` 包与 RFC 3339：<https://pkg.go.dev/time>
- 书籍：《Go程序设计语言》，Alan A. A. Donovan、Brian W. Kernighan，机械工业出版社
- 书籍：《Go语言实战》，William Kennedy，人民邮电出版社
- 书籍：《Go语言高级编程》，柴树杉、曹春晖，人民邮电出版社
