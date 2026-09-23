# L18 运行时原理：GMP、GC 与内存模型

（本篇定位：把「写 Go」升级为「知道 Go 运行时在替我们做什么」，用于解释调度延迟、内存增长、GC 停顿与并发正确性问题。对应 W17–W24，前置为 [L14](14-performance.md) 的 pprof 与逃逸分析、[L16](16-microservices-and-observability.md) 的运行时指标。语言基线 Go 1.25，涉及 1.26 / 1.27 的能力逐条标注版本。）

## 1. GMP 模型

| 角色 | 是什么 | 关键性质 |
| --- | --- | --- |
| G（goroutine） | 用户态协程，含栈、指令位置、状态 | 创建成本约几 KB 栈，可同时存在百万级 |
| M（machine） | 操作系统线程 | 真正的执行者；阻塞系统调用会占用一个 M |
| P（processor） | 调度上下文，持有本地运行队列与各类缓存 | **数量由 `GOMAXPROCS` 决定**，是并行度的上限 |

三者关系：M 必须绑定一个 P 才能执行 G；P 的数量固定（默认等于可用 CPU 数，可配置），M 的数量按需增长。这解释了「为什么 goroutine 数远超线程数仍能跑」和「为什么 `GOMAXPROCS` 影响并行度而不是并发度」。

```text
        全局运行队列（加锁，兜底）
              │ 取一批
              ▼
   P0 ──本地队列──► M（绑定 P 执行 G）──► G 运行中
   P1 ──本地队列──► M ──► G            └─ 阻塞时：handoff 把 P 交给其他 M
   P2 ──空闲──────┘                    └─ 网络等待：G 挂起，fd 注册进 netpoller
        ▲                                   ▲
        └──────── work stealing（偷一半）────┘
```

### 1.1 goroutine 的阻塞与唤醒

| 阻塞原因 | 运行时行为 | 是否占用线程 |
| --- | --- | --- |
| channel 收发（无数据可读/无人接收） | G 挂起，加入 channel 的等待队列，P 被释放 | 不占用 |
| 网络读写 | fd 注册到 netpoller，G 挂起 | 不占用 |
| 定时器 / `time.Sleep` | 交给运行时定时器，G 挂起 | 不占用 |
| `sync.Mutex` 竞争 | G 挂起在信号量上 | 不占用 |
| 同步文件 I/O、`os` 的阻塞调用 | 阻塞 M，触发 handoff | 占用一个 M（可能需要新建 M） |
| cgo 调用 | 阻塞 M，触发 handoff | 占用一个 M |
| `select` 多路等待 | 无就绪分支时挂起在多个等待队列，任一就绪即唤醒 | 不占用 |

「不占用线程」的等待是 Go 高并发的关键：连接数与线程数解耦。反过来，把同步文件 I/O 或 cgo 放在高并发路径上，线程数会随并发线性增长，调度延迟随之恶化。

调度结构：

| 结构 | 作用 | 设计动机 |
| --- | --- | --- |
| 本地运行队列（每 P 一个） | 存放待运行的 G，容量有限 | 大多数调度决策无需加锁 |
| 全局运行队列 | 存放溢出的 G | 兜底，加锁访问，作为负载均衡来源 |
| work stealing | 空闲 P 从其他 P 的本地队列尾部偷一半 | 避免「一个 P 忙死、其他 P 空闲」 |
| handoff | 当 M 因系统调用等原因阻塞时，把 P 交给其他 M | 保证 P 不被阻塞的 M 占住 |

系统调用与 netpoller：阻塞式系统调用会让 M 陷入内核，运行时把 P 交给另一个可用的 M（handoff），必要时创建新 M。网络 I/O 走另一条路——文件描述符注册到运行时的 netpoller（Linux 上是 epoll），goroutine 在等待网络时被挂起并让出 P，事件就绪后由 netpoller 唤醒。因此**「一个 goroutine 一个连接」的阻塞式写法在网络场景下不会浪费线程**；真正会浪费线程的是无法被 netpoller 接管的阻塞操作（同步文件 I/O、cgo、`time.Sleep` 期间之外的锁等待等）。

