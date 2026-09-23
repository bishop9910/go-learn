# 作业 08：context 治理与并发安全重构

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W8 |
| 发放日期 | 2026-11-23 |
| 交付日期 | 2026-11-29 |
| 预计工时 | 18 小时 |
| 难度 | 进阶 |
| 对应讲义 | L11、L12 |
| 前置作业 | hw07、hw05 |

## 1. 背景与目标

hw05 / hw06 的博客服务与 hw07 的 `concur` 工具已经能跑通功能，但生命周期不可控：请求路径不看取消信号、超时只在入口设一层、配置热更新靠共享 `map`、后台 goroutine 无法回收。本作业不新增功能，只把「能跑」重构成「生命周期可控、无数据竞争」。

| 目标 | 可观测结果 |
| --- | --- |
| 取消信号全链路传播 | 客户端断开后，服务端在一层调用内停止后续依赖访问 |
| 超时预算分层 | 各层 deadline 严格递减，慢依赖触发 504 / 503 语义 |
| 共享状态无竞争 | `go test -race -count=1 ./...` 零报告 |
| 配置热更新 | 读路径不加锁，写路径 copy-on-write |
| goroutine 可回收 | 取消后 goroutine 数量回落到基线 |

## 2. 需求（必须项）

需求编号即交付项编号，每条独立可验收。

**M1 context 纪律审计表。** 对 hw05 / hw06 服务与 hw07 `concur` 逐处审计，产出表格，列固定为 `位置 / 现状 / 问题 / 改法 / 验证方式`。必须覆盖 5 类检查项，每类至少一条带 `文件:行号` 的结论：`context.Context` 是否作为第一个参数传递；是否被存进 `struct` 字段；库函数内部是否用 `context.Background()` 短路上游取消；取消是否沿调用链向下传播；超时是否分层（入口 / 服务 / 依赖）。

**M2 超时预算表。** 给出逐层递减预算表，列固定为 `层级 / 预算 / 设置方式 / 超时后返回 / 依据`。建议入口 2s、service 1.5s、单次 DB 或缓存 1s，并说明每个数字的来源（P99 延迟、依赖约定或压测数据）。

**M3 慢依赖验证。** 注入慢依赖（可替换的 `time.Sleep` 实现，或 `net/http/httptest` 的慢 handler），证明三件事：上游 deadline 到达时下游调用被实际中止而非等它跑完；HTTP 侧超时返回 504，依赖整体不可用返回 503；服务记录的错误链能区分「超时」与「不可用」。

**M4 取消原因区分。** 用 `context.WithCancelCause`（Go 1.20 引入）与配套的 `context.Cause`，把「客户端断开」「超时」「内部主动取消」映射到三个可区分的哨兵错误；日志必须打印具体取消原因，而不是笼统的 `context canceled`。给出三种原因各自的触发方式与日志样例。

**M5 优雅关闭。** 用 `os/signal.NotifyContext` 接管中断信号，按「停止接收新请求 → 等在途请求到上限 → 释放资源」的顺序退出。必须写清版本差异：**Go 1.26 起 `os/signal.NotifyContext` 用 `CancelCauseFunc` 取消，取消原因携带收到的信号错误**；Go 1.25 上只有普通 `CancelFunc`，需自行记录信号名。

**M6 数据竞争修复（至少 2 处真实竞争）。** 可用现有代码，也可自行引入（共享 `map`、惰性初始化、计数器）。要求：用 `go test -race -count=1 ./...` 与并发压测复现，附 race 报告原始片段；每处写明「触发条件 / 报告关键行 / 修复手段 / 为什么选它」；三种手段各至少用一次——`sync.Mutex`、`sync/atomic` 的 `atomic.Int64`、`sync.Once` / `sync.OnceValue`。

**M7 并发安全的配置热更新。** 用 `atomic.Pointer[T]` 实现 copy-on-write 配置替换：写路径构造全新不可变配置后原子换指针，读路径只做一次原子读、不加锁。写并发测试：多个读 goroutine 持续读字段，同时多次写入替换，`-race` 下无报告，并断言每次读到的配置都是某个完整版本（不会由两个版本拼成）。

