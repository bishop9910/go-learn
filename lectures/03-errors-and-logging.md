# L03 错误处理与结构化日志

本篇定位：把「错误如何产生、如何包装、如何判定、如何对外暴露」定成项目规范，并给出 `log/slog` 的落地写法，让日志从拼字符串升级为可检索的结构化事件。
对应学习周：W1–W2。
前置要求：完成 [L02 语言精要与易错点](02-language-essentials.md)，特别是接口的 nil 陷阱与方法集。
版本纪律：`errors.AsType` 与 `log/slog.NewMultiHandler` 需要 **Go 1.26+**；`slog.GroupAttrs` 与 `slog.Record.Source` 需要 **Go 1.25+**；其余示例在本机 `go1.25.7` 下可编译。

---

## 1. error 接口与错误的产生

`error` 只有一个方法：`Error() string`。任何实现了它的类型都是错误，这也埋下了 L02 讲过的 nil 陷阱。

| 方式 | 写法 | 适用场景 |
| --- | --- | --- |
| 哨兵错误 | `var ErrNotFound = errors.New("store: not found")` | 调用方需要按**类型**判定同一类错误 |
| 即时构造 | `errors.New("db: connection refused")` | 一次性、不打算被判定 |
| 带上下文包装 | `fmt.Errorf("load user %d: %w", id, err)` | 传递过程中补充「在哪一步失败」 |

```go
// 哨兵错误：命名以 Err 开头，声明为包级 var，错误文本小写且不带结尾标点。
var (
	ErrNotFound = errors.New("store: not found")
	ErrConflict = errors.New("store: conflict")
)
```

只在**调用方确实需要判定**时才设哨兵，滥用会让包的错误表面积无限膨胀。

### 1.1 `%w` 与 `%v` 的区别

```go
base := store.ErrNotFound

wrapped := fmt.Errorf("service: load user: %w", base) // 建立错误链
plain := fmt.Errorf("service: load user: %v", base)   // 只保留文本

fmt.Println(errors.Is(wrapped, store.ErrNotFound)) // true
fmt.Println(errors.Is(plain, store.ErrNotFound))   // false
```

| 格式动词 | 是否建立错误链 | `errors.Is` / `errors.As` 能否穿透 |
| --- | --- | --- |
| `%w` | 是（实现 `Unwrap() error`） | 能 |
| `%v` / `%s` | 否（只是文本） | 不能 |

**这是最常见的错误处理 bug 来源**：日志里能看到完整原因，代码里 `errors.Is` 却判定不出来。规范：向上传递原始错误时一律用 `%w`；`%v` 只用于「不再需要判定、只想留下线索」的场合。

### 1.2 多错误包装

```go
var errs []error
for _, f := range files {
	if err := process(f); err != nil {
		errs = append(errs, fmt.Errorf("process %s: %w", f, err))
	}
}
if err := errors.Join(errs...); err != nil { // Go 1.20 引入
	return fmt.Errorf("batch: %w", err)
}
```

`errors.Join` 返回的错误实现了返回 `[]error` 的 `Unwrap`，因此 `errors.Is` 会检查每一个分支；`errors.Join(nil, nil)` 返回 `nil`，不必预先判断长度。`fmt.Errorf` 也支持同一格式串中出现多个 `%w`（Go 1.20 起），效果类似。

---

## 2. 错误的判定

### 2.1 `errors.Is` 的遍历规则

`errors.Is(err, target)` 按以下顺序检查，任一命中即返回 `true`：

1. `err == target`（可比较时）。
2. `err` 实现了 `Is(error) bool` 且返回 `true`。
3. `err` 实现了 `Unwrap() error`：递归检查该结果。
4. `err` 实现了 `Unwrap() []error`：对每个子错误递归检查。

### 2.2 `errors.As` 的遍历规则

`errors.As(err, &target)` 沿同样的链遍历，找到**第一个可赋值给目标类型**的错误并写入，返回 `true`。

```go
var qe *QueryError
if errors.As(err, &qe) {
	logger.Warn("query failed", "query", qe.Query)
}
```

