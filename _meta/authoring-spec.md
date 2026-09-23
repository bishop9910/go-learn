# 编写规范与事实基线

本文件是本仓库所有讲义与作业的**唯一事实来源与风格约束**。任何新增内容都必须符合本文件。

- 生成日期：2026-09-22
- 目标读者：已有 Go 基础、以 Go 后端开发为主线的工程师
- 语言版本基线：**Go 1.25**（本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`）
- 计划周期：2026-09-22 起，至 2027-03-28，共 25 个学习周（含开营周 W0）

---

## 一、事实基线（Go 1.25 / 1.26 / 1.27）

> 以下内容取自官方 Release Notes（<https://go.dev/doc/go1.25>、<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>）。
> **只允许使用本节列出的具体 API 名称。** 本节没有列出、但属于标准库的包，只能写包路径并给官方链接，不得编造函数名、签名或版本号。

### Go 1.25（2025-08 发布）

| 类别 | 变更 |
| --- | --- |
| 语言 | 无影响程序的语法变更；语言规范中移除了 core types 概念，改为专门的行文描述 |
| sync | 新增 `sync.WaitGroup.Go` |
| 测试 | `testing/synctest` 转正（`synctest.Test`、`synctest.Wait`）；1.24 的旧实验 API 在 `GOEXPERIMENT=synctest` 下仍存在，1.26 移除 |
| 测试 | `T.Attr` / `B.Attr` / `F.Attr`；`T.Output`（`io.Writer`）；`testing.AllocsPerRun` 在有并行测试运行时 panic |
| JSON | 新增实验包 `encoding/json/v2` 与 `encoding/json/jsontext`，用 `GOEXPERIMENT=jsonv2` 启用；启用后 `encoding/json` 由新实现支撑，并新增若干配置项 |
| 运行时 | 容器感知 `GOMAXPROCS`：Linux 上考虑 cgroup CPU bandwidth limit；所有平台周期性更新。可用 `GODEBUG=containermaxprocs=0` 与 `updatemaxprocs=0` 关闭；新增 `runtime.SetDefaultGOMAXPROCS` |
| 运行时 | 新 GC 实验 `GOEXPERIMENT=greenteagc`（标记与扫描小对象更好，GC 开销降 10%–40%） |
| 运行时 | `runtime/trace.FlightRecorder`（内存环形缓冲，`WriteTo` 导出最近数秒 trace）；`FlightRecorderConfig` |
| 运行时 | 未被处理的 panic 输出改为 `panic: X [recovered, repanicked]`；Linux 上匿名 VMA 标注（`GODEBUG=decoratemappings=0`） |
| 运行时 | `AddCleanup` 调度的清理函数现在并发并行执行；新增 `GODEBUG=checkfinalizers=1` |
| 工具 | `go vet` 新增 `waitgroup`（`WaitGroup.Add` 位置错误）与 `hostport`（`fmt.Sprintf("%s:%d")` 拼地址，IPv6 不适用，建议 `net.JoinHostPort`） |
| 工具 | `go.mod` 新增 `ignore` 指令；新增 `work` 包匹配模式；`go doc -http`；`go version -m -json`；更新 `go` 行时不再自动写 `toolchain` 行 |
| 工具 | 发行版减少预编译工具二进制，非构建类工具由 `go tool` 按需构建运行 |
| 编译器 | 修复 Go 1.21 引入的 nil 检查延迟 bug（现在会正确 panic）；DWARF 5 调试信息；切片的底层数组在更多情况下可栈分配 |
| 标准库 | `log/slog.GroupAttrs`；`slog.Record.Source` |
| 标准库 | `net/http.CrossOriginProtection`（基于 Fetch metadata 的 CSRF 防护，无需 token） |
| 标准库 | `os.Root` 新增方法：`Chmod` `Chown` `Chtimes` `Lchown` `Link` `MkdirAll` `ReadFile` `Readlink` `RemoveAll` `Rename` `Symlink` `WriteFile`；`DirFS` 与 `Root.FS` 实现 `io/fs.ReadLinkFS`；Windows 上 `os.NewFile` 支持异步 I/O 句柄 |
| 标准库 | `reflect.TypeAssert`；`hash.Cloner` / `hash.XOF`；`io/fs.ReadLinkFS`；`mime/multipart.FileContentDisposition`；`crypto.MessageSigner` / `crypto.SignMessage`；`crypto/sha3.SHA3.Clone`；`crypto/tls.Config.GetEncryptedClientHelloKeys`；`mime`、`net`、`unicode` 若干小改动（`unicode.Cn`、`unicode.LC`、`unicode.CategoryAliases`） |

### Go 1.26（2026-02 发布）

| 类别 | 变更 |
| --- | --- |
| 语言 | 内建 `new` 的运算对象可以是表达式（`new(expr)`），用于给可选字段取初值 |
| 语言 | 泛型类型可以在自身类型参数列表中自引用，例如 `type Adder[A Adder[A]] interface { Add(A) A }` |
| 工具 | `go fix` 完全重写，成为 modernizers 的入口（与 `go vet` 共用同一套 analysis 框架），支持 `//go:fix inline` 源码级内联；旧的 fixer 全部移除 |
| 工具 | `go mod init` 默认写较低版本：`1.N.X` 工具链创建 `go 1.(N-1).0`，预发布版创建 `go 1.(N-2).0`；可用 `go get go@version` 调整 |
| 工具 | `cmd/doc` 与 `go tool doc` 删除（用 `go doc`）；`pprof` Web UI 默认火焰图视图 |
| 运行时 | Green Tea GC 默认启用（`GOEXPERIMENT=nogreenteagc` 关闭，预计 1.27 移除该开关） |
| 运行时 | cgo 调用基线开销降低约 30%；64 位平台堆基址随机化（`GOEXPERIMENT=norandomizedheapbase64` 关闭） |
| 运行时 | goroutine 泄漏 profile 实验：`GOEXPERIMENT=goroutineleakprofile`，暴露 `/debug/pprof/goroutineleak` |
| 标准库 | 新增 `crypto/hpke`（RFC 9180） |
| 标准库 | 新增实验 `simd/archsimd`（`GOEXPERIMENT=simd`，仅 amd64）与实验 `runtime/secret`（`GOEXPERIMENT=runtimesecret`，Linux amd64/arm64） |
| 标准库 | `errors.AsType`（泛型版 `As`，类型安全且更快）；`fmt.Errorf("x")` 分配更少 |
| 标准库 | `io.ReadAll` 更快、内存占用约为原来一半；`bytes.Buffer.Peek` |
| 标准库 | `log/slog.NewMultiHandler`；`slog` 的 `MultiHandler` 会调用所有 handler |
| 标准库 | `net/http`：`Transport.NewClientConn`；`HTTP2Config.StrictMaxConcurrentRequests`；`ServeMux` 尾斜杠重定向由 301 改为 307；`Client` 的 cookie 作用域按 `Request.Host` |
| 标准库 | `net/http/httputil.ReverseProxy.Director` 废弃，改用 `Rewrite`（Director 无法阻止客户端通过 hop-by-hop 头删除代理添加的头） |
| 标准库 | `net/url.Parse` 拒绝 host 中出现冒号（`urlstrictcolons=0` 恢复旧行为）；`net/netip.Prefix.Compare`；`net.Dialer` 新增 `DialIP` `DialTCP` `DialUDP` `DialUnix` |
| 标准库 | `os.Process.WithHandle`；`os/signal.NotifyContext` 用 `CancelCauseFunc` 取消；Windows `os.OpenFile` 的 flag 支持 Windows 专有标志 |
| 标准库 | `reflect` 新增迭代器：`Type.Fields` `Type.Methods` `Type.Ins` `Type.Outs` `Value.Fields` `Value.Methods` |
| 标准库 | `runtime/metrics` 新增 `/sched/goroutines*`、`/sched/threads:threads`、`/sched/goroutines-created:goroutines` |
| 标准库 | `testing.T.ArtifactDir`（配合 `go test -artifacts` 与 `-outputdir`）；`B.Loop` 不再阻止循环体内联；`testing/cryptotest.SetGlobalRandom` |
| 标准库 | `database/sql.ConvertAssign`；`database/sql/driver.RowsColumnScanner`；`image/jpeg` 新实现（输出可能变化）；`go/ast.ParseDirective`；`crypto/tls` 默认启用混合后量子密钥交换 `SecP256r1MLKEM768` / `SecP384r1MLKEM1024` |

