# 作业 07：并发模式——Worker Pool、Pipeline 与 Fan-in/Fan-out

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W7 |
| 发放日期 | 2026-11-16 |
| 交付日期 | 2026-11-22 |
| 预计工时 | 18 小时 |
| 难度 | 进阶 |
| 对应讲义 | L11、L12 |
| 前置作业 | hw02（并发 URL 检查器） |

## 1. 背景与目标

把 hw02 的并发 URL 检查器重构成一个正式项目 `concur`：三种并发模式各自独立、可测试，并且满足「有界、可取消、不泄漏、可观测」四条工程要求。完成本作业后你应能：

- 写出「谁生产、谁关闭、谁等待」都唯一确定的 channel 生命周期。
- 用数据说明无界并发的危害，而不是只凭感觉说「会炸」。
- 用可复现的断言证明取消后 goroutine 数量回到基线。
- 把重试、退避、超时、限流组合成不违背 `context` 取消语义的调用链。

## 2. 需求（必须项）

**项目形态**

- **M1** 项目名 `concur`，命令行必须支持：`--concurrency`（最大并发，必须有上限校验）、`--timeout`（全局预算）、`--retry`（最大重试次数）、输入来源（文件或参数）。
- **M2** 三种模式必须能被独立调用与独立测试（各自一个包或一组导出函数 + 构造函数），不得互相耦合。

**模式一：Worker Pool**

- **M3** 有界并发：固定数量的 worker 从任务 channel 取任务，结果经结果 channel 汇总；退出必须遵循三段式：投递方 `close(jobs)` → worker 用 `sync.WaitGroup` 等齐全部退出 → 汇总方 `close(results)`。
- **M4** 至少一条路径使用 Go 1.25 新增的 `sync.WaitGroup.Go`，另一条路径保留「先 `Add` 再起 goroutine」的旧写法；README 用表格对比两种写法的代码行数、审查难点与出错场景（`sync.WaitGroup.Go` 自 Go 1.25 提供，见 <https://go.dev/doc/go1.25>）。

**模式二：Pipeline**

- **M5** 至少三个阶段：抓取 → 解析 → 汇总；每个阶段是独立函数，能用假数据源单独测试。
- **M6** 阶段间用 channel 串联，取消用 `context` 短路；错误传播用 `golang.org/x/sync/errgroup`（以官方最新稳定版为准），README 说明在「一错即停」与「收集全部错误后统一处理」之间选了哪种策略及理由。

**模式三：Fan-in / Fan-out**

- **M7** 多生产者（多个抓取源）+ 单消费者（汇总），用 `select` 同时处理结果、错误与 `context` 取消，不得使用忙等的默认分支。
- **M8** README 必须写明「谁关闭 channel」的规则表；必须有一个用例演示「向已关闭 channel 发送会 panic」，并说明你的设计如何规避（例如只有唯一的发送方负责关闭，消费者从不关闭）。

**有界、取消与背压**

- **M9** 必须给出「无界 vs 有界」对比实验并记录 goroutine 峰值与内存占用，以表格写入 README；数据来源可用 `runtime/metrics` 的 `/sched/goroutines` 系列指标（Go 1.26+，见 <https://pkg.go.dev/runtime/metrics>）或 `runtime` 包的 goroutine 计数接口（见 <https://pkg.go.dev/runtime>）。
- **M10** `--timeout` 是全局预算，单任务另有更短超时；任一预算耗尽都必须触发取消并向上返回明确错误。
- **M11** 必须有一个测试断言「取消后 goroutine 数量回落到基线附近」。断言写法必须稳定：轮询 + 超时上限，或 `testing/synctest`（Go 1.25 转正，见 <https://pkg.go.dev/testing/synctest>）做确定性等待；不得用固定 `time.Sleep` 猜时序。
- **M12** 背压：任务队列使用带缓冲 channel；README 说明缓冲大小的选择依据，以及溢出策略（阻塞 / 丢弃 / 快速失败三选一）对上游的影响。

**重试与可观测**

- **M13** 重试必须区分可重试错误（超时、连接重置、上游 5xx）与不可重试错误（客户端 4xx、解析失败）；退避用指数 + 抖动；有最大次数；重试期间必须尊重 `context` 取消。
- **M14** 用 `log/slog` 输出每阶段的结构化日志，字段至少包含任务 id、阶段、耗时、结果；结束时输出统计：成功数、失败数、重试次数、耗时 P50/P95。