| 项 | 说明 |
| --- | --- |
| 第二个参数 | 必须是指向「实现了 error 的类型」的指针，否则 panic |
| 匹配条件 | 链上任一错误的**动态类型**可赋值给目标类型 |
| 只取第一个 | 命中后立即返回，多错误链中不会继续找 |
| 与 `Is` 的分工 | `Is` 判「是不是这类错误」，`As` 取「这类错误里的数据」 |

### 2.3 `errors.AsType`（Go 1.26+）

**Go 1.26 起**提供泛型版 `errors.AsType`，省掉「先声明变量再取地址」的样板，且类型安全、更快：

```go
// 需要 Go 1.26 及以上
if qe, ok := errors.AsType[*QueryError](err); ok {
	logger.Warn("query failed", "query", qe.Query)
}
```

对比 1.25 及以前必须写 `var qe *QueryError; errors.As(err, &qe)`。精确签名与类型参数约束以官方文档为准（<https://pkg.go.dev/errors>）；在写成 `go 1.25.0` 的模块里使用它，`go vet` 的 `stdversion` 检查（Go 1.27 起默认开启）会直接报错。

### 2.4 判定方式的选择

| 需求 | 用什么 |
| --- | --- |
| 判断「是不是这一类错误」 | `errors.Is(err, ErrXxx)` |
| 取出错误里的结构化字段 | `errors.As`，或 Go 1.26+ 的 `errors.AsType` |
| 判断「有没有任何一个子错误满足」 | `errors.Is` 天然支持多错误链 |
| `err == ErrXxx` 直接比较 | 只在同包内且确定没被包装过时使用 |

---

## 3. 自定义错误类型

需要「携带数据供调用方读取」「自定义相等判定」「自定义展示」三者之一时，才值得定义类型；否则 `errors.New` 加 `%w` 足够。

```go
type QueryError struct {
	Op    string
	Query string
	Err   error // 底层原因，可能为 nil
}

func (e *QueryError) Error() string {
	if e.Err == nil {
		return fmt.Sprintf("store: %s %q failed", e.Op, e.Query)
	}
	return fmt.Sprintf("store: %s %q failed: %v", e.Op, e.Query, e.Err)
}

// Unwrap 让 errors.Is / errors.As 能穿透到 e.Err。
func (e *QueryError) Unwrap() error { return e.Err }
```

| 项 | 约定 |
| --- | --- |
| 接收者 | 用指针接收者，避免大结构体复制 |
| `Error()` 里如何拼底层原因 | 用 `%v` 拼进文本即可，**不要调用 `Unwrap`**（那是判定用的） |
| `Unwrap()` 返回 nil | 没有底层原因时返回 nil，`errors.Is` 能正确处理 |
| 构造入口 | 提供 `NewQueryError(...) error` 便捷函数，避免调用方写出 typed nil |
| 不要把 `Unwrap` 用于「返回同级错误」 | 那是 `errors.Join` 的场景 |

自定义相等判定用 `Is`：

```go
// Code 是稳定的业务错误码。
type Code string

type Error struct {
	Code Code
	Msg  string
}

func (e *Error) Error() string { return string(e.Code) + ": " + e.Msg }

// Is 定义：目标也是 *Error 且 Code 相同即同类。
func (e *Error) Is(target error) bool {
	t, ok := target.(*Error)
	return ok && e.Code == t.Code
}
```

---

## 4. 错误处理策略

### 4.1 只在能处理的地方处理

处理 = 做了某个决定：重试、降级、给用户默认值、翻译成别的类型、终止流程。以下都不算处理：

```go
data, _ := os.ReadFile(path) // 反例一：把错误吃掉

if err != nil {
	log.Println("read failed:", err) // 反例二：只打日志不返回，控制流继续
}

if err != nil {
	return err // 反例三：深层调用里原样返回，等于没处理
}
```

### 4.2 包装时补充上下文

每层只补「本层独有的信息」，不要复述上一层（别层层写 `failed: %w`）。格式约定：`本层操作 + 对象标识 + : %w`，错误文本首字母小写、不带句号，因为会被拼接。

```go
return fmt.Errorf("load user %d: %w", id, err)
```

