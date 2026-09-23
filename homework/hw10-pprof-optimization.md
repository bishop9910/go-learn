# 作业 10：pprof 性能调优与工程化收口

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W10 |
| 发放日期 | 2026-12-07 |
| 交付日期 | 2026-12-13 |
| 预计工时 | 22 小时 |
| 难度 | 挑战 |
| 对应讲义 | L14、L13 |
| 前置作业 | hw09 |

## 1. 背景与目标

对 hw06 的博客服务做一次有数据支撑的性能优化，并产出可复现的优化报告。本作业的重点不是「改得快」，而是「先量化、再定位、后优化、可回归」——每一条优化都必须有基线数据、profile 证据和优化后的对比数据。

| 目标 | 可观测结果 |
| --- | --- |
| 基线可复现 | 固定数据集与固定压测命令，任何人可复跑得到同量级数据 |
| 热点可定位 | CPU 与内存 profile 各产出 Top 5 热点及解释 |
| 优化有数据 | 每条优化给出「问题 → 手段 → 数据 → 代价」四段式 |
| 实验有对比 | `GOGC` / `GOMEMLIMIT` 至少 3 组组合的权衡表 |
| 门禁全绿 | `golangci-lint`、`go vet`、`govulncheck`、`gofmt -l .` 无输出 |

## 2. 需求（必须项）

**M1 可复现基线。** 用固定数据集（例如 1 万篇文章、100 个标签）与固定压测方式（`go test -bench`，或外部压测工具——工具名与用法以该工具官方说明为准）测出基线：QPS、P50 / P95 / P99 延迟、内存峰值、GC 次数、goroutine 峰值。数据集生成脚本、压测命令、参数全部写进 README，保证他人可复跑。

**M2 CPU profile。** 接入 `net/http/pprof`（见 <https://pkg.go.dev/net/http/pprof>），采集 30s CPU profile，用 `go tool pprof -http` 查看火焰图。**Go 1.26 起 pprof Web UI 默认火焰图视图，图形视图在 View → Graph 或 `/ui/graph`**；在 1.25 上需手动切到 Graph 或 Flame Graph。列出 Top 5 热点函数，逐个解释「为什么在这里热」。

**M3 内存 profile。** 分别采集 `heap` 的 `inuse_space` 与 `alloc_space`，说明两者差异并各自找出分配热点。用 `GODEBUG=gctrace=1` 记录优化前后的 GC 输出，并逐字段解读（GC 轮次、堆大小、目标堆大小、STW 与并发标记耗时）。

**M4 锁与阻塞 profile。** 开启 block / mutex profile，定位至少 1 处锁竞争或阻塞点（例如全局 `sync.Mutex`、缓存层串行化），给出 profile 证据与修复后的对比数据。

**M5 减少分配（至少 3 处）。** 每处给出「问题 → 手段 → 数据 → 代价」。手段可包括：`strings` 包的 `Builder` 替代重复拼接、`strconv` 替代 `fmt` 做数值转换、预分配切片容量、`sync.Pool` 复用缓冲区、避免 interface 装箱。

**M6 逃逸分析（至少 2 处）。** 用 `go build -gcflags=-m ./...` 找出不必要的堆分配并消除，附前后输出片段与 `allocs/op` 变化。

**M7 GC 调优实验（至少 3 组）。** 组合 `GOGC` 与 `GOMEMLIMIT`（Go 1.19 引入），给出内存 / CPU / 延迟的权衡表，列固定为 `配置 / 内存峰值 / GC 次数 / CPU 使用 / P99 延迟 / 结论`。必须说明版本差异：**Go 1.26 起 Green Tea GC 默认启用**（Go 1.25 上是 `GOEXPERIMENT=greenteagc` 实验，可通过 `GOEXPERIMENT=nogreenteagc` 在 1.26 上关闭）；**Go 1.27 起小于 80 字节的分配走 size-specialized malloc，开销最多降 30%**（`GOEXPERIMENT=nosizespecializedmalloc` 关闭）。

**M8 数据库或缓存层优化（至少 1 处）。** 从索引缺失、N+1 查询、缓存命中率低、连接池配置不当中选至少一项，给出优化前后的查询耗时或命中率数据。

**M9 trace 观测。** 用 `runtime/trace` 或 `go tool trace` 采集一次运行，给出调度延迟、GC 停顿、阻塞事件的观察结论。必须描述并在本地实际使用 `runtime/trace.FlightRecorder`（Go 1.25 新增，内存环形缓冲，配置项为 `FlightRecorderConfig`，用 `WriteTo` 导出最近数秒 trace），用它抓取一次「偶发慢请求」的最近数秒 trace。

