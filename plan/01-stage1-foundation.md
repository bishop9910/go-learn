# 阶段一：基础补全（W1–W2）

- 时间：**2026-10-08（周四）至 2026-10-18（周日）**，共 11 天（W1 为 4 天残周）
- 对应原始规划：阶段 1「基础补全」，建议周期 2–4 周，本计划取 2 周下限（因为已具备 Go 基础语法与部分标准库经验）
- 作业：hw01（10-11 交付）、hw02（10-18 交付）
- 阶段门：G1

---

## 1. 这一阶段要解决什么

不是重学 `if` / `for`，而是把四件事变成肌肉记忆：

| 目标 | 判定标准 |
| --- | --- |
| 工具链可控 | 能解释 `go env` 里每个关键项；能独立处理模块下载失败；知道 `go.mod` 的 `go` 行与 `toolchain` 行的区别 |
| 工程结构成型 | 能一次写出 `cmd/` + `internal/` 分层且依赖方向正确的项目骨架 |
| 错误处理成体系 | 包装、判定、翻译三层分明；`errors.Is` / `errors.As` 用对；知道 `panic` 的边界 |
| 标准库能用 | `io` / `os` / `strings` / `strconv` / `encoding/json` / `time` / `log/slog` 不查文档也能写出常见用法 |

同时把 **Go 1.25 的新东西**上手，因为后面所有阶段都会用到：`sync.WaitGroup.Go`、`testing/synctest`、`log/slog` 的 `GroupAttrs` 与 `Record.Source`、`net/http.CrossOriginProtection`、`os.Root`、`go vet` 的新分析器。

---

## 2. W1：工具链、模块与错误处理（2026-10-08 ~ 10-11，4 天）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 10-08 | 四 | 工具链与版本策略 | 复跑 `go version` / `go env`；搞清 `GOPROXY`、`GOSUMDB`、`GOFLAGS`、`GOTOOLCHAIN` 的语义；建 `logscan` 模块并跑通 `init`/`tidy`/`build`/`run`/`vet` | 一份环境记录 + 能跑的 `hello` 模块 |
| 10-09 | 五 | 模块与多包结构 | 拆出 `cmd/logscan`、`internal/cli`、`internal/logline`、`internal/version`；理解 `internal` 的编译期边界；`replace` 与 `go work` 的最小实验 | 多包骨架 + 目录说明 |
| 10-10 | 六 | 错误处理与退出码 | 哨兵错误、自定义错误类型、`%w` 包装三层、`errors.Is`/`errors.As` 判定、`main` 统一翻译成退出码 0/1/2 | `logscan` 的错误链路 + 退出码表 |
| 10-11 | 日 | 测试入门与 hw01 收口 | 表格驱动测试、`t.Run`、`t.Helper`、`t.TempDir`、覆盖率报告；`gofmt -l .` 与 `go vet ./...` 清零 | **hw01 交付** |

配套讲义：[L01 工具链与 Go Modules](../lectures/01-toolchain-and-modules.md)、[L02 语言精要与易错点](../lectures/02-language-essentials.md)、[L03 错误处理与结构化日志](../lectures/03-errors-and-logging.md)

W1 的坑：不要一开始就引入 CLI 框架。`flag` 包 + 手写子命令分发虽然笨，但能逼你把参数解析、错误处理、退出码想清楚。

---

## 3. W2：标准库核心与新特性（2026-10-12 ~ 10-18，7 天）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 10-12 | 一 | 字符串与字节处理 | `strings.Builder` vs `+` 的分配差异；`strings.Cut`；`strconv` 替代 `fmt` 的热路径取舍；写一个 benchmark 验证 | 一组前后对比 benchmark |
| 10-13 | 二 | 流式 IO 与文件安全 | `bufio.Scanner` 的 64KB 行上限与 `Scanner.Buffer`；`Scanner.Err()`；`bufio.Writer` 的 Flush 时机；**`os.Root`（Go 1.24 引入、1.25 补齐方法）限制文件访问根目录** | `logscan clean` 的流式实现 + 越权拒绝测试 |
| 10-14 | 三 | JSON 与配置 | v1 的 tag 与 `omitempty` 语义；用指针区分「字段缺失」与「零值」；`Decoder.DisallowUnknownFields`；预告 **Go 1.27 起 `encoding/json` 由 v2 实现支撑** 的行为差异 | `logscan stats` 的配置加载与校验 |
| 10-15 | 四 | 结构化日志与 Go 1.25 新特性 | `log/slog` 的 `TextHandler` / `JSONHandler` / `Level` / `With` / `WithGroup`；**`slog.GroupAttrs` 与 `slog.Record.Source`**；**`sync.WaitGroup.Go`**；**`go vet` 的 `waitgroup` 与 `hostport` 分析器**（Go 1.27 起 `waitgroup` 改名 `waitgroupgo`） | 全项目日志改造 + 一段可以用 vet 抓出问题的反例代码 |
| 10-16 | 五 | HTTP 客户端与超时 | `http.Client` 超时、`Transport` 连接复用、必须读完并关闭 `resp.Body`；`context` 超时；缓冲 channel 做信号量实现有界并发 | `logscan check` 的初版 |
| 10-17 | 六 | 作业主体 | 把 clean / stats / check 三条路径打通，补错误分支与边界；写 `httptest` 集成测试 | hw02 主体完成 |
| 10-18 | 日 | 收口与阶段门 | benchmark + `-race`；README（用法、配置 schema、退出码）；按 G1 清单自评 | **hw02 交付 + G1 通过** |