错误最终会进日志，因此**不要把密码、token、完整 SQL 参数拼进错误文本**。

### 4.3 不要用 error 当控制流

「未找到」在多数业务里是正常结果。判据：**如果这个「错误」在正常流量里出现的频率与成功同量级，它就不是错误**——应改为返回 `(T, bool)` 或约定 nil 指针表示不存在，让错误只覆盖真正的异常路径。

### 4.4 边界处统一翻译

| 层 | 错误职责 |
| --- | --- |
| repository / store | 产生底层错误（驱动错误、哨兵错误），保留原始细节 |
| service / usecase | 用 `%w` 包装并补充业务上下文；把驱动错误翻译为领域哨兵 |
| handler / transport | 决定 HTTP 状态码与响应体；**不把内部错误文本回给客户端**；在此处记录完整日志 |

统一错误响应结构体形如 `APIError{Code string; Message string}`（JSON 字段为 `code` 与 `message`），`Code` 是稳定的机器可读标识，`Message` 是给用户看的通用文案：

```go
func writeError(w http.ResponseWriter, logger *slog.Logger, err error) {
	status, code := http.StatusInternalServerError, "INTERNAL"
	switch {
	case errors.Is(err, store.ErrNotFound):
		status, code = http.StatusNotFound, "NOT_FOUND"
	case errors.Is(err, store.ErrConflict):
		status, code = http.StatusConflict, "CONFLICT"
	case errors.Is(err, context.DeadlineExceeded):
		status, code = http.StatusGatewayTimeout, "TIMEOUT"
	}
	logger.Error("request failed", "status", status, "code", code, "err", err) // 完整错误只进日志

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(APIError{Code: code, Message: messageFor(code)})
}
```

---

## 5. panic 与 recover 的边界

| 场景 | 是否可用 panic |
| --- | --- |
| 程序初始化阶段的不可恢复配置错误 | 可用，快速失败优于带错运行 |
| 逻辑上不可能到达的分支 | 可用 |
| 任何业务错误（参数非法、资源不存在、余额不足） | **绝不可用**，必须返回 error |
| 库代码遇到可预期的错误 | **绝不可用**，库不应 panic，应返回 error 让调用方决定 |
| 已导出 API 收到非法参数 | 返回 error；仅在调用方明确违反文档承诺时才 panic |

库代码不 panic 的理由很实际：库无法知道调用方的进程有多关键，一次 panic 会终止整个进程，可能连带杀死同进程内所有无关请求。

**Go 1.21 起，`panic(nil)` 产生的 panic 值不再是 nil。** 以前 `recover()` 返回 nil 既可能是「没有 panic」也可能是「panic 了 nil 值」，调用方无法判断；现在 panic 值必定非 nil，`recover()` 的返回值可以直接用来判断是否真的发生了 panic。迁移期间可用对应的 `GODEBUG` 开关临时回退旧行为，具体开关名与生命周期以官方发布说明为准。

**Go 1.25 起**，未被处理的 panic 输出改为带标记的形式：`panic: X [recovered, repanicked]`，表示该 panic 曾被 `recover` 捕获后又重新抛出。运维告警若在匹配 panic 文本，需要同步更新规则。

`recover` 的唯一正当用法是在**请求/进程边界**兜底：

```go
func safeHandler(logger *slog.Logger, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				logger.Error("panic recovered", "panic", rec, "path", r.URL.Path,
					"stack", string(debug.Stack())) // runtime/debug，见 https://pkg.go.dev/runtime/debug
				http.Error(w, "internal error", http.StatusInternalServerError)
			}
		}()
		next(w, r)
	}
}
```

要点：`recover` 只在 `defer` 的函数里有效且必须直接位于该函数体内；`recover` 后**不要静默继续**，至少要记日志；并发读写 map 触发的是 `fatal error`，`recover` 无效（见 [L02](02-language-essentials.md)）。

---

## 6. `log/slog` 结构化日志

`log/slog` 自 Go 1.21 起进入标准库（<https://pkg.go.dev/log/slog>），业务日志只用它，不再用旧的 `log` 包。

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)