**M10 工程化收口。** `golangci-lint` v2（配置文件以 `version: "2"` 开头）、`go vet ./...`、`govulncheck ./...`、`gofmt -l .` 全部无输出。用 **Go 1.26 重写的 `go fix`**（modernizers，与 `go vet` 共用同一套 analysis 框架）扫一遍代码库并记录它的改动；若本地仍是 Go 1.25，则在交付说明中写明「1.26+ 才可用，列为升级后待办」。同时说明 Go 1.27 新增的 modernizers `atomictypes`、`embedlit`、`slicesbackward`、`unsafefuncs`，以及 `waitgroup` 分析器改名为 `waitgroupgo`。

## 3. 需求（加分项）

- **A1 PGO。** 用真实压测产生的 CPU profile 作为 PGO 输入（Go 1.21 起正式支持），给出开启前后的 benchmark 对比，并说明 profile 需要随代码更新而重采。
- **A2 FlightRecorder 常驻。** 把 `runtime/trace.FlightRecorder` 做成常驻组件，在慢请求阈值触发时自动导出 trace 并落盘，写清磁盘占用上限与轮转策略。
- **A3 goroutine 创建速率观测。** 用 `runtime/metrics` 的新指标（Go 1.26 新增 `/sched/goroutines*`、`/sched/goroutines-created:goroutines`）建立 goroutine 创建速率的长期观测，并给出优化前后的对比。
- **A4 分配回归门禁。** 把 hw09 的 `testing.AllocsPerRun` 分配回归测试接入 CI（注意 Go 1.25 起在有并行测试运行时它会 panic，需串行执行）。

## 4. 技术约束

| 约束 | 说明 |
| --- | --- |
| 语言基线 | 本仓库基线 Go 1.25；使用 1.26 / 1.27 能力必须标注版本，并说明未升级时的替代做法 |
| 依赖 | 优化不得引入新运行时依赖；压测与 lint 工具用 `go tool` 或独立安装，不进 `go.mod` 的运行时依赖 |
| 功能不变 | 优化不得改变 HTTP 契约与业务语义；优化后必须回归 hw09 的全部测试 |
| 数据纪律 | 任何性能结论必须附原始输出或截图说明；不得只给结论数字 |
| 测量纪律 | 对比数据必须同机、同数据集、同参数、同 `-count`；至少 `-count=10` |
| pprof 暴露 | `net/http/pprof` 只能挂在调试端口，不得暴露在业务端口上 |

## 5. 交付物清单

| 编号 | 交付物 | 形式 |
| --- | --- | --- |
| D1 | 优化报告 | `docs/hw10-optimization-report.md`，含前后数据表与每处优化的四段式 |
| D2 | 火焰图与 profile 证据 | `docs/hw10-profiles/`（截图或 `pprof` 文本输出） |
| D3 | 可复现压测脚本 | `scripts/bench.ps1` 与 `scripts/bench.sh`，参数写入 README |
| D4 | 数据集生成脚本 | `scripts/gen-dataset.go` 或等价实现 |
| D5 | 优化后的代码 | `internal/...`、`cmd/...` |
| D6 | GC 权衡表 | D1 内独立章节，至少 3 组配置 |
| D7 | lint / vet / vulncheck 报告 | `docs/hw10-gates.md`，含命令与原始输出 |
| D8 | `go fix` 改动记录 | `docs/hw10-go-fix.md`，含版本说明 |

## 6. 验收标准（可执行命令）

```bash
go test -bench . -benchmem -count=10 ./... | tee bench-after.txt
go tool pprof -http=:8080 cpu.prof
golangci-lint run
govulncheck ./...
gofmt -l .
go vet ./...
```

| 命令 | 通过条件 |
| --- | --- |
| `-bench . -benchmem -count=10` | 全部通过；输出可用于 benchstat 与报告中的对比 |
| `go tool pprof -http=:8080 cpu.prof` | 能打开 Web UI 并看到火焰图；报告中给出 Top 5 热点 |
| `golangci-lint run` | 无输出、退出码 0 |
| `govulncheck ./...` | 无已知漏洞可达性报告 |
| `gofmt -l .` | 无文件列出 |
| `go vet ./...` | 无输出 |
| 功能回归 | hw09 全部测试在优化后仍通过 |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 基线可复现性 | 15 | 数据集与压测命令完整入 README；前后数据同参数同 `-count`；含 P50/P95/P99 |
| CPU + 内存 profile | 25 | 两者都有采集与解读；Top 5 热点解释到位；`inuse_space` 与 `alloc_space` 区分清楚 |
| 优化实施与数据 | 30 | 分配优化 ≥3、逃逸消除 ≥2、DB/缓存 ≥1；每条四段式齐全；有代价说明 |
| GC 与 trace 实验 | 15 | ≥3 组 `GOGC`/`GOMEMLIMIT` 权衡表；trace 结论具体；FlightRecorder 实际使用 |
| 工程化门禁 | 15 | 四类门禁全绿；`go fix` 改动有记录；版本差异标注准确 |

## 8. 提示与思路

