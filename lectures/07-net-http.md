# L07 net/http 与 Web 服务

本篇定位：只用标准库把 HTTP 服务写对——路由、中间件、请求解析、响应、客户端、安全与优雅关闭。对应 W3–W4（后端项目落地）。前置要求：会用 `context` 取消与超时，读过[L06 测试工程](06-testing.md)里的 `httptest`，能写[L05 序列化](05-serialization-json.md)里的严格 JSON 解码。

## 1. Handler 与路由

| 类型 | 要点 | 用途 |
| --- | --- | --- |
| `http.Handler` | 只有一个处理请求的方法 | 中间件与路由都围绕它组合 |
| `http.HandlerFunc` | 让普通函数满足 `http.Handler` 的适配器 | 写内联 handler，不必定义新类型 |
| `http.ServeMux` | 标准库路由器，本身也是 `http.Handler` | 按 pattern 分发到子 handler |

Go 1.22 起 `ServeMux` 的 pattern 支持方法与通配符：

| pattern | 匹配 |
| --- | --- |
| `/users/{id}` | 单段通配符，用 `r.PathValue("id")` 取值，未匹配为空串 |
| `/files/{path...}` | 多段通配符，只能出现在末尾 |
| `GET /users/{id}` | 限定方法，方法不匹配返回 405 |
| `/users/{$}` | 只匹配 `/users/`，不匹配 `/users/42` |

优先级规则：**越具体的 pattern 胜出**，与注册顺序无关。`/users/{id}/posts` 比 `/users/{id}` 具体，多段通配符最宽泛，方法限定参与同一套比较；`/` 是兜底 pattern。

### 1.1 尾斜杠重定向：Go 1.26 起由 301 改为 307

`ServeMux` 对尾斜杠不匹配的请求会重定向。Go 1.26 起状态码由 **301 改为 307**：301 会让客户端把非 GET 请求也当作 GET 重发（丢掉方法与请求体），307 保留方法与请求体。依赖旧行为的客户端或网关需要显式注册尾斜杠路由并自行处理。

## 2. http.Server 的安全默认值

零值 `http.Server` 没有任何超时，慢客户端能长期占住连接。生产配置必须显式设置：