logger.Info("server starting", "addr", ":8080", "env", "dev")
logger.Warn("slow query", "table", "orders", "ms", 320)

reqLog := logger.With("request_id", "req-1", "trace_id", "tr-1").WithGroup("db")
reqLog.Error("query failed", "op", "select", "err", "connection refused")
```

| 构造 | 用途 |
| --- | --- |
| `slog.New(h)` | 用指定 handler 建 logger |
| `slog.NewTextHandler(w, opts)` | 人类可读的 `key=value` 文本，适合本地开发 |
| `slog.NewJSONHandler(w, opts)` | 每行一个 JSON 对象，适合采集与检索 |
| `slog.SetDefault(l)` | 替换包级默认 logger |
| `slog.Level` | 级别类型：`slog.LevelDebug` / `LevelInfo` / `LevelWarn` / `LevelError` |
| `l.With(...)` / `l.WithGroup(name)` | 返回带固定属性 / 进入分组的子 logger |

键值对是**交替的可变参数**（`"k", v, "k2", v2`）；奇数个参数会产生一条 `!BADKEY` 属性，不 panic 但会污染日志。`With` 与 `WithGroup` 都返回新 logger，**不修改原 logger**。

### 6.1 Go 1.25：`slog.GroupAttrs` 与 `slog.Record.Source`

**`slog.GroupAttrs`（Go 1.25+）** 把若干属性在**单条日志**内组合成一个分组，不需要先建子 logger：

```go
// 需要 Go 1.25 及以上
logger.LogAttrs(nil, slog.LevelInfo, "request done",
	slog.GroupAttrs("http", slog.String("method", "GET"), slog.Int("status", 200)),
	slog.GroupAttrs("db", slog.Int("queries", 3)),
)
```

输出形如 `{"msg":"request done","http":{...},"db":{...}}`。需要「之后所有事件都带上某组属性」用 `WithGroup`，需要「只在这一条里成组」用 `GroupAttrs`。

**`slog.Record.Source`（Go 1.25+）** 让 handler 能拿到日志调用点的源码位置：在 handler 选项里要求记录源信息后，即可在 `Handle` 里通过 `Record` 的 `Source` 读出文件、行号与函数。生产环境通常**关闭**源信息采集（每次调用要做运行栈解析），只在排障时临时开启。

### 6.2 Go 1.26：`log/slog.NewMultiHandler`

**Go 1.26 起**，`log/slog` 提供 `NewMultiHandler`，把一条日志同时分发给多个 handler：

```go
// 需要 Go 1.26 及以上
file, err := os.Create("app.log")
if err != nil {
	panic(err)
}
defer file.Close()

logger := slog.New(slog.NewMultiHandler(
	slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}),
	slog.NewJSONHandler(file, &slog.HandlerOptions{Level: slog.LevelWarn}),
))

