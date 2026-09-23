# L11 并发原语与 context

本篇定位：把 Go 并发的「地基」讲清楚——goroutine 是什么、channel 与 `select` 的同步语义、`sync` 与 `sync/atomic` 的取舍、happens-before 与 `-race`、`context` 的构造与传播纪律、goroutine 泄漏的排查思路。对应 W7–W8（阶段三）。前置：会写基本 Go 程序，用过 `go func`、`chan`、`error`、`defer`；综合示例建议先读 [L03 错误处理与结构化日志](03-errors-and-logging.md) 的错误包装约定。

命名约定：标准库标识符一律以 <https://pkg.go.dev> 官方文档为准；本仓库事实基线未逐一列出的类型或函数，本文只给包路径与官方链接，不手写签名，避免与实现漂移。

## 1. goroutine 的本质与成本

| 维度 | goroutine | OS 线程 |
| --- | --- | --- |
| 初始栈 | 很小，KB 量级 | MB 量级 |
| 栈增长 | 按需增长，也可收缩 | 创建时基本定死 |
| 调度者 | Go 运行时（用户态调度） | 操作系统内核 |
| 切换成本 | 较低，不陷入内核 | 较高，涉及内核态 |
| 数量级 | 十万级常见 | 数千级即吃紧 |

运行时把大量 goroutine 复用到少量 OS 线程上，因此起一个 goroutine 既不需要系统调用，也不预留 MB 级栈。但**创建廉价不等于免费**：每个存活 goroutine 至少占用栈内存与调度结构，并让它引用的对象保持可达（GC 无法回收）。`for _, item := range items { go handle(item) }` 这类写法在 items 有百万条时会让峰值 goroutine 数等于数据量。

正确做法是固定数量的 worker 消费任务 channel，见 [L12 并发模式与并发安全](12-concurrency-patterns.md) 第 1 节。判断标准不是「能不能起」，而是「同时存活的峰值有多少、它们什么时候退出」。

## 2. channel 的语义

| 类型 | 发送何时完成 | 接收何时完成 | 语义 |
| --- | --- | --- | --- |
| 无缓冲 `make(chan T)` | 有接收方接手时 | 有发送方交付时 | 交接（rendezvous），强同步点 |
| 有缓冲 `make(chan T, n)` | 缓冲未满时立即返回 | 缓冲非空时立即返回 | 队列，缓冲满时发送阻塞 |

无缓冲 channel 的发送方返回时，可以确定接收方已经拿到值。有缓冲 channel 只保证「值进了队列」，不保证「对方处理了」；把它当异步通知用时，必须想清楚缓冲耗尽后发送方会被阻塞——这其实是背压，见 L12 第 4 节。

`close` 的规则：

1. **只有发送方关闭**。接收方关闭会与「发送方仍在发送」竞态。
2. **多个发送方需要协调者**：由汇总方关闭，或用 `sync.WaitGroup` 等所有发送方退出后再关闭。
3. **向已关闭 channel 发送会 panic**（`send on closed channel`）；**重复 close 也会 panic**（`close of closed channel`）。
4. 关闭是「不会再有值」的广播，不是传递数据的手段。
5. 关闭后已缓冲的值仍可读完，排空前 `ok` 仍为 `true`；排空后读到零值且 `ok == false`，且可无限次读取，不会阻塞。

```go
ch := make(chan int, 2)
ch <- 1
close(ch)
v, ok := <-ch // v == 1, ok == true
v, ok = <-ch  // v == 0, ok == false —— 已关闭且已排空，返回零值
v, ok = <-ch  // 仍然 v == 0, ok == false
```

零值 channel 是 `nil`，对它的发送与接收都**永久阻塞**（不 panic），`close(nil)` 会 panic——这是静默泄漏的常见来源。它同时是 `select` 的实用技巧：把某分支的 channel 设为 `nil`，该分支永不就绪，等价于禁用分支。

```go
for a != nil || b != nil { // nil channel 在 select 中永不就绪
	select {
	case v, ok := <-a:
		if !ok { a = nil; continue } // 关闭后置 nil：禁用该分支，避免零值刷屏
		out <- v
	case v, ok := <-b:
		if !ok { b = nil; continue }
		out <- v
	}
}
```

## 3. select 与定时器