| 字段 | 作用 | 建议 |
| --- | --- | --- |
| `ReadHeaderTimeout` | 只限制读请求头的时长 | 必设，5s 级；Slowloris 类攻击的第一道闸 |
| `ReadTimeout` | 限制读整个请求（含 body） | 有上传接口时按最大上传耗时放大 |
| `WriteTimeout` | 限制写响应的时长 | SSE 与流式响应要设 0 或很大，否则连接被掐断 |
| `IdleTimeout` | keep-alive 空闲连接存活时长 | 比上游代理的 idle 略短，避免用到已关闭的连接 |
| `MaxHeaderBytes` | 请求头字节上限 | 默认 1MB，内网服务可调小 |
| `MaxHeaderValueCount` | 单个请求头值的数量上限 | Go 1.27 新增，不设置时取 `DefaultMaxHeaderValueCount` |
| `Handler` | 根 handler | 挂路由或中间件链 |

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           router,
	ReadHeaderTimeout: 5 * time.Second,   // 必设
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      60 * time.Second,  // 有 SSE 时改为 0
	IdleTimeout:       90 * time.Second,
	MaxHeaderBytes:    1 << 16,
}
```

`MaxHeaderValueCount` 与 `DefaultMaxHeaderValueCount` 的语义见 <https://pkg.go.dev/net/http#Server>。

## 3. 优雅关闭

目标：收到终止信号后停止接收新连接，等在途请求结束，再收尾后台任务。

```go
func main() {
	srv := &http.Server{Addr: ":8080", Handler: router, ReadHeaderTimeout: 5 * time.Second}
	srv.RegisterOnShutdown(flushMetrics) // 关闭钩子：先于「等待在途请求」执行

	// Go 1.26 起 NotifyContext 的取消函数是 context.CancelCauseFunc，可带取消原因。
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			stop(err)
		}
	}()
	<-ctx.Done()
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		slog.Error("shutdown", "err", err)
	}
}
```

| 要点 | 说明 |
| --- | --- |
| `Shutdown(ctx)` | 停止监听并等待在途请求，ctx 到期后强制返回 |
| 不要用 `Close()` | 立即断开所有连接，在途请求全部失败 |
| `http.ErrServerClosed` | `ListenAndServe` 在 `Shutdown` 后的正常返回，必须排除 |
| `RegisterOnShutdown` | 适合关连接池、停定时任务、上报指标，不要在里面长阻塞 |
| 长连接 | SSE、WebSocket 之类的长连接不随 `Shutdown` 立即结束，业务层要监听取消信号主动收尾 |
| 后台任务 | 用 `context` 串联，让请求取消与进程关闭走同一条取消链 |

## 4. 中间件

中间件就是 `func(http.Handler) http.Handler`：收到下一个 handler，返回一个新的 handler。

```go
func requestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" {
			id = newID()
		}
		w.Header().Set("X-Request-ID", id)
		next.ServeHTTP(w, r.WithContext(withRequestID(r.Context(), id)))
	})
}
func recoverer(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if v := recover(); v != nil {
				slog.Error("panic", "value", v, "request_id", requestIDFrom(r.Context()))
				writeError(w, http.StatusInternalServerError, "internal", "服务内部错误")
			}
		}()
		next.ServeHTTP(w, r)
	})
}
func logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		slog.Info("http", "method", r.Method, "path", r.URL.Path,
			"duration_ms", time.Since(start).Milliseconds())
	})
}
```

超时与 CORS 用同样形状写：超时是 `context.WithTimeout(r.Context(), d)` 后 `next.ServeHTTP(w, r.WithContext(ctx))`；CORS 只在 `Origin` 命中白名单时回写允许头（并设置 `Vary: Origin`），对 `OPTIONS` 直接返回 204。

顺序为什么重要（越靠外越先执行）：

| 顺序 | 中间件 | 原因 |
| --- | --- | --- |
| 1 | recover | 必须最外层，才能兜住内层所有 panic |
| 2 | 请求 ID | 内层日志与错误响应都要带 ID |
| 3 | 结构化日志 | 要记录真实状态码与耗时，必须在业务之外 |
| 4 | 超时 | 给业务与下游调用统一的时间上限 |
| 5 | CORS | 预检请求要在到达业务前短路 |
| 6 | 鉴权 | 只让已认证请求进入业务逻辑 |
| 7 | 业务 | 最内层 |

顺序由嵌套决定：`recoverer(requestID(logger(timeout(3*time.Second)(cors(origin)(mux)))))`。

## 5. 请求解析

### 5.1 体积限制与流式解码

```go
func decodeBody(w http.ResponseWriter, r *http.Request, dst any) error {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1MiB，超限报错，应映射为 413
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
```

顺序不能反：先限制体积再解析；`json.Decoder` 是流式的，不把整个 body 读进内存，但没有 `MaxBytesReader` 就等于没有上限。另外要校验 `Content-Type`，非 `application/json` 直接返回 415，避免解析歧义。

### 5.2 表单与 multipart 上传

| API | 用途 | 注意 |
| --- | --- | --- |
| `r.ParseForm` / `r.FormValue` | 表单与查询参数 | 会消耗 body，用哪条路径要先定 |
| `r.ParseMultipartForm` | 解析 multipart，参数是内存中保留的最大字节数 | 超出部分落临时文件，总量仍要 `MaxBytesReader` 兜 |
| `mime/multipart` | 手工遍历 multipart 部件 | 需要流式处理大文件时用 |
| `mime/multipart.FileContentDisposition` | Go 1.25 新增：处理 Content-Disposition | 签名见官方文档 |

```go
func upload(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 32<<20) // 总量上限
	if err := r.ParseMultipartForm(4 << 20); err != nil { // 4MiB 内留内存，其余落盘
		writeError(w, http.StatusBadRequest, "bad_multipart", err.Error())
		return
	}
	file, _, err := r.FormFile("avatar")
	if err != nil {
		writeError(w, http.StatusBadRequest, "missing_file", "缺少 avatar 字段")
		return
	}
	defer file.Close() // header.Filename 不可信：只取 filepath.Base，校验落盘目录后再 io.Copy
}
```

### 5.3 路径与查询参数校验

```go
func listUsers(w http.ResponseWriter, r *http.Request) {
	limit, err := strconv.Atoi(r.URL.Query().Get("limit"))
	if err != nil || limit <= 0 || limit > 100 {
		writeError(w, http.StatusBadRequest, "invalid_limit", "limit 必须是 1..100")
		return
	}
	if !validID(r.PathValue("id")) {
		writeError(w, http.StatusBadRequest, "invalid_id", "id 格式不合法")
	}
}
```

路径参数与查询参数一律当不可信输入，先校验再进业务；失败返回 400 并带上字段名。

## 6. 响应

| 目标 | 做法 |
| --- | --- |
| 纯文本错误 | `http.Error(w, msg, code)`，自动设置 Content-Type 与状态码 |
| JSON | 先 `json.Marshal` 到内存，再设 Content-Type、写状态码、写 body |
| 静态文件 | `http.ServeFile` 单个文件；`http.ServeContent` 支持 Range 与条件请求 |
| 流式输出 | 断言 `w.(http.Flusher)` 并周期性 Flush |
| SSE | `Content-Type: text/event-stream`，逐条写 `data: ...` 并以空行结束，写完立即 Flush |

```go
func stream(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/event-stream")
	flusher, ok := w.(http.Flusher)
	if !ok {
		http.Error(w, "streaming unsupported", http.StatusInternalServerError)
		return
	}
	for i := 0; i < 10; i++ {
		select {
		case <-r.Context().Done(): // 客户端断开或超时
			return
		case <-time.After(time.Second):
		}
		fmt.Fprintf(w, "data: tick %d\n\n", i)
		flusher.Flush()
	}
}
```

三条硬要求：正确的 Content-Type、事件以空行结束、每次写完立即 Flush。此外必须监听 `r.Context().Done()`，否则客户端断开后 goroutine 一直挂着；同时注意第 2 节的 `WriteTimeout` 会掐断长连接。

## 7. 客户端

```go
var httpClient = &http.Client{
	Timeout: 10 * time.Second, // 覆盖连接、重定向与读取的总时长
	Transport: &http.Transport{
		MaxIdleConns:        100,              // 全局空闲连接上限
		MaxIdleConnsPerHost: 20,               // 每个 host 的空闲连接上限
		IdleConnTimeout:     90 * time.Second, // 空闲连接回收
	},
}