配套讲义：[L03](../lectures/03-errors-and-logging.md)、[L04 标准库核心](../lectures/04-stdlib-core.md)、[L05 序列化](../lectures/05-serialization-json.md)、[L07 net/http](../lectures/07-net-http.md)（只读客户端与超时部分）

---

## 4. 本阶段必须记住的 Go 1.25+ 差异

| 特性 | 版本 | 说明 |
| --- | --- | --- |
| `sync.WaitGroup.Go` | 1.25 | 创建并计数 goroutine 的惯用写法，替代手写 `wg.Add(1)` + `go func(){ defer wg.Done() ... }()` |
| `testing/synctest` 转正 | 1.25 | 虚拟时钟气泡，测试带 `time.Sleep` 的逻辑无需真实等待 |
| `slog.GroupAttrs` / `slog.Record.Source` | 1.25 | 前者从 `[]Attr` 构造分组属性，后者取记录来源位置 |
| `os.Root` 补齐方法 | 1.25 | `ReadFile` / `WriteFile` / `MkdirAll` / `RemoveAll` / `Rename` / `Symlink` / `Readlink` 等，用于把文件操作限制在目录内 |
| `go vet` 的 `waitgroup` / `hostport` | 1.25 | 前者报 `WaitGroup.Add` 位置错误，后者报用 `fmt.Sprintf("%s:%d")` 拼地址（IPv6 不适用），建议 `net.JoinHostPort` |
| `go.mod` 的 `ignore` 指令 | 1.25 | 让 `./...` 等模式忽略指定目录 |
| `errors.AsType` | 1.26 | 泛型版 `As`，类型安全且更快 |
| `io.ReadAll` 更快更省内存 | 1.26 | 中间分配更少，返回最小容量切片 |
| `new(expr)` | 1.26 | 内建 `new` 可带初值表达式，适合给可选字段取初值 |
| 泛型方法 | 1.27 | 方法可自带类型参数；接口方法不能带类型参数，也不能由泛型方法实现 |
| `encoding/json/v2` 转正 | 1.27 | `encoding/json` 由 v2 支撑；默认拒绝非法 UTF-8 与重复键；`GOEXPERIMENT=nojsonv2` 可退回 |
| `strings.CutLast` / `bytes.CutLast` | 1.27 | 按最后一次出现切分 |
| 标准库 `uuid` 包 | 1.27 | 生成与解析 UUID，见 <https://pkg.go.dev/uuid> |

**注意**：Go 1.25 修复了一个从 1.21 引入的编译器 bug——过去在检查 `err` 之前就使用返回值（例如 `f, err := os.Open(...)` 后先调 `f.Name()`）不会 panic，现在会正确地 panic。如果这类代码是你的习惯，本阶段就要改掉：**错误检查紧跟产生错误的语句**。

---

## 5. 阶段验收清单（G1）

- [ ] `go version` 为本机 1.25.7 或更高，且能解释 `go env` 中 `GOPROXY`、`GOSUMDB`、`GOFLAGS`、`GOTOOLCHAIN` 的作用
- [ ] 能独立从零建立「`cmd/` + `internal/` 多包」模块，且能说明为什么用 `internal`
- [ ] 项目里的错误有三层结构：哨兵/自定义类型 → `%w` 包装 → 边界翻译为退出码或 HTTP 状态码
- [ ] 至少写过 8 个表格驱动测试用例，并能用 `go test -cover` 给出覆盖率
- [ ] `go vet ./...`、`go test -race ./...`、`gofmt -l .`（输出为空）三项全通
- [ ] 会用 `log/slog` 输出 JSON 日志，并能解释 `GroupAttrs` 与 `Record.Source` 的用途
- [ ] 能用 `os.Root` 写出一个「拒绝 `../` 越权访问」的测试
- [ ] 至少写过 2 个 benchmark，能解释 `allocs/op` 与 `ns/op` 的差别
- [ ] hw01、hw02 自评均 ≥70 分

---

## 6. 资源

官方（按优先级）：

- A Tour of Go：<https://go.dev/tour/>
- Effective Go：<https://go.dev/doc/effective_go>
- Go 1.25 Release Notes：<https://go.dev/doc/go1.25>
- 标准库文档：<https://pkg.go.dev/std>
- Go by Example：<https://gobyexample.com/>

书籍（都只给书名，自行通过正规渠道获取）：

- 《Go 程序设计语言》——Alan A. A. Donovan、Brian W. Kernighan。适合作为语言与标准库的权威参照，本阶段读第 1–7 章。
- 《Go 语言实战》——William Kennedy 等。偏工程实践，本阶段读「错误处理」「接口」两章。
- 《Go 语言学习笔记》——雨痕。中文，适合查漏补缺。

不建议本阶段看的：涉及运行时内部实现的书（留到阶段五）。

---

## 7. 本阶段的典型风险

| 风险 | 表现 | 对策 |
| --- | --- | --- |
| 把基础阶段当复习，跳着做 | 直接开始写 Web 项目，错误处理与测试欠账 | 阶段门的 9 条必须逐条打勾，不允许「大概会了」 |
| 引入过多第三方库 | CLI 用 cobra、配置用 viper、日志用 zap | 本阶段只用标准库，目的是把标准库练熟 |
| 只写正常路径 | 测试只覆盖 happy path | 每写一个功能，强制补一个失败用例 |
| 忽略 `gofmt` | 代码风格混乱，后续 lint 大量报错 | 每天收工前跑一次 `gofmt -l .` |
| 代理与工具链问题卡住 | `go mod tidy` 卡在下载 | 配置 `GOPROXY` 并固定 `GOTOOLCHAIN=local`，见 [L01](../lectures/01-toolchain-and-modules.md) |