`select` 在多个 channel 操作中等待，随机选择一个**当前就绪**的分支，因此没有固定优先级造成的饿死。没有就绪分支时：有 `default` 就执行它，没有就阻塞。`default` 不是「等一会儿」而是「现在没就绪就走」，`for { select { case v := <-ch: handle(v); default: } }` 这种写法会把一个核跑满。

标准写法是 `for { select { ... case <-ctx.Done(): return ctx.Err() } }`，让取消成为唯一的退出通道。超时场景在热循环里另有一个坑：`time.After` 每次调用都新建定时器，未触发前不回收，高频循环会持续堆积内存。改成可复用定时器：

```go
t := time.NewTimer(idle) // 循环外创建一次
defer t.Stop()
for {
	select {
	case <-ctx.Done():
		return ctx.Err()
	case v := <-ch:
		handle(v)
		t.Stop() // Go 1.23 起 timer channel 无缓冲：Stop 返回 false 表示值已被取走
		t.Reset(idle) // 或正在投递，无需再手动排空 channel
	case <-t.C:
		onIdle()
		t.Reset(idle)
	}
}
```
版本要点：**Go 1.23** 起 `time.Timer` 的 channel 变为无缓冲（同步交付），旧代码里 `Stop`/`Reset` 之后「非阻塞读一次 channel 排空」的写法不再需要，保留反而可能吃掉下一轮信号；**Go 1.27** 起 `asynctimerchan` 这个 `GODEBUG` 开关被永久移除，`time` 包的 channel 恒为无缓冲。以 1.23+ 为基线写代码即可，不要再写依赖缓冲语义的兼容分支。`Reset` 的姿势是「先 `Stop` 再 `Reset`」，并保证 `Reset` 与 `t.C` 的接收在同一 goroutine 中协作。

## 4. sync 包的原语

| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| 临界区短、写不少 | `sync.Mutex` | 一条原子路径，开销稳定 |
| 读多写少、临界区较长 | `sync.RWMutex` | 允许读并发 |
| 只保护一个整数或指针 | `sync/atomic` | 无锁，成本最低 |

**RWMutex 不一定更快**：它要维护读者计数，读锁本身也是原子操作与运行时调度路径；临界区只有几条指令或写比例不低时，额外开销与写者饥饿风险会让它慢于 `Mutex`。规则是「先用 `Mutex`，压测证明读是瓶颈再换」。另外 `RWMutex` 不可递归取读锁（读锁内再取读锁可能死锁）。

**WaitGroup 与 Go 1.25 的 `WaitGroup.Go`**：

```go
var wg sync.WaitGroup // 经典三段式（所有版本可用）
for _, u := range urls {
	wg.Add(1) // 必须在 go 之前
	go func(u string) { defer wg.Done(); fetch(u) }(u)
}
wg.Wait()

var wg2 sync.WaitGroup // Go 1.25+：Add + go + defer Done 合并为一次调用
for _, u := range urls { // Go 1.22 起循环变量每轮独立，可直接捕获
	wg2.Go(func() { fetch(u) })
}
wg2.Wait()
```

两者语义等价，区别在于 `Go` 在启动时内部完成计数登记，因此 `Wait` 与 `Go` 并发调用也安全，`Add`/`Done` 不配对的整类错误被消除。工具链侧配合 **Go 1.25 的 `go vet`** 新增的 `waitgroup` 分析器（检查 `Add` 位置错误）；**Go 1.27** 该分析器改名为 `waitgroupgo`。

**Once 家族**：`sync.Once` 保证多 goroutine 下只执行一次，用于单例、连接池、注册表。**Go 1.21** 增加三个薄封装，省掉包级变量：`sync.OnceFunc`、`sync.OnceValue`、`sync.OnceValues`。

```go
var (
	once   sync.Once
	client *http.Client
)
func getClient() *http.Client { once.Do(func() { client = &http.Client{} }); return client }

var loadConfig = sync.OnceValue(func() (*Config, error) { return parseConfig("config.yaml") })
cfg, err := loadConfig() // 首次调用才执行，之后返回同一份结果
```

`Once` 是「进程内只执行一次」而不是缓存；配置热更新、缓存失效重建不能用它，见 L12 第 8 节。

**Cond**：`sync.Cond` 用于等待由锁保护的条件成立——`Wait` 原子地释放锁并挂起，被唤醒后重新加锁。适合「多等待者 + 广播」，如自建队列、连接池借还。