## 2. 调度细节

goroutine 状态机（简化）：可运行（在队列中等待 P）→ 运行中（绑定到 M/P）→ 等待（阻塞在 channel、锁、网络、定时器或系统调用）→ 可运行。所有「等待」到「可运行」的迁移都会把 G 放回队列，而不是就地忙等。

| 机制 | 说明 |
| --- | --- |
| 协作式抢占 | 在函数序言等安全点检查是否需要让出（早期实现的主要方式） |
| 基于信号的异步抢占 | 用信号打断长时间无安全点的循环，避免一个死循环 goroutine 独占 P |
| `sysmon` | 后台监控线程：抢占长时间运行的 G、回收长时间处于系统调用的 P、触发强制 GC、归还物理内存给操作系统 |
| 调度延迟来源 | P 不足（`GOMAXPROCS` 太小）、长 GC 停顿、频繁系统调用或 cgo、锁竞争、单次运行过长的 G |

**Go 1.25 起容器感知 `GOMAXPROCS`**：Linux 上会考虑 cgroup CPU bandwidth limit，并在所有平台周期性更新，避免 `GOMAXPROCS` 远大于实际可用 CPU 配额；可用 `GODEBUG=containermaxprocs=0` 与 `GODEBUG=updatemaxprocs=0` 分别关闭容器感知与周期更新，或用 1.25 新增的 `runtime.SetDefaultGOMAXPROCS` 显式设定默认值。

观察手段：

| 手段 | 用途 |
| --- | --- |
| `GODEBUG=schedtrace=1000` | 每秒打印一行调度器状态：gomaxprocs、idle/runqueue 长度、线程数、GC 状态 |
| `runtime/trace` | 生成可交互的追踪：G 的生命周期、P 的利用率、系统调用、GC 与阻塞事件 |
| `runtime/trace.FlightRecorder`（1.25 新增） | 内存环形缓冲，用 `WriteTo` 导出最近数秒的 trace，适合「事后抓最后一次故障」 |
| goroutine 数量与线程数指标 | 1.26 起 `runtime/metrics` 新增 `/sched/goroutines*`、`/sched/threads:threads`、`/sched/goroutines-created:goroutines` |

### 2.1 观察与设置运行时参数的骨架

```go
package main

import (
	"log/slog"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	// 1.25 起容器感知 GOMAXPROCS 默认生效；此处演示显式设定与查看生效值
	runtime.SetDefaultGOMAXPROCS(runtime.GOMAXPROCS(0))
	slog.Info("runtime", slog.Int("gomaxprocs", runtime.GOMAXPROCS(0)))

	// GOMEMLIMIT：软上限，通常取容器内存 limit 的 70%–80%
	debug.SetMemoryLimit(768 << 20)

	// 周期性输出调度与内存概况，便于和生产指标对照
	go func() {
		t := time.NewTicker(10 * time.Second)
		defer t.Stop()
		for range t.C {
			var ms runtime.MemStats
			runtime.ReadMemStats(&ms)
			slog.Info("runtime stats",
				slog.Int("goroutines", runtime.NumGoroutine()),
				slog.Uint64("heap_alloc_mb", ms.HeapAlloc>>20),
				slog.Uint64("gc_count", uint64(ms.NumGC)),
			)
		}
	}()

	select {} // 真实程序里换成正常的启动与退出逻辑
}
```

注意 `runtime.ReadMemStats` 会短暂停顿所有 goroutine（STW），只适合低频采样；高频采集请使用 `runtime/metrics`（见 <https://pkg.go.dev/runtime/metrics>）。

## 3. 栈、堆与逃逸分析

| 概念 | 机制 |
| --- | --- |
| goroutine 栈 | 初始很小，按需增长；采用连续栈：容量不足时分配更大的栈并**整体拷贝**（copy stack），因此栈上对象的地址会变化 |
| 收缩 | 长时间使用很浅的栈会被收缩，避免大量 goroutine 各占大栈 |
| 栈上分配条件 | 编译器能证明对象不会逃出当前函数（不被返回、不被存入堆对象、不被发送到 channel 之外等） |
| 逃逸分析 | 编译期决定对象去栈还是去堆；用 `go build -gcflags=-m` 观察判定结果 |