resp, err := httpClient.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
_, _ = io.Copy(io.Discard, resp.Body) // 排空后再关闭，否则连接无法复用
```

| 问题 | 说明 |
| --- | --- |
| 为什么不用 `http.DefaultClient` | 没有超时，且被全进程共享，改一处影响所有调用方 |
| 为什么不能每次请求新建 `Client` | 连接池属于 `Transport`，新建即失去复用，连接数暴增 |
| `MaxIdleConnsPerHost` 默认只有 2 | 高并发下连接被反复关闭重建，TIME_WAIT 飙升 |
| 为什么要排空再关闭 | 未读完的 body 会让连接无法复用 |

版本相关：Go 1.26 新增 `Transport.NewClientConn`（让 `Transport` 按需建立下层连接，便于自定义拨号与测试替身），同一版本 `Client` 的 cookie 作用域改为按 `Request.Host`，`HTTP2Config.StrictMaxConcurrentRequests` 提供 HTTP/2 并发流上限的严格模式；Go 1.27 起 HTTP/1 的 `Response.Body` 关闭时会自动排空未读内容，`Close` 之后连接即可复用。签名与用法见 <https://pkg.go.dev/net/http#Transport>。

## 8. 安全

### 8.1 CrossOriginProtection（Go 1.25）

Go 1.25 新增 `net/http.CrossOriginProtection`：基于 Fetch metadata 请求头判断请求是否来自跨源，在**不需要 CSRF token** 的前提下拦截跨站请求。它适合 cookie 会话型接口；用 Authorization 头的纯 token 接口本来就不受 CSRF 影响。

| 维度 | 传统 CSRF token | `CrossOriginProtection` |
| --- | --- | --- |
| 改动面 | 模板、表单、AJAX 都要带 token | 服务端包一层即可 |
| 状态 | 需要会话存储或签名校验 | 无状态 |
| 兼容性 | 全部浏览器 | 依赖客户端发送 Fetch metadata 头 |
| 绕过控制 | 白名单逻辑自己写 | 支持基于 origin 与基于 pattern 的 bypass |

零值即可使用；具体的包装方法与 bypass 选项见 <https://pkg.go.dev/net/http#CrossOriginProtection>。它防的是跨站请求伪造，不防 XSS：站点一旦存在 XSS，攻击者可在同源上下文直接发请求，任何 CSRF 防护都失效。

### 8.2 CSRF / XSS / SSRF / 请求走私

| 风险 | 最小防御 |
| --- | --- |
| CSRF | cookie 设 `SameSite`，状态变更接口校验 Origin/Referer，或用 `CrossOriginProtection` |
| XSS | 输出按上下文转义；JSON API 保持默认 HTML 转义；设 `Content-Security-Policy` 与 `X-Content-Type-Options: nosniff` |
| SSRF | 出站 URL 只允许白名单 host 与 scheme，禁止跳到内网；解析后校验目标 IP 不属于私有网段 |
| 请求走私 | 由一层网关规范化请求，不要前后端对 `Content-Length`/`Transfer-Encoding` 判断不一致 |
| 目录穿越 | 上传文件名只取 `filepath.Base`，并校验结果仍在目标目录内 |

### 8.3 ReverseProxy：Director 已废弃

Go 1.26 起 `net/http/httputil.ReverseProxy.Director` 废弃，改用 `Rewrite`。原因是安全性的：`Director` 之后仍会经过 hop-by-hop 头清理流程，恶意客户端可以借这些头把代理自己添加的头删掉；`Rewrite` 在清理之后运行，代理设置的头不会被客户端影响。

```go
proxy := &httputil.ReverseProxy{
	Rewrite: func(pr *httputil.ProxyRequest) {
		pr.SetURL(target)                    // 重写目标
		pr.SetXForwarded()                   // 规范地补充转发头
		pr.Out.Header.Set("X-Proxy", "edge") // 清理之后设置，客户端删不掉
	},
}
```

`ProxyRequest` 的完整方法与迁移细节见 <https://pkg.go.dev/net/http/httputil#ReverseProxy>。

## 9. 完整骨架：用户 API

只用标准库：增强路由 + 中间件 + 统一错误 + 优雅关闭，可直接 `go run`。中间件与 `decodeBody` 复用第 4、5.1 节的实现。

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

type User struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

var users = map[string]User{}

// 先编码到内存再写状态码：编码失败时 headers 还没发出。
func writeJSON(w http.ResponseWriter, status int, v any) {
	buf, err := json.Marshal(v)
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(`{"code":"internal","message":"encoding failed"}`))
		return
	}
	w.WriteHeader(status)
	_, _ = w.Write(buf)
}
func writeError(w http.ResponseWriter, status int, code, msg string) {
	writeJSON(w, status, map[string]string{"code": code, "message": msg})
}

func routes() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /users/{id}", func(w http.ResponseWriter, r *http.Request) {
		var u User
		if err := decodeBody(w, r, &u); err != nil { // 见 5.1 节：限体积 + 严格解码
			writeError(w, http.StatusBadRequest, "invalid_json", err.Error())
			return
		}
		u.ID = r.PathValue("id")
		users[u.ID] = u
		writeJSON(w, http.StatusCreated, u)
	})
	mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
		u, ok := users[r.PathValue("id")]
		if !ok {
			writeError(w, http.StatusNotFound, "not_found", "用户不存在")
			return
		}
		writeJSON(w, http.StatusOK, u)
	})
	// 中间件顺序即第 4 节的顺序：recover 最外层。
	return recoverer(requestID(logger(mux)))
}

func main() {
	srv := &http.Server{
		Addr:              ":8080",
		Handler:           routes(),
		ReadHeaderTimeout: 5 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       90 * time.Second,
	}
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			stop(err)
		}
	}()
	<-ctx.Done()
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		slog.Error("shutdown", "err", err)
	}
}
```