- 优化顺序固定：先补基线 → 再看 CPU profile 找热点 → 再看内存 profile 找分配 → 最后调 GC 参数。跳过基线直接调参数，等于没有优化。
- 火焰图读法：宽而扁的栈是「调用次数多」，窄而深的栈是「单次开销大」。先处理宽度占前 3 的叶子函数。
- `inuse_space` 反映当前存活对象，`alloc_space` 反映累计分配量。降低 `alloc_space` 通常直接减少 GC 压力，降低 `inuse_space` 影响内存峰值。
- `GOMEMLIMIT` 是软限制而非硬上限，设得过低会让 GC 疯狂运行，表现为 CPU 飙升而内存不降。
- `sync.Pool` 的语义是「复用临时对象」，池内对象随时可能被回收，不能当缓存用。
- 微基准与真实负载会有差距：benchmark 用于快速迭代，最终结论要在端到端压测里复核。
- `runtime/trace.FlightRecorder` 的价值在于「事后取证」：慢请求已经发生，但常规 profile 抓不到现场。
- `go fix` 的 modernizers 会改源码，务必在干净工作区运行，改动单独提交，便于逐条审查。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 没有基线就优化 | 无法证明优化有效，甚至变慢 | 凭直觉改代码 | 先建立可复现基线，再动代码 |
| 微基准脱离真实负载 | benchmark 提升但线上延迟不变 | 微基准只覆盖单个函数，忽略了 I/O 与锁 | 微基准用于迭代，端到端压测用于定论 |
| 把 `allocs/op` 当唯一指标 | 分配降了但延迟没降 | 分配只是中间指标 | 同时看 `ns/op`、P95/P99 与 GC 次数 |
| `sync.Pool` 当缓存 | 命中率随机、数据时有时无 | 池对象随时可能被回收 | 池只用于复用临时缓冲区，不做缓存语义 |
| `GOMEMLIMIT` 设得过低 | CPU 飙升、吞吐下降 | GC 被强制高频运行 | 设置为内存上限的合理比例，并配合 `GOGC` 一起实验 |
| 只看平均值不看 P99 | 平均延迟很好但用户仍报慢 | 长尾被平均值掩盖 | 报告必须给出 P50/P95/P99 |
| 优化后未回归功能测试 | 性能上去了、功能坏了 | 只跑 benchmark 没跑测试 | 优化后先跑 hw09 全量测试与 `-race` |
| 在业务端口暴露 pprof | 调试接口可被外部访问 | 图省事直接注册到主 mux | pprof 只挂调试端口，或仅本地监听 |

## 10. 参考实现要点

只给关键设计决策与片段，不给完整成品代码。

```go
// 决策 1：profile 采集入口与调试端口隔离
// 业务 mux 不注册 pprof；另起一个仅监听回环地址的调试 mux，
// 通过 net/http/pprof 的默认注册路径提供 CPU / heap / block / mutex profile。
// 采集命令：go tool pprof -http=:8080 cpu.prof
// Go 1.26+：pprof Web UI 默认火焰图；图形视图在 View -> Graph 或 /ui/graph
```

```bash
# 决策 2：可复现的测量命令（写入 README）
go test -bench . -benchmem -count=10 ./... | tee bench-after.txt
benchstat bench-before.txt bench-after.txt

# 决策 3：GC 实验（至少 3 组）
GOGC=100 GOMEMLIMIT=512MiB go test -bench . -benchmem -count=10 ./...
# 注意：Go 1.26 起 Green Tea GC 默认启用；Go 1.25 需 GOEXPERIMENT=greenteagc
```

```go
// 决策 4：FlightRecorder 抓取偶发慢请求（Go 1.25+）
// 启动时用 FlightRecorderConfig 创建内存环形缓冲并 Start；
// 慢请求阈值触发时调用 WriteTo 导出「最近数秒」的 trace 到文件，
// 再用 go tool trace 打开分析调度延迟、GC 停顿与阻塞事件。
```

```go
// 决策 5：减少分配的常见手法（逐条记录前后 allocs/op）
var b strings.Builder
b.Grow(estimate) // 预分配，避免多次扩容

// strconv 替代 fmt 做数值转换
n, err := strconv.Atoi(s)

// sync.Pool 复用缓冲区：取用、重置、归还，语义是临时对象而非缓存
```

```yaml
# 决策 6：golangci-lint v2 配置骨架
version: "2"
linters:
  enable:
    - govet
    - staticcheck
```

- 优化报告建议模板化：每处优化一个小节，固定四个子标题「问题 / 手段 / 数据 / 代价」。
- 火焰图证据建议同时保留 `pprof` 文本输出（`top`、`list`）与截图：文本便于评审核对函数名，截图便于看调用关系。
- `gctrace` 输出建议粘贴优化前后各一段完整循环，并逐字段标注含义。
- `go fix` 记录建议给出「改动前 / 改动后 / 对应 modernizer 名」三列。