### Go 1.27（2026-08 发布，当前最新）

| 类别 | 变更 |
| --- | --- |
| 语言 | **支持泛型方法**：方法声明可以自带类型参数（如 `(*Rand).N[Int intType](Int) Int`）。注意：接口的方法不能声明类型参数，接口方法也不能由泛型方法实现 |
| 语言 | 结构体字面量的键可以是任意合法字段选择器，不再限于顶层字段名 |
| 语言 | 函数类型推断推广：泛型函数被赋值给（或转换为）匹配的函数类型时也参与推断 |
| 工具 | `compile` `link` `asm` `cgo` `cover` `pack` 支持响应文件（`@file`）；`go` 命令移除 `bzr` 支持 |
| 工具 | `go test` 默认执行 `stdversion` vet 检查（报告超出该文件生效 Go 版本的标准库符号）；`go test -json` 的 output 行新增 `OutputType` |
| 工具 | `go doc` 支持 `package@version` 与 `-ex`；`go fix` 新增 modernizers `atomictypes` `embedlit` `slicesbackward` `unsafefuncs`，移除 `fmtappendf`，`waitgroup` 分析器改名 `waitgroupgo` |
| 工具 | `go mod tidy` 对 `go 1.27` 及以上模块自动合并重复的 require 块（最多两个：直接依赖与间接依赖） |
| 工具 | `go tool trace -http=:6060` 仅监听 localhost，需要对外须显式写 `-http=0.0.0.0:6060` |
| 运行时 | traceback 头行包含 `runtime/pprof` goroutine 标签（`GODEBUG=tracebacklabels=0` 关闭）；`asynctimerchan` GODEBUG 永久移除，`time` 包的 channel 恒为无缓冲（同步） |
| 运行时 | 小于 80 字节的分配使用 size-specialized malloc，开销最多降 30%（`GOEXPERIMENT=nosizespecializedmalloc` 关闭） |
| 运行时 | goroutine 泄漏 profile 转正：`runtime/pprof` 的 `goroutineleak` profile，以及 `/debug/pprof/goroutineleak` |
| 标准库 | **`encoding/json/v2` 与 `encoding/json/jsontext` 转正**；`encoding/json` 由 v2 实现支撑，行为保持，错误文本可能变化；`GOEXPERIMENT=nojsonv2` 可退回旧实现 |
| 标准库 | v2 默认更严格：拒绝字符串中的非法 UTF-8、拒绝 JSON 对象中的重复键。相对实验期的变化：移除 `format` tag 选项、移除 `unknown` tag 选项、移除 `DiscardUnknownMembers`、移除 `SkipFunc`；`inline` tag 改名为 `embed`；`string` tag 语义与 `MatchCaseInsensitiveNames` 有调整；`jsontext` 的数值 `Token` 访问器改为同时返回错误 |
| 标准库 | 新增 `crypto/mldsa`（FIPS 204 后量子签名）；`crypto/x509` 支持 ML-DSA；`crypto/tls` 支持 ML-DSA 签名（`MLDSA44` / `MLDSA65` / `MLDSA87`） |
| 标准库 | **新增 `uuid` 包（生成与解析 UUID）** —— 仅知其存在与用途，不要编造函数名，需要示例时给 <https://pkg.go.dev/uuid> 链接 |
| 标准库 | 新增实验包 `simd`（可移植 SIMD，`GOEXPERIMENT=simd`）；`simd/archsimd` 继续实验并扩充到 arm64 Neon 与 wasm |
| 标准库 | `strings.CutLast`、`bytes.CutLast`；`math/rand/v2` 的 `Rand` 新增泛型方法 `N` |
| 标准库 | `testing/synctest.Sleep`（等价于 `time.Sleep` + `synctest.Wait`） |
| 标准库 | `net/http`：`Server.MaxHeaderValueCount`（配合 `DefaultMaxHeaderValueCount`）；HTTP/2 服务端支持 RFC 9218 客户端优先级（`Server.DisableClientPriority` 关闭）；HTTP/1 的 `Response.Body` 关闭时自动排空未读内容 |
| 标准库 | `net/http/httptest.NewTestServer`（配套 `testing/synctest` 的内存假网络）；`net/url.URL.Clone`、`net/url.Values.Clone` |
| 标准库 | `database/sql` 与 `database/sql/driver` 的新接口（`RowsColumnScanner`）；`compress/flate` 更快（gzip/zlib/zip/png 输出可能变化）；`unicode` 升级到 Unicode 17 |
| 工具链 | 构建引导要求：1.26 需要 Go 1.24.6+ 引导；预计 1.28 需要 1.26 的某个小版本 |