```bash
go run . & curl -i http://127.0.0.1:8080/healthz
curl -i -X POST http://127.0.0.1:8080/users/42 -d '{"name":"alice"}'
```

## 10. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 零值 `http.Server` 上线 | 慢连接长期占用，内存攀升 | 无任何超时 | 显式设置四个超时与 `MaxHeaderBytes` |
| 忽略 `http.ErrServerClosed` | 关闭时打错误日志、退出码异常 | 把正常关闭当失败 | 用 `errors.Is` 排除 |
| 用 `Close()` 代替 `Shutdown(ctx)` | 在途请求全部失败 | `Close` 立即断开 | `Shutdown` 并给 ctx 超时 |
| 无 recover 中间件 | 单个请求打挂整个连接 | 缺兜底 | recover 放最外层并记栈 |
| 先 `Decode` 再限体积 | 内存被大 body 打满 | 顺序反了 | `MaxBytesReader` 放最前 |
| 直接信任上传文件名 | 目录穿越写任意路径 | 文件名不可信 | 取 `filepath.Base` 并校验落盘目录 |
| 忘记 `Content-Type` | 客户端按文本解析 JSON | 未设置头 | 写 body 前统一设置 |
| SSE 不监听 `r.Context().Done()` | 客户端断开后 goroutine 泄漏 | 无退出条件 | 在 select 中监听 ctx |
| 用 `http.DefaultClient` | 偶发长时间挂起 | 无超时且全局共享 | 自建带超时的 `Client` |
| 每次请求新建 `Client` | 连接无法复用，端口耗尽 | 连接池属于 `Transport` | 进程内复用一个 `Client` |
| 未排空就关闭 body | 连接无法复用、泄漏句柄 | 未读内容阻碍复用 | `io.Copy(io.Discard, ...)` 后再 `Close` |
| 出站 URL 用用户输入 | SSRF，可探测内网 | 未做白名单与 IP 校验 | 白名单 host 并禁止跳内网 |
| 仍用 `ReverseProxy.Director` | 代理自加的头可被客户端删除 | Go 1.26 起已废弃 | 改用 `Rewrite` |

