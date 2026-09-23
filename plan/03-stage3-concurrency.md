# 阶段三：并发与工程化（W7–W10）

- 时间：**2026-11-16（周一）至 2026-12-13（周日）**，共 4 周
- 对应原始规划：阶段 3「并发与工程化」，建议周期 4–6 周，本计划取 4 周
- 作业：hw07（11-22）、hw08（11-29）、hw09（12-06）、hw10（12-13）
- 阶段门：G3

---

## 1. 这一阶段要解决什么

Go 后端能不能写扎实，分水岭就在这一段。目标是把三件事变成硬能力：

| 目标 | 判定标准 |
| --- | --- |
| 并发生命周期可控 | 每个 goroutine 都有明确的退出路径；`-race` 全绿；取消能传导到底；不靠 `time.Sleep` 做同步 |
| 测试能挡住回归 | 表格驱动 + 集成测试 + fuzz + benchmark 四件套齐全；覆盖率有解释而不只是数字 |
| 性能问题能被定位 | 能用 pprof 指出热点，用 trace 看出调度与阻塞，用逃逸分析解释分配，用数据证明优化有效 |

这一段的核心产物是**证据**：不是「我觉得快了」，而是「火焰图显示热点在 X，改法 Y，P99 从 A 降到 B，代价是 C」。

---

## 2. W7：并发原语与并发模式（2026-11-16 ~ 11-22）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 11-16 | 一 | goroutine 与 channel | goroutine 栈与成本；无缓冲 vs 有缓冲的同步语义；`close` 的规则与谁负责关闭；`nil` channel 的阻塞特性 | 一组最小复现程序 |
| 11-17 | 二 | `select` 与定时器 | 多路复用、默认分支忙等陷阱、`ctx.Done()` 分支；热循环里 `time.After` 的内存问题与 `time.NewTimer`/`Reset` 替代；**Go 1.23 起 timer channel 无缓冲**（Go 1.27 永久移除 `asynctimerchan` 开关） | 超时与取消的写法对照 |
| 11-18 | 三 | `sync` 原语 | `Mutex`/`RWMutex` 取舍（RWMutex 不一定更快）、`Once` 与 `OnceFunc`/`OnceValue`、`Cond`、`Pool`；**Go 1.25 的 `sync.WaitGroup.Go`** 与旧写法对比 | 原语清单 + 用法示例 |
| 11-19 | 四 | Worker Pool 与 Fan-in/Fan-out | 「`close(jobs)` + `WaitGroup` + `close(results)`」三段式；谁关闭 channel 的三种设计；用 `runtime.NumGoroutine` 观察峰值 | Worker Pool 实现 |
| 11-20 | 五 | Pipeline 与背压 | 阶段化处理与短路取消；`golang.org/x/sync/errgroup`；缓冲 channel 当队列的局限、信号量模式 | Pipeline 实现 |
| 11-21 | 六 | 作业主体 | 把 hw02 的 URL 检查器重构成 `concur`，补取消/超时/泄漏断言测试 | hw07 主体完成 |
| 11-22 | 日 | 收口 | benchmark 不同并发度；goroutine 峰值对比表；自评打分 | **hw07 交付** |

配套讲义：[L11 并发原语与 context](../lectures/11-concurrency-and-context.md)、[L12 并发模式与并发安全](../lectures/12-concurrency-patterns.md)

必做的一个实验：写两版代码——无界创建 goroutine vs 有界并发——用 `runtime.NumGoroutine()` 与内存占用对比，记录数据。这个实验能让你在以后每次想「先 `go` 出去再说」时停一下。

---

## 3. W8：context 治理与并发安全（2026-11-23 ~ 11-29）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 11-23 | 一 | context 纪律审计 | 逐处检查：是否第一个参数、是否存进 struct、库函数里是否用 `Background()`、取消是否被检查 | context 审计表 |
| 11-24 | 二 | 超时预算分层 | 入口 → service → repository → Redis/MySQL 逐层递减；用慢依赖验证超时能正确中止并返回 504/503 语义 | 超时预算表 + 验证 |
| 11-25 | 三 | 数据竞争定位 | `go test -race -count=1` 与压测找出至少 2 处真实竞争；理解「`-race` 干净 ≠ 没有逻辑竞态」 | 竞争清单 + 复现用例 |
| 11-26 | 四 | 并发安全手段 | mutex / 原子类型 / `sync.Once` 三种修复路径的取舍；`atomic.Pointer[T]` 做配置热更新（copy-on-write） | 修复记录 + 热更新实现 |
| 11-27 | 五 | 泄漏治理 | 三种泄漏形态（发送无人收 / 接收无人发 / 忘记取消）；写泄漏回归测试；**Go 1.26 实验 `GOEXPERIMENT=goroutineleakprofile` 与 Go 1.27 转正的 `goroutineleak` profile（`/debug/pprof/goroutineleak`）** | 泄漏回归测试 |
| 11-28 | 六 | 作业主体 | 把 hw05/hw06 的服务与 `concur` 一起重构 | hw08 主体完成 |
| 11-29 | 日 | 收口 | 压测前后对比；`cancel()` 泄漏自查；自评打分 | **hw08 交付** |

