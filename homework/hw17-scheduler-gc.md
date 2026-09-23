# 作业 17：调度器与 GC/内存深度实验报告

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W17–W19 |
| 发放日期 | 2027-01-25 |
| 交付日期 | 2027-02-14 |
| 预计工时 | 24 小时 |
| 难度 | 挑战 |
| 对应讲义 | L18 |
| 前置作业 | hw10 |

## 1. 背景与目标

本作业的核心产物不是「跑通的代码」，而是一份实验报告 `report.md`：每条结论都要有环境、方法、原始数据、图表说明与不确定性与反例。训练点是把「我觉得」换成「我测到」，并区分变量与噪声。

六个实验：GMP 调度延迟、goroutine 成本、GC 参数矩阵、分配与逃逸、`sync.Pool`、goroutine 泄漏检测。

## 2. 需求（必须项）

1. MUST 报告开头写明环境：CPU 型号与核数、内存、OS、Go 版本（基线 Go 1.25，本机 `go1.25.7`）、是否容器化、容器 CPU/内存 limit、`GOMAXPROCS` 实际取值与获取方式。
2. MUST 提供可复现脚本或 `Makefile`/`Taskfile`，他人按 README 能复现全部数据表。
3. MUST 实验一（GMP 与调度延迟）：写一个能暴露调度延迟的程序，混合「大量 goroutine」「阻塞系统调用」「P 数量不足」三类负载；用 `GODEBUG=schedtrace=1000` 与 `go tool trace` 采集；对比 `GOMAXPROCS=1`、`GOMAXPROCS=2` 与本机核数三档下的吞吐与尾延迟（P50/P99/P999），给出数据表与结论。
4. MUST 说明 Go 1.25 起容器感知 `GOMAXPROCS` 对实验一的影响（Linux 上考虑 cgroup CPU bandwidth limit，所有平台周期性更新；`GODEBUG=containermaxprocs=0` 与 `updatemaxprocs=0` 可关闭；另有 `runtime.SetDefaultGOMAXPROCS`），标注 Go 1.25+；容器内实验必须显式记录这一点。
5. MUST 实验二（goroutine 成本）：测量创建 10 万与 100 万 goroutine 的时间与内存，用 `runtime.ReadMemStats` 或 `runtime/metrics`（见 <https://pkg.go.dev/runtime/metrics>）采集；说明初始栈大小与栈增长对测量值的影响，并给出至少一次「创建但不执行」与「创建且阻塞」的对照。
6. MUST 实验三（GC 参数矩阵）：同一负载下跑 `GOGC`（off/100/400）与 `GOMEMLIMIT`（至少三档）的组合矩阵，记录吞吐、P99、RSS、GC 周期数与 CPU 占用；输出「什么负载下调哪一个」的结论表。
7. MUST 用 `GODEBUG=gctrace=1` 至少完整解读一次输出：逐字段说明含义，并与 `runtime/metrics` 中的对应指标对照。
8. MUST 说明 Green Tea GC：Go 1.25 的实验开关 `GOEXPERIMENT=greenteagc`（官方称 GC 开销降低 10%–40%），Go 1.26 起默认启用、`GOEXPERIMENT=nogreenteagc` 关闭；若实验环境为 1.26+，须说明默认启用对结论的影响（标注版本）。
9. MUST 说明 Go 1.27 起小于 80 字节的分配使用 size-specialized malloc（开销最多降 30%，`GOEXPERIMENT=nosizespecializedmalloc` 关闭），并说明它对实验二/实验四小对象测量结果的影响（标注 Go 1.27+）。
10. MUST 实验四（分配与逃逸）：选 5 段代码（闭包捕获、返回指针、interface 装箱、切片预分配、`sync.Pool` 复用），用 `go build -gcflags=-m` 观察逃逸决策，用 benchmark 测 `allocs/op` 与 `ns/op`，给出「改写前 → 改写后」对照表。
11. MUST 实验五（`sync.Pool`）：测量不同对象大小与不同并发度下的命中率与收益；说明它为什么不能当长期缓存（对象可能在任意 GC 周期被清理、没有容量语义）；给出「值得用 / 不值得用」的判断标准。
12. MUST 实验六（goroutine 泄漏检测）：构造三种泄漏（发送方无人接收、接收方无人发送、忘记取消），用普通 goroutine profile 定位；说明 Go 1.26 的实验 `GOEXPERIMENT=goroutineleakprofile` 与 Go 1.27 转正的 `runtime/pprof` `goroutineleak` profile、端点 `/debug/pprof/goroutineleak` 如何直接报出泄漏及其基于可达性的判定原理（标注版本）。
13. MUST 每条实验附「不确定性与反例」小节，至少给出一条「结论可能不成立」的边界条件。
14. MUST 原始 profile/trace 文件或采集说明随包提交（大文件可只给采集命令与校验值）。