- **M15** `go test -race ./...` 全绿；至少 3 个并发相关测试：取消、超时、worker 内 panic 恢复（panic 不得拖垮进程或泄漏 goroutine）。
- **M16** 至少 2 个 benchmark，对比不同并发度下的吞吐，并在 README 记录结果。

## 3. 需求（加分项）

- **B1** 增加「有界且公平」的变体，保证慢任务不饿死后续任务，并说明调度差别。
- **B2** 用 `runtime/trace.FlightRecorder`（Go 1.25 新增，见 <https://go.dev/doc/go1.25>）在出现长尾时导出最近数秒的 trace，用于事后分析。
- **B3** 把重试与限速（`golang.org/x/time/rate`，以官方最新稳定版为准）组合成「按目标的并发 + 速率」双上限。
- **B4** 用 `testing/synctest` 替换部分轮询式断言，给出确定性与可读性的对比。

## 4. 技术约束

| 编号 | 约束 |
| --- | --- |
| C1 | Go 1.25 工具链；使用 `sync.WaitGroup.Go` 的路径需注明该 API 自 Go 1.25 起提供 |
| C2 | 不得用 `time.Sleep` 做同步；它只能用于退避间隔与轮询间隔 |
| C3 | 每个 goroutine 的创建点都必须能回答「谁等它、谁取消它」 |
| C4 | 禁止向已关闭 channel 发送、禁止重复关闭、禁止消费者关闭生产者的 channel |
| C5 | 所有 channel 的关闭责任唯一且写在代码注释与 README 的规则表里 |
| C6 | 除标准库与作业列出的模块外不引入新依赖 |
| C7 | 需要分析时使用 `runtime/pprof`（见 <https://pkg.go.dev/runtime/pprof>）与 `net/http/pprof`（见 <https://pkg.go.dev/net/http/pprof>），并在 README 说明采集命令 |

## 5. 交付物清单

| 交付物 | 要求 |
| --- | --- |
| 源代码 | `concur` 模块，三种模式各自独立可调用 |
| README | 三种模式说明、并发度调优数据表、goroutine 峰值对比表、channel 关闭责任表、重试策略说明 |
| 测试 | 覆盖取消、超时、panic 恢复，以及「向已关闭 channel 发送」的演示用例 |
| benchmark 报告 | 命令、并发度、吞吐、内存分配数据 |
| 分析产物 | 加分项的 trace 或 profile 文件与解读 |

## 6. 验收标准（可执行命令）

| 命令 / 检查 | 通过标准 |
| --- | --- |
| `go vet ./...` | 无输出。注意 Go 1.25 起 `go vet` 新增 `waitgroup` 分析器（报告在 goroutine 内 `Add` 这类位置错误，见 <https://go.dev/doc/go1.25>）；Go 1.27 起该分析器改名为 `waitgroupgo`（见 <https://go.dev/doc/go1.27>） |
| `go test -race -count=1 ./...` | 全部通过，无数据竞争 |
| `go test -bench . -benchmem ./...` | 输出每个并发度的吞吐与分配数据 |
| `./concur --concurrency=1`、`=8`、`=64` 跑同一输入 | 三次都成功退出；README 记录耗时与 goroutine 峰值 |
| 取消用例 | 断言取消后 goroutine 数量回落到基线，且不依赖固定 sleep |
| panic 用例 | worker 内 panic 被恢复并作为错误上报，进程正常退出 |
| 泄漏检查 | 全部用例结束后无残留 goroutine（用轮询断言或 Go 1.27 的 goroutine 泄漏 profile，见 <https://go.dev/doc/go1.27>） |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 三种模式正确性 | 35 | Worker Pool 三段式正确；Pipeline 三阶段可独立测试；Fan-in/Fan-out 的关闭责任清晰且有 panic 演示用例 |
| 取消与泄漏治理 | 25 | 全局与单任务超时都生效；取消后 goroutine 回落且有稳定断言；context 不泄漏 |
| 有界与背压 | 15 | 并发上限可配置且有校验；缓冲与溢出策略有依据；有界/无界有数据对比 |
| 测试与 benchmark | 15 | 三个并发测试齐备；`-race` 干净；两个 benchmark 有结论 |
| 文档 | 10 | 关闭责任表、并发度调优表、goroutine 峰值表与实现一致 |

## 8. 提示与思路