### 3.1 常见逃逸原因与改法

| 写法 | 为什么逃逸 | 改法 |
| --- | --- | --- |
| 函数返回指向局部变量的指针 | 生命周期超出函数 | 若能改为返回值语义或由调用方传入缓冲，可留在栈上 |
| 把值存入 `any` 或接口 | 编译器无法确定动态类型的大小与生命周期 | 缩减接口边界，热路径用具体类型或泛型 |
| 闭包捕获局部变量且闭包逃逸 | 变量必须活到闭包执行时 | 把捕获参数改为显式入参，或避免把闭包存进堆结构 |
| 把对象发到 channel | 接收方可能在另一个 goroutine 使用 | 传值或传 id，把所有权问题显式化 |
| 存入切片/map 的元素是接口 | 元素本身逃逸到堆容器 | 用具体类型容器，或预分配后复用 |
| 变量过大（超过栈容量阈值） | 放进栈会浪费栈空间 | 减小结构体或改为按需分配 |
| `defer` 捕获了逃逸的参数 | 参数需活到函数返回 | 参数改为基本类型，避免 `defer` 里用闭包捕获大对象 |

判断顺序建议：先用 `-gcflags=-m` 找出逃逸点，再判断它是否在热路径上，最后才动手改——不是所有逃逸都值得消除，堆分配有时比复制更大对象更便宜。

**Go 1.25 起切片的底层数组在更多情况下可栈分配**（编译器改进）。切片相关的逃逸判定还有一个开关：`-gcflags=all=-d=variablemakehash=n` 用于关闭「按变量名生成 make 哈希」的行为，便于把不同位置的 `make` 调用分开统计与对比——它是诊断手段，不是性能优化手段，不要放进生产构建。

可观测项（标准库接口见 <https://pkg.go.dev/runtime>）：

| 观测项 | 含义 |
| --- | --- |
| `runtime.NumGoroutine` | 当前 goroutine 数量，突增往往意味着泄漏 |
| `runtime.ReadMemStats` | 堆分配总量、当前堆大小、GC 次数与暂停时间累计等 |
| `runtime/metrics` | 结构化指标集合，是采集端的稳定接口 |

### 3.2 一次分配经过哪几层

| 层 | 职责 | 对性能的意义 |
| --- | --- | --- |
| 小于 16 字节的极小对象 | 多个对象被合并到同一个内存块中紧凑摆放 | 单独申请极小对象几乎不额外花钱，但不释放其中单个对象 |
| 按尺寸分级的小对象分配 | 按 size class 归类，从每 P 的本地分配缓存取内存 | 无锁快速路径，是绝大多数分配走的路 |
| 本地缓存不足 | 向中心缓存批量补充 | 加锁，但摊薄到多次分配上 |
| 中心缓存不足 | 向堆申请新的内存块（span） | 更慢，可能触发 GC |
| 大对象 | 直接从堆按页申请 | 走慢路径，且大对象更容易造成内存碎片 |

这解释了三条常见结论：大量小对象分配本身不一定慢，**真正的成本在于它们最终都要被 GC 扫描与回收**；减少分配要从「每个请求创建的对象数量」入手；预分配并复用缓冲（`sync.Pool`、`bytes.Buffer`）通常比微调分配器更有效。

排查 goroutine 泄漏：先看数量随请求量是否单调增长；再用 `net/http/pprof`（见 <https://pkg.go.dev/net/http/pprof>）抓 goroutine profile 看栈；1.26 起还有实验性的 goroutine 泄漏 profile（`GOEXPERIMENT=goroutineleakprofile`，暴露 `/debug/pprof/goroutineleak`），1.27 起该 profile 转正为 `runtime/pprof` 的 `goroutineleak` profile 与 `/debug/pprof/goroutineleak`。

## 4. 垃圾回收