## 3. 需求（加分项）

1. 用 `runtime/trace.FlightRecorder`（Go 1.25+，配合 `FlightRecorderConfig`）做「出问题才导出」的低开销采集，并对比其开销。
2. 用 `testing/synctest`（Go 1.25 转正：`synctest.Test`、`synctest.Wait`）写确定性调度测试，作为实验六的补充。
3. 给出容器 CPU limit 与 `GOMAXPROCS` 的敏感性曲线（限流比例 vs P99）。
4. 把 GC 矩阵导出 CSV 并绘图，CSV 一并提交。
5. 复现一次 `GOGC=off` 的内存增长过程，记录多长时间触发 OOM 及当时的堆指标。

## 4. 技术约束

- 每条实验必须在同一台机器、同一 Go 版本下完成；跨机器数据不得混入同一张表。
- 每个数据点至少 5 次重复，报告须给中位数与分位，不得只给单次结果或只给平均值。
- 必须声明是否在容器内运行；容器内必须记录 CPU/内存 limit 与 `GOMAXPROCS`。
- 不得把 Go 1.26/1.27 才有的能力写成 1.25 可用；所有版本相关内容必须标注版本。
- 基准测试使用 `go test -bench`，不得用 `time.Now()` 手工计时冒充 benchmark（如确需手工计时，须说明理由并标注误差）。
- 报告中引用的第三方库只写模块路径并注明「以官方最新稳定版为准」。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `report.md` | 环境、方法、数据表、图表说明、结论、不确定性、反例 |
| `bench/` | 六个实验的可运行代码与 benchmark |
| `scripts/` | 采集脚本（`GODEBUG` 组合、profile 采集、CSV 导出） |
| `data/` | 原始数据 CSV、`gctrace`/`schedtrace` 输出片段 |
| `profiles/` 或采集说明 | profile/trace 文件，或复现命令与校验值 |
| `README.md` | 一页复现指南 |

## 6. 验收标准（可执行命令）

```bash
go test -bench=. -benchmem -count=5 ./bench/... | tee data/bench.txt
GODEBUG=gctrace=1 GOGC=100 go test -bench=BenchmarkGC -benchtime=10s ./bench/...
GODEBUG=schedtrace=1000 GOMAXPROCS=1 go run ./bench/sched
go test -run TestGoroutineLeak -v ./bench/leak
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine
go tool trace -http=:6061 trace.out
GOEXPERIMENT=greenteagc GOGC=100 go test -bench=BenchmarkGC ./bench/...   # Go 1.25
GOEXPERIMENT=nogreenteagc go test -bench=BenchmarkGC ./bench/...          # Go 1.26+
```

判定标准：README 中的每条命令可直接执行；`report.md` 中每张数据表都能对应到 `data/` 中的原始文件；泄漏实验的 profile 能明确指出泄漏 goroutine 的栈位置；GC 矩阵的每个单元格都有重复次数记录。

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 实验设计与可复现性 | 20 | 变量控制、重复次数、脚本一键复现 |
| 数据完整性与解读 | 30 | 原始数据齐全、报分位、`gctrace` 逐字段解读 |
| 结论准确性与边界 | 25 | 结论有数据支撑、给出适用范围与反例 |
| 泄漏检测 | 15 | 三种泄漏均可定位、能说明可达性判定原理 |
| 报告质量 | 10 | 结构清晰、图表可读、数字均有出处 |