## 11. 动手练习

1. 用 `"GET /users/{id}"` 写一组路由，覆盖路径参数、多段通配符、方法不匹配（405）、`{$}` 与兜底 `/`，每条规则配一个 `httptest` 用例。
2. 把第 9 节的骨架补全：加 CORS 与超时中间件，并写出你选择的中间件顺序与理由。
3. 给上传接口加「只允许图片、单文件不超过 8MB」的限制，测试覆盖超限、错误 MIME、文件名带 `../` 三种情况。
4. 实现一个 SSE 端点，用 `curl -N` 观察事件流，再人为断开客户端，确认服务端 goroutine 在 1 秒内退出。
5. 写一个使用 `Rewrite` 的 `ReverseProxy`，用 `httptest` 验证客户端传入 `Connection` 等 hop-by-hop 头时，代理添加的头仍然存在；再给服务加上 `CrossOriginProtection`，验证同源通过、跨源被拦截。

## 12. 自检清单

- [ ] 能说出 Go 1.22 起 `ServeMux` 的 pattern 语法与优先级规则。
- [ ] 知道 Go 1.26 起尾斜杠重定向从 301 变成 307，并能解释差异。
- [ ] 生产 `http.Server` 显式设置了四个超时、`MaxHeaderBytes`，并按需设置 `MaxHeaderValueCount`。
- [ ] 优雅关闭用 `Shutdown(ctx)`，排除 `http.ErrServerClosed`，并用 `RegisterOnShutdown` 收尾后台任务。
- [ ] 中间件写成 `func(http.Handler) http.Handler`，顺序有明确理由。
- [ ] 每个读 body 的 handler 都先用 `http.MaxBytesReader` 限体积，上传接口校验文件名与 MIME。
- [ ] 响应统一走 `writeJSON`，先编码到内存再写状态码。
- [ ] 流式/SSE 响应监听 `r.Context().Done()`，且 `WriteTimeout` 配置与之一致。
- [ ] 出站请求用自建 `Client` 且有超时，响应体先排空再关闭。
- [ ] 知道 `CrossOriginProtection` 基于 Fetch metadata、不需要 token，也不防 XSS；代理改用 `Rewrite`。

## 13. 延伸阅读

- Go 1.22 Release Notes（`ServeMux` 增强路由）：<https://go.dev/doc/go1.22>
- Go 1.25 Release Notes（`CrossOriginProtection`）：<https://go.dev/doc/go1.25>
- Go 1.26 Release Notes（`ReverseProxy.Rewrite`、尾斜杠 307）：<https://go.dev/doc/go1.26>
- Go 1.27 Release Notes（`MaxHeaderValueCount`、响应体自动排空）：<https://go.dev/doc/go1.27>
- `net/http`：<https://pkg.go.dev/net/http>
- `net/http/httputil`：<https://pkg.go.dev/net/http/httputil>
- `mime/multipart`：<https://pkg.go.dev/mime/multipart>