### 更早版本的关键基线（可放心使用）

| 版本 | 内容 |
| --- | --- |
| 1.18 | 泛型、模糊测试、`go work` 工作区 |
| 1.19 | `sync/atomic` 的 `atomic.Int64` 等类型、`GOMEMLIMIT`、`runtime/debug.SetMemoryLimit` |
| 1.20 | `errors.Join`、`context.WithCancelCause`、`unsafe.String` / `unsafe.StringData` / `unsafe.SliceData`、`bytes.Clone`、PGO 预览 |
| 1.21 | 内建 `min` / `max` / `clear`、`slices` / `maps` / `cmp` 包、`log/slog`、`sync.OnceFunc` / `OnceValue` / `OnceValues`、`context.AfterFunc`、PGO 正式支持 |
| 1.22 | `for range` 支持整数、循环变量每轮独立、`net/http` 增强 `ServeMux` 路由（方法与通配符）、`math/rand/v2` |
| 1.23 | `iter` 包与 range-over-func、`unique`、`structs.HostLayout`、`time.Timer` 同步通道、`slices` / `maps` 的迭代器函数 |
| 1.24 | `os.Root`、`weak`、`runtime.AddCleanup`、泛型类型别名、`go.mod` 的 `tool` 指令（`go get -tool` 与 `go tool`） |