配套讲义：[L11](../lectures/11-concurrency-and-context.md)、[L12](../lectures/12-concurrency-patterns.md)

必答的问题：**你现在手上这个服务，正在跑多少个 goroutine？它们分别在等什么？** 答不出来说明还没做这一周。

---

## 4. W9：测试工程（2026-11-30 ~ 12-06）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 11-30 | 一 | 表格驱动与测试结构 | `t.Run` 子测试、`t.Helper()`、`t.Cleanup()`、`t.TempDir()`、`t.Parallel()` 陷阱；单元测试与集成测试的目录与 `-short` 分层 | 三个包的测试重构 |
| 12-01 | 二 | httptest 与测试替身 | `NewRecorder` 测中间件链、`NewServer` 测客户端；手写 fake vs mock 框架的取舍（`go.uber.org/mock` 只给模块路径） | handler 层测试 |
| 12-02 | 三 | benchmark 与 benchstat | `b.ReportAllocs()`、`-benchmem`、`-count=10`；`benchstat` 的 `p-value`/`geomean` 读法；**Go 1.26 起 `b.Loop()` 不再阻止循环体内联** | 3 个 benchmark + 对比表 |
| 12-03 | 四 | 模糊测试 | `Fuzz` 目标、种子语料、`-fuzztime`；把发现的崩溃固化为 `testdata/fuzz/` 回归用例；说明 CI 只跑种子用例 | 2 个 fuzz 目标 + 语料 |
| 12-04 | 五 | 虚拟时间测试 | **`testing/synctest`（Go 1.25 转正）** 的 `Test`/`Wait` 测试带 `time.Sleep` 的超时逻辑；**Go 1.27 新增 `synctest.Sleep`** | 无需真实等待的并发测试 |
| 12-05 | 六 | 作业主体 | 补覆盖率报告与「未覆盖分支的说明」 | hw09 主体完成 |
| 12-06 | 日 | 收口 | `go test -race -count=1 ./...` 全绿；自评打分 | **hw09 交付** |

配套讲义：[L06 测试工程](../lectures/06-testing.md)

一个硬性要求：**每个 fuzz 或 benchmark 发现的真实问题都要固化成回归测试**。否则这些工作只是表演。

---

## 5. W10：性能分析与工程化收口（2026-12-07 ~ 12-13）

| 日期 | 星期 | 主题 | 具体任务 | 产出 |
| --- | --- | --- | --- | --- |
| 12-07 | 一 | 基线与 CPU profile | 固定数据集与压测方式测出基线（QPS、P50/P95/P99、内存峰值、GC 次数、goroutine 峰值）；`net/http/pprof` 采 30s CPU profile；**Go 1.26 起 pprof Web UI 默认火焰图，图视图在 View → Graph 或 `/ui/graph`** | 基线数据 + 火焰图 |
| 12-08 | 二 | 内存 profile 与 GC | `inuse_space` vs `alloc_space` 的选法；分配热点定位；`GODEBUG=gctrace=1` 输出逐字段解读 | 分配热点清单 |
| 12-09 | 三 | 逃逸分析与降分配 | `go build -gcflags=-m` 读法；消除至少 2 处不必要的堆分配；`strings.Builder`、`strconv` 替代 `fmt`、预分配容量、`sync.Pool` | 降分配前后对比 |
| 12-10 | 四 | trace 与偶发慢请求 | `go tool trace` 看调度延迟、GC 停顿、阻塞事件；**Go 1.25 的 `runtime/trace.FlightRecorder`** 抓取最近数秒 trace；**Go 1.27 起 `go tool trace -http=:6060` 只监听 localhost** | trace 分析结论 |
| 12-11 | 五 | 锁竞争与质量门禁 | block / mutex profile（`SetBlockProfileRate`/`SetMutexProfileRate`）；**Go 1.25 起 mutex profile 中运行时内部锁的竞争点指向临界区末尾**；`golangci-lint` v2 / `govulncheck` / `gofmt` 清零 | 竞争点修复 + 门禁全绿 |
| 12-12 | 六 | 作业主体与报告 | 按「问题 → 手段 → 数据 → 代价」四段式写优化报告；补可复现压测脚本 | hw10 主体完成 |
| 12-13 | 日 | 收口与阶段门 | 按 G3 清单自评；写阶段复盘 | **hw10 交付 + G3 通过** |