logger.Info("only stdout")
logger.Error("stdout and file")
```

- `MultiHandler` 会调用**所有** handler，过滤由各 handler 自己的级别设置负责。
- 每个 handler 拿到同一份 `Record`；需要消费属性时先克隆记录（`slog.Record` 的克隆能力见官方文档），避免影响其他消费者。
- 各 handler 的写入错误由自己负责；对关键审计日志应实现自定义 handler 把写入错误暴露出来。

### 6.3 自定义 `Handler` 的最小实现思路

| 方法 | 职责 |
| --- | --- |
| `Enabled(ctx, level) bool` | 该级别是否需要处理；返回 false 时 `Handle` 不会被调用 |
| `Handle(ctx, r Record) error` | 把一条记录写成字节；写失败要返回 error |
| `WithAttrs(attrs []Attr) Handler` | 返回带这些固定属性的新 handler |
| `WithGroup(name string) Handler` | 返回进入该分组的 handler |

两条纪律：**`WithAttrs` / `WithGroup` 必须返回新值**，不能修改自身，否则并发使用会串数据；**不要直接改写 `Record` 后转发**，先克隆再添加属性。脱敏 handler 的典型做法是在 `Handle` 里遍历属性，命中敏感键名（`password`、`token`）时替换为固定占位符，再交给内层 handler。

### 6.4 日志级别规范

| 级别 | 判定标准 | 示例 |
| --- | --- | --- |
| `Debug` | 只在排障时打开 | 缓存命中情况、解析后的请求结构 |
| `Info` | 业务上值得记录的正常事件，量可控 | 服务启动、请求完成摘要、任务完成 |
| `Warn` | 不影响本次成功，但预示风险 | 重试后成功、慢查询、降级到默认值 |
| `Error` | 本次操作失败且需要人介入 | 依赖不可用、数据不一致、未处理 panic |

`Error` 必须稀有，否则告警失去意义，可恢复的重试不要打 `Error`。同一错误只在**边界层**记一次，中间层只包装不记录。

### 6.5 字段命名与敏感信息

| 字段 | 含义 | 约束 |
| --- | --- | --- |
| `request_id` | 单次请求标识 | 入口生成，全程通过 `With` 传递 |
| `trace_id` | 分布式追踪标识 | 由网关传入，缺失时留空而不是编造 |
| `user_id` | 主体标识 | 不得写用户名、邮箱等直接标识 |
| `path` / `method` / `status` | HTTP 维度 | 只在接入层记录 |
| `duration_ms` | 耗时 | 单位写进字段名 |
| `err` | 错误对象 | 传 `error` 值本身，不要先 `err.Error()` |

规则：字段全部**小写加下划线**；`msg` 只写「发生了什么」，动态内容一律进字段，让 `msg` 的基数可控。

**禁止记录**：密码与密码哈希、access / refresh token 与 session id、身份证号与银行卡号与手机号、Cookie 与 `Authorization` 头（请求头按白名单记录而非黑名单排除）、完整请求体（默认不记）。工程手段是**白名单 + 脱敏 handler** 双保险。

### 6.6 采样与限流

| 手段 | 做法 | 适用 |
| --- | --- | --- |
| 级别开关 | 生产 `Info` 起步，`Debug` 按开关临时打开 | 全局 |
| 采样 | 同类事件按比例记录；必须带「已采样比例」字段，否则统计失真 | 高频成功事件 |
| 限流 | 同一错误指纹在时间窗内只记第一次，之后累计次数汇总输出 | 依赖故障引发的错误风暴 |
| 聚合 | 循环内累计计数与首个错误样本，循环后输出一条汇总 | 批处理 |

反模式是在循环里逐条打日志：100 万行数据会产出 100 万条日志，既拖慢处理又淹没信息。

---

## 7. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `fmt.Errorf("load user: %v", err)` 后调用方 `errors.Is` | 判定永远为 false | `%v` 不建立错误链 | 需要判定时用 `%w` |
| `return err` 层层原样上传 | 日志只有最底层信息 | 没有补充本层上下文 | 每层 `fmt.Errorf("本层操作: %w", err)` |
| `logger.Info("user " + id + " failed, err=" + err.Error())` | 无法按字段检索或聚合 | 结构化信息拼进了字符串 | `logger.Info("failed", "user_id", id, "err", err)` |
| 循环体内逐条 `logger.Info` | 日志量爆炸、吞吐下降 | 没有聚合 | 循环内计数，循环后打一条汇总 |
| `data, _ := os.ReadFile(p)` | 上层拿零值继续跑 | 错误被丢弃且无日志 | 至少 `if err != nil { return err }`；丢弃要写注释说明 |
| 库代码里 `log.Fatal` / `os.Exit` | 调用方进程被杀，defer 不执行 | 库越权决定调用方进程生命周期 | 库返回 error，由 `main` 决定退出 |
| `errors.As(err, target)` 传了非指针 | 运行时 panic | 第二个参数必须是指向 error 类型的指针 | 传 `&target`；Go 1.26+ 优先 `errors.AsType` |
| handler 把 `err.Error()` 放进响应体 | 泄漏内部路径、SQL 或依赖版本 | 内部错误未经翻译即外泄 | 只回稳定 `code` 与通用文案，细节只进日志 |
| 用 `panic` 表达「参数不合法」 | 一个坏请求打死整个进程 | 把业务错误当程序错误 | 返回 error，在边界翻译成 4xx |
| 日志记录 `Authorization` 头或完整请求体 | 凭据泄漏进日志系统 | 没有脱敏层 | 白名单字段 + 脱敏 handler |
| `err == store.ErrNotFound` 判定 | 一旦被 `%w` 包装过就失效 | `==` 不穿透错误链 | 用 `errors.Is` |

---

## 8. 动手练习

1. **`%w` vs `%v`**：写三层调用链（store → service → handler），分别用 `%w` 与 `%v` 包装，用 `errors.Is` 验证差异并对比完整错误文本。
2. **自定义错误**：实现带 `Op`、`Query`、`Err` 的 `QueryError` 并支持 `Unwrap`；再实现一个带业务码且覆盖 `Is` 的 `Error` 类型，验证「Code 相同即同类」。
3. **`errors.AsType`**：用 Go 1.26+ 工具链把练习 2 的 `errors.As` 改写为 `errors.AsType`，并在文件头注释标注所需版本。
4. **日志分级**：给一个 HTTP handler 加 `request_id`、`method`、`path`、`status`、`duration_ms`，成功走 `Info`、4xx 走 `Warn`、5xx 走 `Error`，用 `JSONHandler` 输出。
5. **脱敏 handler**：实现包装 handler，把 `password` / `token` 的值替换为 `***`，并用单元测试断言输出中不含明文。
6. **panic 边界**：写一个兜底 recover 的 middleware，验证 `panic("boom")` 与 `panic(nil)` 两种输入下 `recover()` 的返回值。

---

## 9. 自检清单

- [ ] 能说出 `%w` 与 `%v` 在错误链上的差异，并知道何时用哪个
- [ ] 能描述 `errors.Is` 的四步遍历规则
- [ ] 会用 `errors.As` 取错误字段，知道第二个参数必须是指针
- [ ] 明确知道 `errors.AsType` 需要 **Go 1.26+**
- [ ] 会自定义错误类型并正确实现 `Error()` / `Unwrap()` / `Is()`
- [ ] 知道错误该在哪一层包装、哪一层记录、哪一层翻译成 HTTP 响应
- [ ] 能说明「用 error 当控制流」的问题并给出替代设计
- [ ] 知道库代码不应 panic、不应调用 `log.Fatal`
- [ ] 能说明 Go 1.21 起 `panic(nil)` 的行为变化
- [ ] 知道 Go 1.25 起未处理 panic 的输出带 `[recovered, repanicked]` 标记
- [ ] 能用 `slog.New` + `NewJSONHandler` 输出结构化日志，并用 `With` 贯穿 `request_id`
- [ ] 会用 `WithGroup` 与 `GroupAttrs`（Go 1.25+）并说清两者区别
- [ ] 明确知道 `slog.NewMultiHandler` 需要 **Go 1.26+**，且它会调用所有 handler
- [ ] 能说出结构化日志的字段命名规则与敏感字段清单
- [ ] 知道高流量下用级别开关、采样、限流、聚合控制日志量

---

## 10. 延伸阅读

- Go 1.20 发布说明（`errors.Join`、多 `%w`）：<https://go.dev/doc/go1.20>
- Go 1.21 发布说明（`log/slog` 进入标准库）：<https://go.dev/doc/go1.21>
- Go 1.25 发布说明（`slog.GroupAttrs`、`slog.Record.Source`、panic 输出格式）：<https://go.dev/doc/go1.25>
- Go 1.26 发布说明（`errors.AsType`、`log/slog.NewMultiHandler`）：<https://go.dev/doc/go1.26>
- `errors` 包：<https://pkg.go.dev/errors>；`log/slog` 包：<https://pkg.go.dev/log/slog>
- 错误处理的官方讨论：<https://go.dev/blog/error-handling-and-go>
- 错误包装与 `errors.Is`：<https://go.dev/blog/go1.13-errors>
- 书籍：*Go 语言编程*，许式伟等，人民邮电出版社
- 相关讲义：[L02 语言精要与易错点](02-language-essentials.md)、[L04 标准库核心](04-stdlib-core.md)