三色标记-清除的直觉模型：所有对象初始为白；从根（栈、全局变量、寄存器）可达的直接引用染灰；从灰对象出发把引用对象染灰、自己染黑；灰集为空时仍为白的就是垃圾。**写屏障**在并发标记期间保证「黑色对象不会指向白色对象」这一不变量，否则会误删仍在使用的对象——这也是并发标记必须付的运行时开销。

| 阶段 | 是否 STW | 内容 |
| --- | --- | --- |
| 标记准备 | 是（短） | 开启写屏障、扫描栈与根 |
| 并发标记 | 否 | 与用户代码并行，是 CPU 开销的主要来源 |
| 标记终止 | 是（短） | 关闭写屏障、完成收尾 |
| 并发清扫 | 否 | 回收白色对象，按需归还内存 |

STW 的具体时长与阶段耗时可用 `runtime/metrics` 观察（GC 相关的停顿指标以官方指标名称为准）。触发条件有两个旋钮：`GOGC` 百分比决定「堆增长到上次存活量的多少倍时触发」，`GOMEMLIMIT`（1.19 引入）是内存软上限，达到时会更积极地触发 GC。两者可以组合：`GOGC` 控制 CPU 与内存的常规权衡，`GOMEMLIMIT` 作为兜底防止 OOM。

```bash
# 逐字段解读：GC 序号、各阶段时间、堆大小变化、目标、P 数量、CPU 利用率
GODEBUG=gctrace=1 ./app
# 例：gc 12 @3.104s 0%: 0.021+1.2+0.004 ms clock, 0.17+1.1/1.9/0.0+0.03 ms cpu, 4->4->2 MB, 5 MB goal, 8 P
#   gc 12        第 12 次 GC
#   @3.104s      程序启动后 3.104 秒
#   0%           被 GC 占用的 CPU 百分比
#   0.021+1.2+0.004 ms clock  标记准备 STW / 并发标记 / 标记终止 STW 的墙钟时间
#   4->4->2 MB   标记开始堆大小 -> 标记结束堆大小 -> 存活堆大小
#   5 MB goal    下一次触发 GC 的目标堆大小
#   8 P          参与运行的 P 数量
```

归还内存：Go 会把空闲内存留给运行时复用，长期不活跃时可调用 `runtime/debug` 包的 `FreeOSMemory`（见 <https://pkg.go.dev/runtime/debug>）强制归还——代价是紧接着的分配会重新向操作系统申请内存，属于「用 CPU 与延迟换 RSS」，只适合低频调用或明确的空闲期。

| 版本差异 | 内容 |
| --- | --- |
| Go 1.25 | 新 GC 实验 `GOEXPERIMENT=greenteagc`：标记与扫描小对象更好，GC 开销降低 10%–40% |
| Go 1.26 | Green Tea GC **默认启用**，用 `GOEXPERIMENT=nogreenteagc` 关闭（预计 1.27 移除该开关） |
| Go 1.26 | 64 位平台堆基址随机化，用 `GOEXPERIMENT=norandomizedheapbase64` 关闭 |
| Go 1.27 | 小于 80 字节的分配使用 size-specialized malloc，开销最多降低 30%，用 `GOEXPERIMENT=nosizespecializedmalloc` 关闭 |

版本相关的结论不能混用：`greenteagc` 在 1.25 上必须显式开启，在 1.26 上默认开启、需要显式关闭。

## 5. 对象生命周期工具

| 工具 | 引入版本 | 特点 |
| --- | --- | --- |
| finalizer（`runtime` 包的 finalizer 接口，见 <https://pkg.go.dev/runtime>） | 早期 | 只能按对象注册一个，回调在独立 goroutine 中串行执行，无法捕获闭包变量，容易阻塞回收 |
| `runtime.AddCleanup` | 1.24 | 可注册多个清理函数、可携带任意参数、与对象生命周期解耦更好 |
| 清理函数的并发执行 | 1.25 | `AddCleanup` 调度的清理函数现在并发并行执行；新增 `GODEBUG=checkfinalizers=1` 用于检查 finalizer/cleanup 相关问题 |
| `weak` 包 | 1.24 | 弱引用：不阻止对象被回收，适合做「可重建的缓存」——缓存未命中就重建，命中就复用 |