```go
type Queue struct{ mu sync.Mutex; cond *sync.Cond; items []int }

func NewQueue() *Queue { q := &Queue{}; q.cond = sync.NewCond(&q.mu); return q }
func (q *Queue) Push(v int) { q.mu.Lock(); q.items = append(q.items, v); q.mu.Unlock(); q.cond.Signal() }
func (q *Queue) Pop() int {
	q.mu.Lock(); defer q.mu.Unlock()
	for len(q.items) == 0 { q.cond.Wait() } // 必须 for，不能 if
	v := q.items[0]; q.items = q.items[1:]
	return v
}
```

`Wait` 必须放在 `for` 里（可能虚假唤醒，或条件已被别人先消费）。多数业务场景用「channel + `select`」更直观，只有需要广播或在锁内等待时才优先 `Cond`。

**Pool**：`sync.Pool` 是临时对象复用，用来降低高频短命对象的分配与 GC 压力（典型：`[]byte` 缓冲、序列化临时结构）。用法是 `Get` 取出、用前重置（如切片 `[:0]`）、`defer Put` 归还。**不保证** `Put` 进去的对象被保留——GC 可在任意时刻清空池，且池按 P 局部化。可放：无状态或可完全重置的临时对象；不可放：有连接语义的对象（数据库连接、socket）、需要跨 GC 存活的对象、当缓存用的对象。

**sync/atomic 的类型化原子操作**：**Go 1.19** 引入 `atomic.Int64`、`atomic.Bool`、`atomic.Pointer` 等类型，比旧的函数式 API 更难用错（例如计数器用 `Add(1)` 与 `Load()`，标志位用 `Store`/`Load`）。泛型形式 `atomic.Pointer[T]` 省掉类型断言，是无锁读共享结构的首选（配置热更新的完整模式见 L12 第 8 节）：

```go
type Router struct{ table atomic.Pointer[map[string]Handler] }

func (r *Router) Lookup(path string) (Handler, bool) { // 无锁读
	t := r.table.Load()
	if t == nil { return nil, false }
	h, ok := (*t)[path]
	return h, ok
}
func (r *Router) Reload(next map[string]Handler) { r.table.Store(&next) } // 整体替换
```

原子操作只对**单个字长字段**保证一致性；多字段必须一起变化时，用锁或「构造不可变快照 + 原子替换」。

## 5. 内存模型与 happens-before

Go 内存模型给的是 happens-before 关系，不是「按书写顺序执行」的承诺。编译器和 CPU 都可以重排无依赖访问、把变量缓存在寄存器、把共享变量提升到循环外。两个 goroutine 访问同一内存且至少一个是写、又无 happens-before 关系时就是数据竞争，结果可以是撕裂值、永久旧值，或随版本与架构变化。

```go
var n int
var wg sync.WaitGroup
for i := 0; i < 100; i++ {
	wg.Add(1)
	go func() { defer wg.Done(); n++ }() // 读-改-写三步，非原子
}
wg.Wait()
fmt.Println(n) // 通常小于 100，且每次不同
```

| 同步手段 | 建立的关系 |
| --- | --- |
| channel 发送完成 → 对应接收完成 | 发送前的写对接收方可见 |
| `close(ch)` → 读到零值/`ok == false` | 关闭前的所有写对接收方可见 |
| `Mutex.Unlock` → 后续 `Lock` | 解锁前的写对下一个持锁者可见 |
| `RWMutex` 写解锁 → 读加锁；读解锁 → 写加锁 | 同 `Mutex` |
| `Once.Do` 返回 → 任意后续 `Do` 返回 | 首次执行中的写对后来者可见 |
| `WaitGroup.Wait` 返回 → 被等待的 goroutine | 该 goroutine 的写在 `Wait` 之后可见 |
| 原子操作（如 `atomic.Pointer` 的 Load/Store） | 顺序一致性语义，可用于发布不可变快照 |

不要靠「时序上看起来先后」推理可见性，要靠同步点。`go test -race` / `go build -race` 打开基于 happens-before 的动态检测器：

| `-race` 能查 | `-race` 查不出 |
| --- | --- |
| 本次运行中真实发生的并发读写冲突 | 未被执行到的代码路径 |
| 未加锁的共享变量读写、并发 map 读写 | 逻辑竞态（TOCTOU，检查与使用之间有窗口） |
| 锁使用不一致（A 处加锁、B 处不加锁） | 死锁、活锁、goroutine 泄漏 |
| 闭包/共享结构捕获类错误 | 内存模型之外的资源竞态（文件、外部服务）、恰好串行的时序（假阴性） |