### 本机工具链校验记录（2026-09-22，go1.25.7 windows/amd64）

用 `go doc <符号>` 在本机 `go1.25.7` 上逐条校验过下列符号的**存在性**，作为版本标注的交叉验证：

| 符号 | 期望版本 | 本机 1.25.7 实测 |
| --- | --- | --- |
| `runtime/trace.FlightRecorder`、`runtime/trace.FlightRecorderConfig` | 1.25+ | 存在 |
| `sync.WaitGroup.Go` | 1.25+ | 存在 |
| `log/slog.GroupAttrs`、`log/slog.Record.Source` | 1.25+ | 存在 |
| `os.Root`、`os.Root.ReadFile` | 1.24 引入 / 1.25 补齐方法 | 存在 |
| `testing/synctest.Test`、`testing/synctest.Wait` | 1.25+ | 存在 |
| `reflect.TypeAssert` | 1.25+ | 存在 |
| `net/http.CrossOriginProtection` | 1.25+ | 存在 |
| `io/fs.ReadLinkFS` | 1.25+ | 存在 |
| `errors.AsType` | 1.26+ | 不存在（符合预期） |
| `bytes.Buffer.Peek` | 1.26+ | 不存在（符合预期） |
| `testing.T.ArtifactDir` | 1.26+ | 不存在（符合预期） |
| `strings.CutLast`、`bytes.CutLast` | 1.27+ | 不存在（符合预期） |
| `testing/synctest.Sleep` | 1.27+ | 不存在（符合预期） |
| `uuid`（标准库包） | 1.27+ | 不存在（符合预期） |