### 5.1 代码骨架

```go
// AddCleanup（1.24 起）：可注册多个清理函数，参数显式传递，不再依赖对象本身
type FileHandle struct{ f *os.File }

func OpenFile(name string) (*FileHandle, error) {
	f, err := os.Open(name)
	if err != nil {
		return nil, err
	}
	h := &FileHandle{f: f}
	// 清理函数在对象不可达后被调度；1.25 起多个清理函数并发并行执行
	runtime.AddCleanup(h, func(f *os.File) { _ = f.Close() }, f)
	return h, nil
}

// weak（1.24 起）：弱引用不阻止对象被回收，适合做「可重建的一级缓存」
var weakCache sync.Map // key -> weak.Pointer[T]（具体类型与函数以 https://pkg.go.dev/weak 为准）

func lookupOrBuild[T any](key string, build func() *T) *T {
	if p, ok := weakCache.Load(key); ok {
		if v := p.(interface{ Value() *T }).Value(); v != nil {
			return v // 命中且对象仍存活
		}
	}
	v := build()       // 未命中或已被回收：重建
	weakCache.Store(key, newWeakPointer(v))
	return v
}
```

弱引用的使用纪律：取出后必须立刻判空并在同一段逻辑内用掉，不能把取到的值跨步骤保存后假设它仍然有效。

弱引用的典型用途是内存敏感的多级缓存：强引用只保留热数据，冷数据用弱引用挂住，一旦 GC 需要内存就自然失效，而不是靠手写 TTL 与淘汰逻辑。注意弱引用随时可能变成 nil，取用后必须立即做非空判断并在同一段逻辑内使用。

清理函数不是析构函数：不要在里面做必须成功的持久化操作，执行时机不确定，也不能依赖它在进程退出前一定被调用。

## 6. `sync.Pool` 与同步原语选择

`sync.Pool` 是「临时对象复用池」，**不保证保留任何对象**：GC 可以随时清空它。因此它只能存放「可以廉价重建」的临时对象（缓冲区、编码器、临时结构体），不能当长期缓存用。

直觉模型：每个 P 有自己的私有槽位，减少锁竞争；回收时对象进入共享部分；此外还有 **victim cache**——上一轮 GC 的对象被移入 victim 区，若本轮未被使用才会真正释放，起到「给一次机会」的缓冲作用。这解释了为什么命中率会在 GC 之后短暂下降。

```go
var bufPool = sync.Pool{
	New: func() any { return new(bytes.Buffer) },
}

func encode(v any) ([]byte, error) {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset() // 必须重置：池里的对象可能带着上一次使用的内容
	defer func() {
		if buf.Cap() <= 64*1024 { // 过大的缓冲区不要放回池里，避免内存被长期钉住
			bufPool.Put(buf)
		}
	}()
	if err := json.NewEncoder(buf).Encode(v); err != nil {
		return nil, err
	}
	return append([]byte(nil), buf.Bytes()...), nil
}
```

### 6.1 同步原语的选择

| 场景 | 首选 | 理由 |
| --- | --- | --- |
| 一对一的交接、背压 | 无缓冲 channel | 语义即「交接完成」，天然同步 |
| 生产者-消费者队列 | 有缓冲 channel 或显式队列 | 缓冲吸收抖动，容量需要按业务延迟目标定 |
| 广播取消、超时 | `context` + `select` | 取消信号可级联，超时可组合 |
| 只保护少量字段的计数 | `sync/atomic` 的原子类型 | 无锁，适合计数器与标志位 |
| 保护复合不变量 | `sync.Mutex` | 语义清晰，优先正确性 |
| 读多写少且读端持锁时间长 | `sync.RWMutex` | 读并发；但写饥饿风险要评估 |
| 一次性初始化 | `sync.Once` / `OnceFunc` / `OnceValue` / `OnceValues` | 保证只执行一次且建立 happens-before |
| 等待一组任务结束 | `sync.WaitGroup`；1.25 起可用 `WaitGroup.Go` | `Go` 把「启动并计数」合成一步，避免 `Add` 与 `go` 顺序写错 |