- **先画生命周期再写代码**：把每个 channel 的生产者、消费者、关闭者各写一行，确认唯一之后再动手。
- **取消的传播靠 context，不靠标志位**：所有阻塞点都接受 `context.Context`，`select` 里必须包含取消分支。
- **基线测量放在测试开头**：先记录当前 goroutine 数作为基线，再发起工作，再轮询等待回落到基线 + 小容差。
- **把 panic 当错误处理**：worker 内用 `defer` 恢复，转成结构化错误送进结果通道，避免整个进程退出。
- **退避要可注入**：把退避函数或时钟做成参数，测试里换成即时实现，避免用例变慢。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 循环体内捕获循环变量 | goroutine 读到同一个值 | Go 1.22 起 `for range` 的循环变量每轮独立，但循环外声明的变量仍被共享 | 明确传参或局部遮蔽；外部声明的 `err`、`ctx` 不要跨 goroutine 读写 |
| 双重关闭 channel | panic: close of closed channel | 多个 goroutine 都执行关闭 | 关闭责任唯一，写进 README 规则表 |
| 用 `time.Sleep` 做同步 | 测试偶发失败或长期变慢 | 用睡眠代替等待信号 | 用 channel、`sync.WaitGroup` 或轮询 + 超时断言 |
| `select` 默认分支忙等 | CPU 打满、吞吐反而下降 | 用 `default` 轮询代替阻塞等待 | 去掉 `default`，或只在明确非阻塞语义时使用 |
| `WaitGroup` 在 goroutine 内 `Add` | `go vet` 报错，计数与实际 goroutine 数不一致 | 在启动后才 `Add`，`Wait` 可能提前返回 | 先 `Add` 再起 goroutine，或直接用 `sync.WaitGroup.Go`（Go 1.25+） |
| 忘记 `cancel()` | context 泄漏、goroutine 长期存活 | 创建了可取消 context 却没调用取消函数 | 创建后立即 `defer cancel()` |
| `errgroup` 忘记传 ctx | 取消不传播，其它 goroutine 继续跑 | 用新的 context 起任务 | 把 `errgroup` 与前缀 context 一起使用 |

## 10. 参考实现要点

- **Worker Pool 骨架**（三段式：投递 → 等齐 → 关闭结果；`sync.WaitGroup.Go` 为 Go 1.25+）：

```go
func Run(ctx context.Context, jobs []Job, concurrency int) <-chan Result {
	results := make(chan Result, concurrency)
	in := make(chan Job)
	var wg sync.WaitGroup
	for range concurrency {
		wg.Go(func() { // Go 1.25 起可用；旧写法：wg.Add(1) 后再 go func(){ defer wg.Done(); ... }()
			for j := range in {
				results <- doOne(ctx, j) // doOne 内部处理单任务超时与重试
			}
		})
	}
	go func() {
		defer close(in) // 唯一发送方负责关闭任务通道
		for _, j := range jobs {
			select {
			case in <- j:
			case <-ctx.Done():
				return
			}
		}
	}()
	go func() { wg.Wait(); close(results) }() // 全部 worker 退出后再关闭结果通道
	return results
}
```

- **channel 关闭责任表**（README 必写）：任务通道由投递方关闭；结果通道由「等待全部 worker 的一方」关闭；消费者永不关闭；每个通道只有一行「关闭者」。
- **稳定的 goroutine 断言写法**（示意，避免固定 sleep 猜时序）：

```go
func waitGoroutines(t *testing.T, target int) {
	t.Helper()
	deadline := time.Now().Add(5 * time.Second)
	for currentGoroutines() > target { // 自行实现：读 runtime 或 metrics
		if time.Now().After(deadline) {
			t.Fatalf("goroutines did not settle, target=%d", target)
		}
		time.Sleep(10 * time.Millisecond) // 轮询间隔，不是同步手段
	}
}
```

- **基准设计**：固定同一份输入，只改 `--concurrency`，同时记录吞吐、P95 与内存分配；把结果按并发度写成表格，指出拐点与拐点处的瓶颈（CPU、外部 IO 或调度）。
- **重试实现顺序**：判断错误是否可重试 → 检查 `context` 是否已取消 → 计算退避（指数 × 抖动）→ 等待可被取消的定时 → 计数 +1；超过最大次数返回最后一个错误。
- **自检**：`-race` 干净；取消用例在慢机器上也不 flaky；README 的每张表都能用仓库里的命令复现。