**M8 goroutine 泄漏治理。** 写泄漏回归测试：记录基线 goroutine 数 → 启动并取消被测逻辑 → 等待退出 → 再次采样并断言回落。采样用 `runtime/pprof` 的 `goroutine` profile 导出后计数（Go 1.26+ 也可读 `runtime/metrics` 的 `/sched/goroutines:goroutines`）。同时说明泄漏 profile 的版本与用途：**Go 1.26 提供实验性 `GOEXPERIMENT=goroutineleakprofile`，暴露 `/debug/pprof/goroutineleak`；Go 1.27 转正为 `runtime/pprof` 的 `goroutineleak` profile，路径相同**。未启用实验时以普通 `goroutine` profile 兜底。

## 3. 需求（加分项）

- **A1 取消回调清理链。** 用 `context.AfterFunc`（Go 1.21 引入）把「取消即释放资源」写成可组合回调，避免依赖 `defer` 顺序，并测试回调只执行一次。
- **A2 并发工具收敛。** 用 `sync.WaitGroup.Go`（Go 1.25 新增）替换 hw07 `concur` 里手写的 `Add` + `go` + `Done`，并说明 `go vet` 的 `waitgroup` 分析器（Go 1.25 新增）能发现哪些 `Add` 位置错误。
- **A3 泄漏 profile 实际接入。** 在 Go 1.26+ 环境启用 `GOEXPERIMENT=goroutineleakprofile`，跑泄漏回归并对比开启前后的 profile 输出。
- **A4 逃逸观察。** 用 `go build -gcflags=-m ./...` 定位至少 2 处因接口装箱或闭包捕获导致的堆分配并消除，附前后输出。

## 4. 技术约束

| 约束 | 说明 |
| --- | --- |
| 语言基线 | 本仓库基线 Go 1.25；标注「Go 1.26+」「Go 1.27+」的用法必须写清版本，且在 1.25 上有降级路径 |
| 依赖 | 不新增第三方依赖，标准库优先 |
| 接口兼容 | 公开 HTTP 路由与响应字段不得变更，重构对调用方透明 |
| 并发原语 | 禁止用 `time.Sleep` 轮询充当同步；禁止持锁期间做 I/O |
| 测试 | 新增测试必须能在 `-race` 下通过，不得用跳过标记或关闭 race 绕过 |
| 提交 | 每处修复单独提交，提交信息写明竞争位置与修复手段 |

## 5. 交付物清单

| 编号 | 交付物 | 形式 |
| --- | --- | --- |
| D1 | context 纪律审计表 | `docs/hw08-context-audit.md` |
| D2 | 超时预算表 | 同上文件独立章节 |
| D3 | 重构后的代码 | `internal/...`、`cmd/...` |
| D4 | race 修复记录 | `docs/hw08-race-fixes.md`，含原始报告片段 |
| D5 | 泄漏回归测试 | `internal/.../leak_test.go` |
| D6 | 压测前后对比数据 | `docs/hw08-bench-before-after.md`，含命令与原始输出 |
| D7 | README 更新 | 「如何验证并发安全」一节 |

## 6. 验收标准（可执行命令）

```bash
go build ./...
go vet ./...
go test -race -count=1 ./...
go test -run TestNoGoroutineLeak -count=10 ./...
go build -gcflags=-m ./... 2>&1 | tee escape.txt
```

| 命令 | 通过条件 |
| --- | --- |
| `go vet ./...` | 无输出、退出码 0 |
| `go test -race -count=1 ./...` | 全部通过，输出无 `DATA RACE` |
| 泄漏测试 | `-count=10` 全通过，goroutine 数不随轮次单调上升 |
| `go build -gcflags=-m ./...` | 产出逃逸报告，且能指出至少 2 处已消除的堆分配 |
| 压测对比 | 前后两组数据使用同一命令、同一数据集、同一 `-count` |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| context 审计完整性 | 25 | 5 类检查项全覆盖；每条有位置、改法、验证方式；无无证据的「已符合」结论 |
| 超时与取消正确性 | 25 | 预算逐层递减且可解释；慢依赖被实际中止；504/503 语义正确；取消原因可区分 |
| 竞争修复 | 25 | ≥2 处真实竞争；三种手段各用到；有原始 race 报告；有选型理由 |
| 泄漏治理 | 15 | 回归测试稳定；基线采样合理；profile 版本说明准确 |
| 文档 | 10 | 表格完整、命令可复现、结论有原始输出支撑 |

## 8. 提示与思路