配套讲义：[L14 性能分析](../lectures/14-performance.md)、[L13 工程化](../lectures/13-engineering.md)

---

## 6. 本阶段的版本注意事项

| 特性 | 版本 | 影响 |
| --- | --- | --- |
| `sync.WaitGroup.Go` | 1.25 | 简化 goroutine 计数写法；本阶段所有 Worker Pool 至少一条路径用它 |
| `testing/synctest` 转正 | 1.25 | 并发与超时测试不再依赖真实时间 |
| `go vet` 的 `waitgroup` 分析器 | 1.25 | 能自动抓出「在 goroutine 内调用 `WaitGroup.Add`」这类错误；1.27 起改名 `waitgroupgo` |
| `runtime/trace.FlightRecorder` | 1.25 | 生产环境偶发问题的低成本 trace 采集手段 |
| mutex profile 语义变化 | 1.25 | 运行时内部锁的竞争点指向临界区末尾，与 `sync.Mutex` 行为一致 |
| `b.Loop()` 不再阻止内联 | 1.26 | 可以放心把所有 `b.N` 风格基准改写成 `b.Loop()` |
| Green Tea GC 默认启用 | 1.26 | 小对象密集程序的 GC 开销下降，但你自己的性能基线要重新测 |
| `go test` 默认跑 `stdversion` | 1.27 | CI 中可能出现「使用了超出该文件 Go 版本的标准库符号」的报错 |
| `goroutineleak` profile 转正 | 1.27 | 泄漏排查从「看 goroutine 数」升级为「运行时直接报泄漏」 |

---

## 7. 阶段验收清单（G3）

- [ ] 能说出当前服务在跑多少 goroutine、分别阻塞在什么原语上，并有办法验证
- [ ] 每个创建 goroutine 的地方都有明确退出路径，取消后 `runtime.NumGoroutine()` 能回落（有测试）
- [ ] context 审计表完成，超时预算逐层递减，慢依赖能触发正确的错误语义
- [ ] 至少修复 2 处 `-race` 报出的真实竞争，并有记录说明为什么选该手段
- [ ] 有一份可直接复现的基线压测脚本与基线数据（含 P99，不只是平均值）
- [ ] 能用 pprof 指出至少 1 个真实 CPU 热点与 1 个分配热点，并给出优化前后数据
- [ ] 至少 3 个优化按「问题 → 手段 → 数据 → 代价」四段式写清楚
- [ ] fuzz 目标至少 2 个，语料已提交，发现的崩溃已固化为回归测试
- [ ] benchmark 至少 3 个，`benchstat` 对比表含 `p-value`
- [ ] `golangci-lint run`、`go vet ./...`、`go test -race ./...`、`govulncheck ./...`、`gofmt -l .` 全部干净
- [ ] hw07–hw10 自评均 ≥70 分

---

## 8. 资源

官方：

- The Go Memory Model：<https://go.dev/ref/mem>
- Go Concurrency Patterns（Rob Pike 演讲）：<https://go.dev/talks/>
- `runtime/pprof` 文档：<https://pkg.go.dev/runtime/pprof>
- `runtime/trace` 文档：<https://pkg.go.dev/runtime/trace>
- Go 官方博客（性能与运行时相关文章）：<https://go.dev/blog/>
- GC 指南：<https://go.dev/doc/gc-guide>

书籍：

- 《Concurrency in Go》——Katherine Cox-Buday。并发模式与反模式，本阶段的主要参考。
- 《Go 语言高级编程》——柴树杉、曹春晖。并发与性能章节，中文。
- 《Go 语言设计与实现》——左书祺。本阶段只读并发原语与调度相关章节，为阶段五预热。

---

## 9. 本阶段的典型风险

| 风险 | 表现 | 对策 |
| --- | --- | --- |
| 并发炫技 | 到处 `go func()`，没人知道什么时候结束 | 每次引入并发前先回答「谁取消、谁等待、谁收集错误」 |
| 测试为覆盖率服务 | 覆盖率 80% 但关键分支没测 | 覆盖率报告必须附「未覆盖分支及理由」 |
| 优化没有基线 | 说「快了很多」但拿不出数据 | 先写基线脚本，再动手 |
| 只看平均延迟 | P99 已经很糟但平均值好看 | 所有性能数据必须含 P95/P99 |
| `-race` 只跑一次 | CI 不稳定，偶发失败 | `-count=1` + 固定随机种子 + 压测场景也跑 `-race` |
| 过早升级 Go 版本 | 升级后行为变化打乱节奏 | 本阶段保持 1.25，升级放在 W13 之后 |