```go
// 1.25 新增：WaitGroup.Go 等价于 Add(1) + go func(){ defer Done(); ... }()
var wg sync.WaitGroup
for _, job := range jobs {
	wg.Go(func() {
		process(job) // 注意：1.22 起循环变量每轮独立，这里不会踩到闭包陷阱
	})
}
wg.Wait()
```

`WaitGroup.Go` 的价值在于消除 `Add` 位置错误这一类 bug：`Add` 必须在 `Wait` 之前调用，把计数与启动写在一个方法里就从结构上排除了这种误用。

误用形态：把数据库连接、需要显式关闭的资源、带状态的业务对象放进池；取出后忘记 `Reset`；返回的对象持有大缓冲区导致内存长期不释放；以为 `Put` 之后一定能 `Get` 到同一个对象。

## 7. 内存模型与 `-race`

内存模型回答的是「一个 goroutine 的写，另一个 goroutine 什么时候一定能看到」。Go 的定义基于 happens-before：如果事件 A happens-before B，则 A 的效果对 B 可见。**happens-before 的来源**：

| 来源 | 建立的关系 |
| --- | --- |
| goroutine 创建 | `go` 语句之前的操作 happens-before 新 goroutine 的开始 |
| goroutine 结束 | goroutine 的结束不一定被外部观察到，需要显式同步（如 `WaitGroup`、channel） |
| channel 收发 | 同一 channel 上的发送 happens-before 对应接收完成 |
| 锁 | 同一把锁的 `Unlock` happens-before 后续的 `Lock` |
| `sync.Once` | `Do` 内的操作 happens-before 任一 `Do` 返回 |
| 原子操作 | `sync/atomic` 的原子操作构成同步点（顺序一致性语义） |

### 7.1 竞争与修复

```go
// 有竞争的版本：-race 会报告读写冲突
type counter struct{ n int }

func (c *counter) Inc() { c.n++ }        // 读-改-写不是原子操作
func (c *counter) Value() int { return c.n }

// 修复一：用原子类型（适合单一计数器）
type atomicCounter struct{ n atomic.Int64 }

func (c *atomicCounter) Inc()        { c.n.Add(1) }
func (c *atomicCounter) Value() int64 { return c.n.Load() }

// 修复二：用互斥锁（适合需要维护多个字段之间不变量的情况）
type lockedCounter struct {
	mu sync.Mutex
	n  int
}

func (c *lockedCounter) Inc() { c.mu.Lock(); defer c.mu.Unlock(); c.n++ }
```

```go
// channel 建立 happens-before 的直观例子：这个程序不会打印 0
func main() {
	ch := make(chan int)
	go func() {
		x := 42
		ch <- x // 发送 happens-before 对应接收完成
	}()
	fmt.Println(<-ch) // 读到的一定是 42，而不是「可能还没写」
}
```

为什么需要它：编译器和 CPU 都会重排指令、缓存写入。如果只按「代码顺序」推理，就会写出「本机测试通过、上线后偶发错误」的代码。典型症状是有时读到零值、有时读到半初始化对象。

`-race` 的原理是动态检测：在编译期插入对内存访问的记录，运行时维护每个内存位置的访问历史与 happens-before 关系（向量时钟），发现「两个未同步的访问、其中至少一个是写」就报错。局限：**只能发现实际执行到的路径**——没跑到的分支、只在生产并发度下出现的交错可能漏报；同时它对性能与内存开销很大，只在测试环境开启。因此 `-race` 通过不等于没有数据竞争，压力测试与代码审查仍不可省。

## 8. 源码阅读路线

| 入口文件 | 解决什么问题 |
| --- | --- |
| `runtime/proc.go` | 调度主循环、P 的获取与释放、work stealing、`sysmon` |
| `runtime/mgcsweep.go` | 清扫与 span 复用、内存归还 |
| `sync/pool.go` | 私有槽位、共享部分与 victim cache 的实现 |
| `runtime/chan.go` | channel 的环形缓冲、阻塞与唤醒、`select` 的实现 |