`-race` 通过只等于「这次没抓到」，必须配合 `-count`、`t.Parallel()` 与 CI 常规运行。三个会被抓到的典型例子：

```go
// 例 1：共享计数器未同步（上面的 n++），报 DATA RACE 并给出两个冲突栈
// 例 2：惰性初始化没有同步；并发写 map 还会 fatal error: concurrent map writes
var cache map[string]string
func get(k string) string {
	if cache == nil { // 检查
		cache = map[string]string{} // 使用：两个 goroutine 并发写
	}
	return cache[k]
}
// 例 3：靠「时间差」传递就绪状态，data 与 full 都会被报竞争
type Buf struct {
	data []byte
	full bool // 无同步的标志位
}
func (b *Buf) Fill(p []byte) { b.data = append(b.data, p...); b.full = true }
func (b *Buf) Ready() bool   { return b.full } // 另一 goroutine 轮询
```

例 3 的修法：`Mutex` 包住读写，或 `atomic.Bool` 加「先发布数据、再置标志」并让读侧原子读。`context.WithTimeout` 与 `WithDeadline` 的关系：前者是相对时间，内部同样落到绝对 deadline，父 context 更早到期时子立即到期，因此「层层加超时」不会延长上游预算。

## 6. context.Context 的构造与取消

| 构造 | 用途 | 注意 |
| --- | --- | --- |
| `context.Background()` | 进程/请求树的根 | 只用于 `main`、测试、初始化、顶层入口 |
| `context.TODO()` | 未确定来源的占位 | 表示「待补充」，不要长期留在生产代码 |
| `context.WithCancel(parent)` | 手工取消 | 返回的 cancel 必须被调用 |
| `context.WithTimeout(parent, d)` | 相对超时 | 内部是 deadline；父更早到期时子立即取消 |
| `context.WithDeadline(parent, t)` | 绝对截止时间 | 跨服务传递要统一时钟与时区 |
| `context.WithValue(parent, k, v)` | 请求域元数据 | 见下文 |

```go
func handle(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel() // 必须：否则父存活期间子节点不被释放
	return work(ctx)
}
```

**WithValue 的正确用法**。可以放：请求 ID、认证主体、幂等键、locale 等贯穿请求、只读、跨层透传的元数据。不要放：可选参数（应显式作函数参数）、依赖对象（应构造注入）、可变状态、敏感信息（容易被日志打印）。

```go
type ctxKey int // 未导出类型，避免与其他包的 key 冲突
const requestIDKey ctxKey = iota

func WithRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey, id)
}
func RequestID(ctx context.Context) (string, bool) { id, ok := ctx.Value(requestIDKey).(string); return id, ok }
```
**Done 与 Err**。`ctx.Done()` 在取消时关闭，是 `select` 的标准退出分支；`ctx.Err()` 给出原因（被取消时 `context.Canceled`，超时时 `context.DeadlineExceeded`）。判定取消优先用 `ctx.Err()`，不要解析下游错误字符串。

**cause（Go 1.20 / 1.21）**。`context.WithCancelCause` 允许取消时附带错误，接收方用配套读取函数（`context` 包，见 <https://pkg.go.dev/context#Cause>）取回，解决「只知道被取消、不知道因为谁」的问题。

```go
var errTooManyFailures = errors.New("too many failures")

ctx, cancel := context.WithCancelCause(parent)
defer cancel(nil)
if failures > limit { cancel(errTooManyFailures) }
if errors.Is(context.Cause(ctx), errTooManyFailures) { // 名称与签名以官方文档为准
	// 走「上游失败」分支，而不是「用户主动取消」分支
}
```

**Go 1.21** 的 `context.AfterFunc` 注册「context 取消后执行」的函数并返回撤销函数，适合把取消映射到关闭资源：`stop := context.AfterFunc(ctx, func() { conn.Close() })`，正常路径 `defer stop()` 以免连接被提前关闭。同一版本的 `context` 包还提供「切断取消传播但保留其中值」的构造函数（见 <https://pkg.go.dev/context>），适用于「请求已被取消，但仍要写审计日志/完成落库」的收尾路径；它只切断取消信号、不给新的截止时间，收尾工作应自己再加短超时，否则可能永久挂住。

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt) // 收到 Ctrl+C 即取消
defer stop() // 恢复默认信号行为，否则第二次信号会被吞掉
<-ctx.Done()