## 8. 提示与思路

- 调度延迟实验的关键是让 goroutine 数远超 P 数，同时混入阻塞系统调用（文件 I/O 或 sleep 类调用），观察运行队列长度与尾延迟的关系；`schedtrace` 输出的队列字段可以佐证。
- 尾延迟必须报分位：平均值会把「少数 goroutine 长期得不到调度」这类问题洗掉。
- goroutine 成本测量要拆成「创建」「首次运行」「栈增长」三段；初始栈很小，创建 100 万个在百 MB 量级，但深调用栈会让栈翻倍增长。
- GC 矩阵建议先固定 `GOGC`，再固定 `GOMEMLIMIT`，避免两个变量同时变化导致无法归因；`GOGC=off` 只作为「无 GC 上界」的极端对照组，不能当优化方案。
- `sync.Pool` 的收益来自减少分配次数，判断标准是：对象构造成本高、生命周期短、允许随时被回收、压测显示 `allocs/op` 明显下降。
- 泄漏检测思路：先看 goroutine 数是否随负载回落，再抓 profile 看栈；可达性判定的意义是「即使 goroutine 阻塞在某个 channel 上，只要它已不可能被唤醒，就算泄漏」。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| 把单次测量当结论 | 结论无法复现 | 至少 5 次重复，报中位数与分位 |
| 不控制变量 | 无法归因 | 一次只改一个变量，其余固定并记录 |
| 只报平均值 | 尾延迟问题被掩盖 | 报 P50/P99/P999 |
| 容器内测 CPU 却忽略 `GOMAXPROCS` | 1.25+ 会自动对齐 CPU limit，与预期不符 | 记录 limit 与 `GOMAXPROCS`，必要时用 `GODEBUG=containermaxprocs=0` 对照 |
| 把 `GOGC=off` 当优化 | 内存无上界，最终 OOM | 用 `GOMEMLIMIT` 给上界，`off` 只做对照 |
| 忽略测试机噪声 | 数据抖动大 | 关闭无关负载、固定频率、多次重复取中位 |
| 过度外推单一负载 | 结论在别的负载下不成立 | 写明适用范围与反例 |
| 用 `time.Now()` 冒充 benchmark | 结果不可比 | 用 `go test -bench` 与 `-benchmem` |

## 10. 参考实现要点

- 内存采样骨架（字段含义以 `runtime.MemStats` 官方文档为准，见 <https://pkg.go.dev/runtime>）：

```go
var m runtime.MemStats
runtime.ReadMemStats(&m)
// 取两次采样的差值，关注堆内存总量、已分配量、GC 次数与累计暂停时间
```

- 调度延迟观测点：在 goroutine 中记录「入队时刻」与「实际开始执行时刻」之差，得到等待延迟分布；再用 `GODEBUG=schedtrace=1000` 的周期性输出交叉验证。
- GC 矩阵建议的组织方式：行是 `GOMEMLIMIT` 档位，列是 `GOGC` 取值，单元格填入 `{吞吐, P99, RSS, GC 周期数, CPU 占用}`；`off` 单独成列并注明不具备可比性。
- `gctrace` 解读要点：每次 GC 输出一行，关注周期序号、堆大小变化、各阶段耗时与 CPU 占比，把「哪一段变慢」对应回矩阵中的哪一档配置。
- 泄漏实验的三种构造：`ch <- v` 无人接收、`<-ch` 无人发送、`context` 未取消导致 goroutine 长期阻塞；每种都要有「修复前泄漏、修复后回落」的对照数据。
- 报告结构建议：环境 → 方法 → 每个实验的「假设 / 设置 / 原始数据 / 解读 / 不确定性」→ 总结论表 → 反例清单 → 复现命令附录。