读源码要**带着问题**，否则会陷在细节里。可用的提问方式：「`sync.WaitGroup.Go`（1.25 新增）是怎么实现的——它和 `Add(1)` + `go func(){ defer Done() }()` 相比少写了什么、又交给运行时做了什么」「为什么关闭的 channel 可以无限次接收零值」「为什么对 nil channel 收发会永久阻塞」。

## 9. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 在循环里创建无退出条件的 goroutine | goroutine 数单调增长，内存不降 | 泄漏，没有退出信号 | 用 `context` 或关闭 channel 传递取消，并用 profile 验证数量回归 |
| 用全局变量传递「当前请求」 | 并发下串数据 | 缺少 happens-before 同步 | 显式传参，或用 `context` |
| 认为 `GOMAXPROCS` 越大越快 | 上下文切换增多、延迟变差 | 并行度受 CPU 配额约束 | 按容器 CPU 配额设置；1.25 起容器感知可自动处理 |
| 用 `sync.Pool` 当缓存 | 缓存命中率忽高忽低、数据意外不一致 | 池不保证保留，GC 会清空 | 池只放可重建的临时对象，长期缓存用显式结构 |
| 从池里取出对象不 `Reset` | 数据串味、出现上一次请求的内容 | 复用未清理状态 | 取出后立即 `Reset` |
| 依赖 finalizer/cleanup 做关键持久化 | 数据偶尔丢失 | 执行时机不确定 | 关键路径显式释放，清理函数只做兜底 |
| 靠 `time.Sleep` 等待初始化完成 | 本机正常、压测偶发失败 | 睡眠不能建立 happens-before | 用 channel、`sync.Once` 或 `WaitGroup` 同步 |
| 只跑 `-race` 就认为并发安全 | 上线后仍有数据竞争 | 动态检测只覆盖实际路径 | `-race` + 压力测试 + 审查共享状态 |
| 用 `GOGC=off` 或过大值省 CPU | 内存持续增长直到 OOM | 放弃 GC 触发保护 | 用 `GOGC` 调比例，用 `GOMEMLIMIT` 兜底 |
| 把 `FreeOSMemory` 放进请求路径 | 延迟毛刺、CPU 升高 | 归还内存后立刻重新申请 | 只在低峰期或明确的空闲期调用 |
| 相信 1.25 上已有 Green Tea GC 收益 | 实测无变化 | 1.25 需 `GOEXPERIMENT=greenteagc` 显式开启 | 1.25 显式开启，1.26 起默认启用 |
| 用逃逸分析开关做生产优化 | 行为与性能可能变差 | 把诊断开关当调优手段 | 用 `-gcflags=-m` 观察，靠改代码减少逃逸 |
| 阻塞式同步文件 I/O 写在热点路径 | 线程数暴涨、调度延迟升高 | 该操作无法被 netpoller 接管 | 放到受限的 worker 池，或改异步 API |
| 长循环里不检查取消 | 抢占与取消都慢，请求超时后仍在跑 | 缺少让出点与取消检查 | 循环内检查 `ctx.Done()`，把大循环拆小 |

## 10. 动手练习（可执行实验）

每个实验都要求「固定基线 → 只改一个变量 → 记录指标 → 复现结论」。