// 关闭流程必须用新的 ctx 派生超时，不要复用已被取消的 ctx
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx) // 等待在途请求结束
```

要点：Windows 上 `syscall.SIGTERM` 不可用，跨平台代码通常只监听 `os.Interrupt`，或把平台专用信号放进带构建标签的文件；关闭流程要用新的 `context.Background()` 派生超时（或用上面的断链构造函数），否则已被取消的 `ctx` 会让 `Shutdown` 立刻返回、在途请求被硬切。**Go 1.26** 起 `signal.NotifyContext` 用 `CancelCauseFunc` 取消，并把收到的信号作为 cause 带上，因此可以用前述 cause 读取函数区分「用户中断」与「编排系统要求退出」，直接写进结构化日志。

## 7. context 传播纪律

| 纪律 | 说明 |
| --- | --- |
| 作为第一个参数 | `func Do(ctx context.Context, req *Req) error`，参数名用 `ctx` |
| 不放进 struct | 请求域的值不应成为长生命周期对象的字段，否则生命周期错配 |
| 库函数不用 `context.Background()` | 自行造根会切断调用方的超时与取消 |
| 不在 `select` 之外忽略取消 | 长任务周期性检查 `ctx.Err()`，否则取消形同虚设 |
| 超时分层、预算传递 | 上游给 2s，下游最多花 1.5s；父 deadline 更早时子立即到期 |
| 取消只由创建者触发 | 只有创建 `WithCancel`/`WithTimeout` 的一方能 `cancel()` |
| 不用 context 传可选参数 | 参数显式化才能被静态检查与文档覆盖 |

```go
for _, item := range items { // 长循环：显式检查取消
	if err := ctx.Err(); err != nil {
		return err
	}
	process(item)
}
ch := make(chan error, 1) // 阻塞 IO：把阻塞转移到另一个 goroutine（要回传数据就传结构体）
go func() { _, err := io.ReadAll(r); ch <- err }()
select { // 缓冲 1：即使调用方已走，发送方也能退出，避免泄漏
case <-ctx.Done():
	return ctx.Err()
case err := <-ch:
	return err
}
```

`net/http` 的请求应把 `ctx` 绑定到请求上（例如经 `http.NewRequestWithContext`），取消时底层连接会被中断，比等 IO 自然返回更及时。

## 8. goroutine 泄漏与排查

goroutine 泄漏指 goroutine 永久阻塞、永不退出，持续占用栈与引用对象，表现为内存只增不减。三种常见形态：

| 形态 | 现象 | 典型代码 |
| --- | --- | --- |
| 发送方无人接收 | 卡在发送 | 向无缓冲/已满 channel 发送，接收方提前 return 或从未启动 |
| 接收方无人发送 | 卡在接收 | 等待一个永远不会有值、也不会被 close 的 channel |
| 忘记取消 | 子树永不结束 | `WithCancel` 的 cancel 未调用；循环未监听 `ctx.Done()` |

排查思路：抓 goroutine profile 看数量与阻塞栈（`net/http/pprof` 的 `/debug/pprof/goroutine`，见 <https://pkg.go.dev/net/http/pprof>）；在测试中重复执行并对比数量增长趋势，比看单次快照可靠（测试侧可用 Go 1.25 起正式可用的 `testing/synctest`）。

**Go 1.26** 提供实验性的 goroutine 泄漏 profile：以 `GOEXPERIMENT=goroutineleakprofile` 构建后暴露 `/debug/pprof/goroutineleak`；**Go 1.27** 该 profile 转正为 `runtime/pprof` 的 `goroutineleak` profile，并可经 `/debug/pprof/goroutineleak` 访问，无需实验开关。它的原理与普通 goroutine profile 不同：普通 profile 只列出当前 goroutine 及其栈，是否泄漏靠人判断；`goroutineleak` 基于**可达性分析**——如果某个阻塞原语（channel、锁等）已不可能再被任何 goroutine 唤醒或投递，且它自身从根不可达，那么等待它的 goroutine 被判定为永久阻塞，直接报为泄漏，显著减少人工甄别，长期合法存活的服务循环不会被误报。

预防性写法：每个 `go func()` 都要能回答「它在什么条件下退出」；跨 goroutine 传递结果用缓冲为 1 的 channel + `select`；每个 `WithCancel`/`WithTimeout` 都配 `defer cancel()`；用分组协调器统一等待（下节）。

## 9. 综合示例：带超时与取消传播的并发抓取

需求：并发抓取多个 URL，整体 3 秒超时，任一失败即取消其余任务，汇总成功结果。

```go
package fetch