- 先审计再改代码。审计表的「验证方式」必须写成可执行动作（跑哪个测试、看哪个日志字段），否则无法判定完成。
- 超时预算从上往下算：入口预算 = 服务预算 + 序列化与网络开销；服务预算 ≥ 各依赖调用预算的最坏组合 + 余量。不要每层设成同一个数，否则最内层永远不会先超时。
- 找竞争的高效路径：先跑 `go test -race -count=1 ./...`，再用并发压测放大窗口。共享 `map` 与惰性初始化通常一次就能复现。
- `sync.Once` 适合「只初始化一次」，需要读取初始化结果时用 `sync.OnceValue` / `sync.OnceValues` 更直接。
- 泄漏回归测试要留调度余量：取消后等待一小段时间再采样，断言「不超过基线 + 小常数」而非严格相等。
- 把「客户端断开」与「服务端超时」分开统计，否则错误率指标会被主动断开的客户端污染。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `context.WithTimeout` 后忘记 `cancel()` | 定时器与子 context 长期存活，内存缓慢上涨 | 只依赖超时自动触发 | 紧跟 `defer cancel()`，父已取消时也要调 |
| 把 `ctx` 存进 `struct` | 取消不生效，或请求 A 的 ctx 泄漏到请求 B | 生命周期被拉长到对象级 | 每次调用显式作为第一个参数传入 |
| `select` 忘记 `ctx.Done()` 分支 | 下游卡住时上游永不返回 | 只等数据通道 | 每个阻塞等待都带 `ctx.Done()` 分支并返回取消错误 |
| `-race` 干净就认为没有并发问题 | 生产仍出现错乱数据 | 检测器只覆盖实际执行到的交错，逻辑竞态（check-then-act）不报 | race 干净只是必要条件，复合操作仍需锁 |
| 用 `atomic` 修复合操作 | 计数丢失或状态机跳变 | 原子读 + 原子写 ≠ 原子读改写 | 读改写用锁，或重新设计为无锁结构 |
| `sync.Once` 内 panic | 后续调用行为与预期不一致 | 函数 panic 后该 `Once` 视为已完成，不再执行 | 不在 `Once` 内做可能 panic 的初始化，或先在外部校验 |
| 在库函数里写 `context.Background()` | 上游取消与超时失效 | 切断了调用链 | 库函数只用传入的 ctx，仅 `main` 与顶层 handler 创建根 ctx |

## 10. 参考实现要点

只给关键设计决策与片段，不给完整成品代码。

```go
// 决策 1：ctx 是第一个参数，不进 struct
func (s *PostService) GetPost(ctx context.Context, id int64) (*Post, error)

// 决策 2：取消原因可区分（Go 1.20+）
var (
	ErrClientGone = errors.New("client gone")
	ErrInternal   = errors.New("internal cancel")
)

ctx, cancel := context.WithCancelCause(parent)
defer cancel(nil) // 正常路径显式收尾
// 客户端断开时调用：cancel(ErrClientGone)
if errors.Is(context.Cause(ctx), ErrClientGone) {
	// 计为客户端断开，不计入服务端错误率
}
```

```go
// 决策 3：优雅关闭（版本差异）
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
defer stop()
// Go 1.26+：NotifyContext 用 CancelCauseFunc 取消，context.Cause(ctx) 携带信号错误
// Go 1.25：只有 CancelFunc，需自行记录收到的信号名
```

```go
// 决策 4：copy-on-write 配置，读路径无锁
type Config struct {
	Rate    int
	Timeout time.Duration
}

var current atomic.Pointer[Config]

func Load() *Config  { return current.Load() } // 无锁读
func Store(c Config) { current.Store(&c) }     // 整体替换，不原地改
```

```go
// 决策 5：三种修复手段的选型边界
var mu sync.Mutex     // 复合操作：读改写、check-then-act
var hits atomic.Int64 // 单一计数：只用原子读写
var load = sync.OnceValue(func() *Heavy { return newHeavy() }) // 只初始化一次
```

- 审计表的「位置」列写到 `文件:行号`，评审可直接跳转核对。
- 超时预算表建议附一张「超时传播时序」的 `text` 图，标出各层 deadline 的相对时刻。
- 泄漏回归测试建议拆成子测试，分别覆盖「显式取消」与「超时取消」两条路径。