1. **不同 `GOMAXPROCS` 下的调度延迟**：写一个生成 CPU 密集任务的程序，分别在 `GOMAXPROCS=1/2/4/8` 下运行，用 `runtime/trace` 观察 P 的利用率与 goroutine 等待时间，记录每次的任务完成耗时与 P 空转比例。
2. **容器感知 `GOMAXPROCS` 验证**：在容器内把 CPU 配额设为 1 核但让宿主机有多核，分别在「默认」「`GODEBUG=containermaxprocs=0`」两种情况下打印生效的并发度并压测，记录 CPU throttling 比例与 P99 延迟差异。
3. **`GOGC` 与 `GOMEMLIMIT` 组合**：同一压测程序跑三组——默认、`GOGC=50`、`GOGC=400` 且设置 `GOMEMLIMIT`，记录吞吐、GC 次数、堆峰值与单次停顿最大值。
4. **`gctrace` 字段解读**：用 `GODEBUG=gctrace=1` 采集 30 秒输出，逐字段写出数值含义，并找出停顿最长的一次 GC，说明它的触发原因。
5. **`sync.Pool` 命中率**：为一个编解码函数实现「每次新建」与「用池复用」两版，记录分配字节数、分配次数（用 `testing.AllocsPerRun` 或 benchmark 的 `allocs/op`）与 GC 次数；再故意取用后不 `Reset`，观察数据串味。
6. **逃逸分析前后对比**：用 `go build -gcflags=-m` 找出会逃逸的返回值或接口装箱，通过改签名或预分配消除逃逸，记录改动前后的 `alloc/op` 与 `ns/op`。
7. **切片栈分配观察**：用 `-gcflags=all=-d=variablemakehash=n` 与默认设置分别编译同一段含多处 `make` 的切片代码，对比逃逸分析输出，说明差异来自哪里，并明确结论只用于诊断。
8. **goroutine 泄漏定位**：写一个忘记退出的 goroutine，用 goroutine profile 找到它，修复后用 `runtime.NumGoroutine` 与 `/sched/goroutines:goroutines` 验证数量回到基线。

## 11. 自检清单

- [ ] 能解释 `GOMAXPROCS`、P、M 三者的数量关系，以及它影响的是并行度而非并发度。
- [ ] 能说明 work stealing 与 handoff 各自解决什么问题。
- [ ] 知道网络 I/O 由 netpoller 接管，而同步文件 I/O 与 cgo 会真正占用线程。
- [ ] 能列出调度延迟的四类来源，并能用 `runtime/trace` 或 `schedtrace` 定位其中一类。
- [ ] 能解释连续栈的拷贝行为，以及为什么栈上对象地址会变。
- [ ] 能用逃逸分析输出判断一个对象去栈还是去堆。
- [ ] 能画出三色标记流程并说明写屏障的作用。
- [ ] 能解释 `GOGC` 与 `GOMEMLIMIT` 的分工，并说明软上限的含义。
- [ ] 能逐字段解读 `gctrace` 输出，并说明 `FreeOSMemory` 的代价。
- [ ] 能准确说出 Green Tea GC、堆基址随机化、size-specialized malloc 各自在哪个版本生效与如何关闭。
- [ ] 能说明 `AddCleanup` 相对 finalizer 的优势与执行时机的不确定性。
- [ ] 能说明 `sync.Pool` 不保证保留的原因，以及 victim cache 的作用。
- [ ] 能列举 happens-before 的五类来源，并说明为什么需要内存模型。
- [ ] 能说明 `-race` 的检测原理与「通过不等于安全」的原因。

## 12. 延伸阅读

- Go 内存模型：<https://go.dev/ref/mem>
- 调度器设计文档：<https://go.dev/doc/go1.25>（容器感知 `GOMAXPROCS`、`FlightRecorder`、Green Tea GC 实验入口）
- GC 指南：<https://go.dev/doc/gc-guide>（`GOGC` 与 `GOMEMLIMIT` 的组合与调优）
- `runtime` 包文档：<https://pkg.go.dev/runtime>；`runtime/metrics`：<https://pkg.go.dev/runtime/metrics>；`runtime/trace`：<https://pkg.go.dev/runtime/trace>
- `sync` 包文档：<https://pkg.go.dev/sync>；`sync.Pool` 说明：<https://pkg.go.dev/sync#Pool>
- Go 官方 Release Notes：<https://go.dev/doc/go1.24>、<https://go.dev/doc/go1.25>、<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>
- 《Go 语言设计与实现》，作者：左书祺（Draven），出版社：人民邮电出版社
- 《The Go Programming Language》，作者：Alan A. A. Donovan、Brian W. Kernighan，出版社：Addison-Wesley
- 相关讲义：[L14 性能分析](14-performance.md)、[L16 微服务架构与可观测性](16-microservices-and-observability.md)、[L19 分布式基础](19-distributed-systems.md)