> `io.ReadAll` 自早期版本就存在，Go 1.26 变更是**性能与内存改进**而非新增 API——描述它时必须写成「Go 1.26 起更快、更省内存」，不能写成「Go 1.26 新增」。

`runtime/trace.FlightRecorder` 的完整签名（本机 `go doc runtime/trace` 实测，可直接引用）：

```go
func NewFlightRecorder(cfg FlightRecorderConfig) *FlightRecorder
func (fr *FlightRecorder) Enabled() bool
func (fr *FlightRecorder) Start() error
func (fr *FlightRecorder) Stop()
func (fr *FlightRecorder) WriteTo(w io.Writer) (n int64, err error)

type FlightRecorderConfig struct {
	MinAge   time.Duration // 窗口期望保留的最小时间跨度
	MaxBytes uint64        // 窗口字节数上界；优先于 MinAge
}
```

约束：同一时刻**最多只有一个** flight recorder 活动；`MaxBytes` 只是上界提示，不保证 `WriteTo` 的数据量与内存上界。

### 禁止事项

1. 不得出现上表中没有的具体 API 名称、函数签名、常量名、版本号。需要引用标准库但上表未列出时，写成
   `包路径` + 官方链接（例如 `net/http/pprof`，见 <https://pkg.go.dev/net/http/pprof>）。
2. 不得把 Go 1.26/1.27 才有的能力写成 1.25 可用；反之，标注「Go 1.27+」的示例不得声称在 1.25 可编译。
3. 不得编造第三方库的版本号。第三方库只写模块路径（如 `github.com/gin-gonic/gin`），并注明「以官方最新稳定版为准」。
4. 不得给出盗版电子书、网盘、破解资源链接。书籍只写书名、作者、出版社。
5. 不得使用 `GOEXPERIMENT` 之外的杜撰环境变量名。

---

## 二、写作风格

1. 正文一律**简体中文**；代码标识符、文件路径、HTTP 路径、字段名、配置键、库名一律保持英文。
2. GitHub 风格 Markdown。信息能用表格表达就优先用表格，不要写成大段散文。
3. 代码块必须带语言标记：`go`、`bash`、`sql`、`yaml`、`json`、`dockerfile`、`text`、`ini`、`proto`。
4. **禁止 emoji**、禁止营销语言、禁止「众所周知」「简单来说」「不言而喻」一类填充语。事实密集，陈述可溯源。
5. Windows 环境下的命令示例：本机 shell 为 PowerShell，shell 示例默认给 `bash` 与 PowerShell 两种写法差异明显时并列；Go 命令本身跨平台，直接写即可。
6. 文件之间的链接使用同级文件名的相对形式，例如 `[L03 错误处理与结构化日志](03-errors-and-logging.md)`。

## 三、讲义文件结构

```
# L0N 标题

（本篇定位：3–5 行，说明解决什么问题、对应哪几个学习周、前置要求）

## 1. ...
### 1.1 ...
（正文 + 可编译的代码示例 + 版本差异说明）

## N. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |

## N+1. 动手练习

## N+2. 自检清单

- [ ] ...

## N+3. 延伸阅读

- 官方链接（go.dev / pkg.go.dev / go blog）
- 书籍只写书名、作者、出版社
```

篇幅要求：每个讲义文件 **250–450 行**。这是**参考区间，不是上限**：当「必须覆盖」的内容本身由大量代码块与表格构成时（GFM 要求代码块与表格前后留空行，行数会被显著放大），可以写得更长，**不要为了压行数删减必需内容**。宁可长而完整，也不要短而缺项。

## 四、作业文件结构

```
# 作业 NN：标题

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W? |
| 发放日期 | YYYY-MM-DD |
| 交付日期 | YYYY-MM-DD |
| 预计工时 | ? 小时 |
| 难度 | 入门 / 进阶 / 挑战 |
| 对应讲义 | L?? |
| 前置作业 | hw??（无则写「无」） |

## 1. 背景与目标
## 2. 需求（必须项）
（编号的 MUST 列表，每条可验收）
## 3. 需求（加分项）
## 4. 技术约束
## 5. 交付物清单
## 6. 验收标准（可执行命令）
## 7. 评分表
| 维度 | 分值 | 评分要点 |
## 8. 提示与思路
## 9. 常见坑
## 10. 参考实现要点
（只给关键设计决策与关键代码片段，不给完整成品代码）
```