// import 省略：context、errors、fmt、io、net/http、sync、time

var ErrFetch = errors.New("fetch failed")

type Result struct {
	URL    string
	Body   string
	Status int
}

// FetchAll 并发抓取 urls，整体超时 timeout；任一任务失败会取消其余任务。
func FetchAll(ctx context.Context, client *http.Client, urls []string, timeout time.Duration) ([]Result, error) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()
	var (
		mu      sync.Mutex
		results = make([]Result, 0, len(urls))
		wg      sync.WaitGroup
	)
	for _, u := range urls {
		wg.Go(func() { // Go 1.25+；旧版本写 wg.Add(1) + defer wg.Done()
			r, err := fetchOne(ctx, client, u)
			if err != nil {
				cancel(fmt.Errorf("%w: %s: %w", ErrFetch, u, err)) // 短路取消其余任务
				return
			}
			mu.Lock() // results 被多个 goroutine 追加，必须加锁
			results = append(results, r)
			mu.Unlock()
		})
	}
	wg.Wait() // 等所有子任务退出，保证没有 goroutine 泄漏
	if err := context.Cause(ctx); err != nil && !errors.Is(err, context.Canceled) {
		return nil, err // 名称与签名以官方文档为准
	}
	return results, nil
}

func fetchOne(ctx context.Context, client *http.Client, url string) (Result, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return Result{}, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return Result{}, err
	}
	defer resp.Body.Close()
	body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20)) // 限制单次读取上限
	if err != nil {
		return Result{}, err
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return Result{}, fmt.Errorf("unexpected status %d", resp.StatusCode)
	}
	return Result{URL: url, Body: string(body), Status: resp.StatusCode}, nil
}
```

设计要点：`wg.Wait()` 必须在读 cause 之前（否则结果可能不完整）；共享 `results` 用 `Mutex` 保护；子任务都从同一个 `ctx` 派生请求，取消能立刻传导到网络层。

用第三方分组库可以让这段更短：模块路径 `golang.org/x/sync/errgroup`（见 <https://pkg.go.dev/golang.org/x/sync/errgroup>，以官方最新稳定版为准）。语义是「分组 + 首错即取消同组 context + 统一等待」：注册的子任务各自返回 `error`，任一非 nil 即取消同组 context 并短路，最后统一等待并返回首个错误；带 context 的构造形式返回的 context 直接用于下游调用。具体函数与方法名以该链接文档为准，本文不复制签名。

## 10. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 循环里无限制 `go func()` | 内存暴涨、调度抖动 | 峰值 goroutine 数等于数据量 | 固定 worker 数 + 任务 channel |
| 接收方 `close(ch)` | `close of closed channel` panic | 关闭权归属错误 | 只由发送方关闭，多发送方由协调者关闭 |
| 向已关闭 channel 发送 | `send on closed channel` panic | 关闭与发送竞态 | 用 `done`/`ctx` 通知退出，不关闭共享 channel |
| `time.After` 写在 `for` 里 | 内存缓慢增长 | 每轮新建定时器，未触发不回收 | `time.NewTimer` + `Stop`/`Reset` |
| `select` 里用 `default` 做等待 | CPU 100% | `default` 立即执行，形成忙等 | 去掉 `default` 或加真实阻塞 |
| 用 `time.Sleep` 做同步 | 测试偶发失败 | 睡眠不建立 happens-before | 用 channel、`WaitGroup`、`Once` |
| 轮询 `ctx.Err()` 空转 | CPU 占用高 | 用循环 poll 代替阻塞等待 | `select` 监听 `ctx.Done()` |
| 忘记调用 `cancel` | goroutine 与 context 子树泄漏 | 缺少 `defer cancel()` | 每个 `WithCancel`/`WithTimeout` 都配 `defer cancel()` |
| `WaitGroup.Add` 写在 `go` 之后 | 竞态、计数错乱 | `Add` 与 `Wait` 并发 | `Add` 在 `go` 之前，或用 `WaitGroup.Go`（1.25+） |
| 把 `ctx` 存进 struct 字段 | 取消/超时行为错乱 | 生命周期与请求域错配 | 每个方法显式传 `ctx` 作首参 |
| 库内自造 `context.Background()` | 上游取消与超时失效 | 断开上下文树 | 沿用调用方传入的 `ctx` |
| 多字段分别用原子变量更新 | 读到不一致中间状态 | 单字段原子不保证组合一致性 | 用锁，或不可变快照 + `atomic.Pointer[T]` |
| 把 `sync.Pool` 当缓存 | 命中率不稳、状态污染 | 池内容随时可能被清空 | 只放可完全重置的临时对象 |
| 依赖「线程时序」而非同步点 | `-race` 报警或线上偶发错误 | 缺少 happens-before | 按第 5 节的同步手段建立关系 |

## 11. 动手练习

1. 写 `merge(done, a, b)` 把两个 channel 合并成一个，正确处理某一侧先关闭（用 `nil` channel 禁用分支），并用 `-race` 跑测试。
2. 把第 3 节的 `time.After` 反例改成可复用的 `time.NewTimer` 版本，并说明「Go 1.23 起 `Stop` 返回 false 后不再手动排空」的理由。
3. 用 `sync.WaitGroup.Go`（Go 1.25+）重写一段旧式 `Add`/`Done` 代码，跑 `go vet ./...` 观察 `waitgroup`（1.25）/`waitgroupgo`（1.27）分析器。
4. 故意写三个数据竞争（共享计数、惰性初始化、无同步标志位），用 `go test -race -count=5` 抓出来再逐个修复。
5. 写一个必然泄漏的 goroutine 程序（发送无人接收），抓 `goroutine` profile；若本机为 Go 1.26，加 `GOEXPERIMENT=goroutineleakprofile` 再看 `/debug/pprof/goroutineleak`，对比两者信息差异。
6. 把第 9 节示例改成「首个成功即返回、取消其余」，讨论 cause 如何区分「业务短路」与「超时」。

## 12. 自检清单

- [ ] 我能说清无缓冲与有缓冲 channel 的同步差别，以及缓冲耗尽意味着什么。
- [ ] 所有 channel 关闭都由发送方完成，代码里没有「向已关闭 channel 发送」的路径。
- [ ] 我知道 `nil` channel 的阻塞语义，并在 `select` 里用它禁用分支。
- [ ] 热循环里没有 `time.After`，定时器用 `Stop`/`Reset` 复用。
- [ ] 我能在 `Mutex`、`RWMutex`、`sync/atomic` 之间给出有理由的选择。
- [ ] 所有 `WaitGroup` 用法的 `Add` 都在 `go` 之前，或使用 Go 1.25 的 `WaitGroup.Go`。
- [ ] 每个 `WithCancel`/`WithTimeout`/`WithDeadline` 都有配对的 `cancel`。
- [ ] `context` 一律作为第一个参数传递，没有存进 struct，库里没有自造 `context.Background()`。
- [ ] 长循环与阻塞 IO 都有取消检查路径。
- [ ] 我能列出 `-race` 查不出什么，并知道它只是动态检测。
- [ ] 我能用 goroutine profile（必要时配合 Go 1.26 实验 / 1.27 转正的泄漏 profile）定位泄漏，且每个新增的 `go func()` 都能回答「什么条件下退出」。
## 13. 延伸阅读

- Go 内存模型：<https://go.dev/ref/mem>
- `sync` 包：<https://pkg.go.dev/sync>；`sync/atomic`：<https://pkg.go.dev/sync/atomic>
- `context` 包：<https://pkg.go.dev/context>；`os/signal`：<https://pkg.go.dev/os/signal>
- `time` 包（定时器与 channel 语义）：<https://pkg.go.dev/time>
- `net/http/pprof`：<https://pkg.go.dev/net/http/pprof>
- Go 1.25：<https://go.dev/doc/go1.25>；Go 1.26：<https://go.dev/doc/go1.26>；Go 1.27：<https://go.dev/doc/go1.27>
- 书籍：《Go 程序设计语言》，Alan A. A. Donovan、Brian W. Kernighan，机械工业出版社
- 书籍：《Go 语言并发之道》，Katherine Cox-Buday，中国电力出版社