篇幅要求：每个作业文件 **100–180 行**。

### 发放日期与交付日期的口径

- **发放日期 = 该作业「对应周次」区间的第一天。** 单周作业即该周周一；W1 是残周，第一天为 2026-10-08（周四）；跨周作业为该区间第一个周的周一。
- **交付日期 = 该作业「对应周次」区间的最后一天**（通常为周日）。
- 例：对应周次 W8（2026-11-23 ~ 11-29）→ 发放 2026-11-23，交付 2026-11-29。
- 对应周次 W23–W25（2027-03-08 ~ 03-28）→ 发放 2027-03-08，交付 2027-03-28。
- 跨周的大作业（hw17–hw20）在区间的第一个周一时就发放，不提前到上一周；需要提前预习的，在「提示与思路」里说明而不是改发放日期。

---

## 五、日期与假期

- 开营日：**2026-09-22（周二）**
- 国庆假期（用户指定，不安排任务）：**2026-09-25（周五）至 2026-10-07（周三）**，共 13 天
- 复课日：**2026-10-08（周四）**
- 学习周以周一至周日为一周；W1 是残周（10-08 至 10-11，4 天）
- 其他法定假日（元旦 2027-01-01、春节 2027-02-06）默认按「降强度 / 机动」处理，见 `plan/00-overview.md`

周次对照：

| 周 | 起 | 止 | 阶段 |
| --- | --- | --- | --- |
| W0 | 2026-09-22 | 2026-09-24 | 开营 |
| （假期） | 2026-09-25 | 2026-10-07 | — |
| W1 | 2026-10-08 | 2026-10-11 | 阶段一 |
| W2 | 2026-10-12 | 2026-10-18 | 阶段一 |
| W3 | 2026-10-19 | 2026-10-25 | 阶段二 |
| W4 | 2026-10-26 | 2026-11-01 | 阶段二 |
| W5 | 2026-11-02 | 2026-11-08 | 阶段二 |
| W6 | 2026-11-09 | 2026-11-15 | 阶段二 |
| W7 | 2026-11-16 | 2026-11-22 | 阶段三 |
| W8 | 2026-11-23 | 2026-11-29 | 阶段三 |
| W9 | 2026-11-30 | 2026-12-06 | 阶段三 |
| W10 | 2026-12-07 | 2026-12-13 | 阶段三 |
| W11 | 2026-12-14 | 2026-12-20 | 阶段四 |
| W12 | 2026-12-21 | 2026-12-27 | 阶段四 |
| W13 | 2026-12-28 | 2027-01-03 | 阶段四 |
| W14 | 2027-01-04 | 2027-01-10 | 阶段四 |
| W15 | 2027-01-11 | 2027-01-17 | 阶段四 |
| W16 | 2027-01-18 | 2027-01-24 | 阶段四 |
| W17 | 2027-01-25 | 2027-01-31 | 阶段五 |
| W18 | 2027-02-01 | 2027-02-07 | 阶段五（春节弹性） |
| W19 | 2027-02-08 | 2027-02-14 | 阶段五（春节弹性） |
| W20 | 2027-02-15 | 2027-02-21 | 阶段五 |
| W21 | 2027-02-22 | 2027-02-28 | 阶段五 |
| W22 | 2027-03-01 | 2027-03-07 | 阶段五 |
| W23 | 2027-03-08 | 2027-03-14 | 阶段五 |
| W24 | 2027-03-15 | 2027-03-21 | 阶段五 |
| W25 | 2027-03-22 | 2027-03-28 | 作品集收口 |

> 校验：2026-09-21 是周一，2026-09-22 是周二，2026-09-25 是周五，2026-10-07 是周三，2026-10-08 是周四，2027-01-01 是周五，2027-02-06 是周六。
