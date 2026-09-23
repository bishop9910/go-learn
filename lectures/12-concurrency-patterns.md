# L12 并发模式与并发安全

本篇定位：把并发原语组装成可维护的模式——Worker Pool、Fan-out/Fan-in、Pipeline、背压与限流、单飞去重、超时重试退避、熔断、并发安全的数据结构，最后给一个可运行的并发爬虫。对应 W7–W8（阶段三）。前置：先读 [L11 并发原语与 context](11-concurrency-and-context.md)，本篇默认你已理解 channel 关闭规则、`WaitGroup`、context 传播与 `-race`。

命名约定：标准库与第三方库均只给模块路径与官方链接，函数与方法名以官方文档为准；第三方库一律「以官方最新稳定版为准」，本文不写版本号。

## 1. Worker Pool

任务分发、结果收集、优雅退出的经典三段式：

```go
func Run(ctx context.Context, jobs []Job, workers int) []Result {
	jobCh := make(chan Job)
	resCh := make(chan Result)

	var wg sync.WaitGroup
	for i := 0; i < workers; i++ { // Go 1.25+ 可用 wg.Go 代替 Add/Done
		wg.Go(func() {
			for j := range jobCh { // channel 关闭即退出循环
				select {
				case resCh <- process(j):
				case <-ctx.Done():
					return
				}
			}
		})
	}
	go func() { // 生产者：唯一发送方，负责关闭 jobCh
		defer close(jobCh)
		for _, j := range jobs {
			select {
			case jobCh <- j:
			case <-ctx.Done():
				return
			}
		}
	}()
	go func() { wg.Wait(); close(resCh) }() // 等所有 worker 退出后再关闭 resCh

	results := make([]Result, 0, len(jobs))
	for r := range resCh { // 消费端：resCh 关闭后循环结束
		results = append(results, r)
	}
	return results
}
```

职责划分必须记牢：`close(jobCh)` 由生产者做，`close(resCh)` 由「等所有 worker 退出」的协调者做，消费端只负责 `range`。**Go 1.25** 的 `sync.WaitGroup.Go` 把「`Add(1)` + `go` + `defer Done()`」合并为一次调用，少一类配对错误；旧版本写 `wg.Add(1)` 且必须在 `go` 之前。

| 负载类型 | 建议池大小 | 说明 |
| --- | --- | --- |
| CPU 密集 | 约等于 `GOMAXPROCS` | 再多只会增加调度与缓存抖动 |
| 阻塞式 IO 密集 | `GOMAXPROCS` 的若干倍 | 取决于下游延迟与并发配额，必须实测 |

**Go 1.25** 起运行时的 `GOMAXPROCS` 在 Linux 上会考虑 cgroup 的 CPU bandwidth limit，并周期性更新；可用 `GODEBUG=containermaxprocs=0`、`updatemaxprocs=0` 关闭，也可用 `runtime.SetDefaultGOMAXPROCS` 设置默认值。容器里「看到宿主机核数」的老问题因此改善，但池大小仍应做成配置项并压测，而不是写死 `runtime.NumCPU() * 100`。

## 2. Fan-out / Fan-in

Fan-out 是一任务源分发给多个执行者（上节的 worker 池即是）。Fan-in 是多个生产者汇入一个消费端：

```go
func fanIn(ctx context.Context, chans ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	for _, c := range chans {
		wg.Go(func() {
			for v := range c {
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		})
	}
	go func() { wg.Wait(); close(out) }() // 全部生产者退出后才关闭 out
	return out
}
```

**谁负责关闭 channel**：只有发送方能关闭。多发送方时，先让所有发送方退出（`WaitGroup`），再由一个协调 goroutine 关闭输出；接收方永远不关闭。

避免「向已关闭 channel 发送」的三种设计：

| 设计 | 做法 | 代价 |
| --- | --- | --- |
| 单一关闭者 | 唯一发送方结束时 `close`，其他协作者经它的 channel 转发 | 需要一个汇总环节 |
| 计数后关闭 | `WaitGroup` 等所有发送方退出，协调者再关闭 | 协调 goroutine 必须唯一且存在 |
| 不发往共享 channel | 每个生产者返回自己的 channel，消费端 `select` 聚合 | channel 数量增多，分支需动态构造 |

兜底通知用 `context` 或独立的 `done` channel，而不是「关闭数据 channel」。向已关闭 channel 发送会 panic，无法恢复出有用状态。

## 3. Pipeline

Pipeline 把处理拆成若干阶段，每个阶段是「读入 channel、写出 channel」的函数，因此可以独立测试：喂一个已填充的 channel，断言输出 channel 的内容与关闭时机。

```go
func square(ctx context.Context, in <-chan int) <-chan int { // 一个阶段
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case out <- v * v:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}
```

要点：阶段只依赖自己的输入输出，`defer close(out)` 让下游感知结束；每个阶段的**发送**都必须有 `ctx.Done()` 分支，否则上游取消时下游永久阻塞（泄漏）。没有取消分支的 pipeline 是 goroutine 泄漏的高发区。

错误传播与短路取消：需要「任一阶段出错即整体停止」时用 `golang.org/x/sync/errgroup`（见 <https://pkg.go.dev/golang.org/x/sync/errgroup>，以官方最新稳定版为准）：它的带 context 构造形式返回的 context 会随首个错误取消，把取消信号自动传给所有阶段。也可让阶段返回 `<-chan error` 由调用方统一收口。选择依据是语义：阶段多、要手工控制生命周期时用「错误 channel + context」；任务集合独立、只要「全部成功或快速失败」时用分组库。

## 4. 背压与限流

有缓冲 channel 做队列的局限：容量固定，满则发送方阻塞——这是背压而不是缺点；而**无界队列**在 Go 里只能用无限增长的 slice 模拟，本质是把背压问题转化为内存问题。判断标准是「队列满时想让上游变慢（背压），还是丢弃（限流）」。

信号量模式：带缓冲 channel 当令牌，容量即并发上限——`sem := make(chan struct{}, 4)` 取令牌 `sem <- struct{}{}`，用完 `defer func() { <-sem }()` 释放；忘记释放等于永久丢掉一个并发额度。

| 维度 | 漏桶 | 令牌桶 |
| --- | --- | --- |
| 放行速率 | 恒定 | 平均恒定，令牌可积累 |
| 突发流量 | 平滑掉（排队或丢弃） | 允许短时突发（消耗积攒的令牌） |
| 典型用途 | 保护下游不被突发打垮 | 面向用户的配额、API 限速 |

生产代码直接使用 `golang.org/x/time/rate`（见 <https://pkg.go.dev/golang.org/x/time/rate>，以官方最新稳定版为准），它基于令牌桶模型，支持等待式与拒绝式两种获取方式，具体类型与方法名以官方文档为准。限速器要按维度设计：按客户端、按上游依赖、按全局资源分别限，混在一个限速器里会互相干扰。

## 5. 单飞与去重

同一份昂贵数据被多个并发请求同时需要时，让一个真正回源、其余等待并共享结果，称为单飞。模块路径 `golang.org/x/sync/singleflight`（见 <https://pkg.go.dev/golang.org/x/sync/singleflight>，以官方最新稳定版为准）：按 key 合并并发重复调用，共享同一结果与错误，调用结束后 key 立即释放；函数与方法名以官方文档为准。

| 维度 | `sync.Once`（含 Go 1.21 的 `OnceFunc`/`OnceValue`/`OnceValues`） | 单飞库 |
| --- | --- | --- |
| 作用域 | 进程生命周期内只执行一次 | 同一 key 的**并发窗口内**合并 |
| 是否缓存结果 | 永久保留首次结果 | 不缓存，调用结束即释放 |
| 适用 | 初始化（配置、连接池、注册表） | 缓存击穿保护、回源合并 |

两者可组合：`Once` 负责初始化共享依赖，单飞负责热点 key 的回源合并。另有一层更简单的去重——「已见过的 key 集合」用 `map` + `Mutex` 保护（见第 10 节爬虫的 `seen`），语义是「只处理一次」，与单飞的「合并并发」不同。

## 6. 超时、重试与退避

重试的三条硬性前提：**幂等**（只重试可安全重复的操作，非幂等写重试会产生重复副作用）、**有上限**（最大次数与总时间预算双重限制，退避必须带抖动，否则会同步打垮下游）、**可取消**（退避等待本身要监听 `ctx.Done()`）。

```go
func retry(ctx context.Context, maxAttempts int, base time.Duration, fn func(context.Context) error) error {
	backoff := base
	for attempt := 1; attempt <= maxAttempts; attempt++ {
		if err := ctx.Err(); err != nil {
			return err
		}
		err := fn(ctx)
		switch {
		case err == nil:
			return nil
		case errors.Is(err, ErrPermanent): // 不可重试：立刻返回
			return err
		case attempt == maxAttempts:
			return fmt.Errorf("giving up after %d attempts: %w", attempt, err)
		}
		jitter := backoff / 2 // 抖动：生产代码应从随机源取 [0, backoff)，见 https://pkg.go.dev/math/rand/v2
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(backoff/2 + jitter):
		}
		backoff *= 2 // 指数退避，建议同时设上限
	}
	return nil
}
```
| 错误类别 | 例子 | 处理 |
| --- | --- | --- |
| 可重试 | 连接失败、超时、5xx、限流（429） | 指数退避 + 抖动后重试 |
| 不可重试 | 参数错误、鉴权失败、资源不存在（4xx） | 立即返回，重试只会放大故障 |

与 `context` 的交互：每轮尝试用同一个 `ctx` 或从它派生的短超时 context；父 context 到期后不要「再试最后一次」，那会让上游超时形同虚设。退避总时长必须小于上游预算：上游给 2s、单次调用 1s、退避 1s，就注定超时。

## 7. 熔断

熔断器是一个三态机，针对「被调用方已经在失败」做快速失败，避免把调用方的连接与预算耗在必然失败的请求上。

| 状态 | 进入条件 | 行为 | 退出 |
| --- | --- | --- | --- |
| closed | 初始态 | 放行请求，统计失败率 | 滑动窗口内失败率或连续失败数超阈值 → open |
| open | 失败率超阈值 | 直接拒绝，不发请求 | 冷却时间到期 → half-open |
| half-open | 冷却到期 | 放行少量探测请求 | 探测成功率达阈值 → closed；任一失败 → 立即回 open |

```go
type Breaker struct {
	mu        sync.Mutex
	state     int // 0=closed 1=open 2=halfOpen
	failures  int
	threshold int
	openedAt  time.Time
	cooldown  time.Duration
}

func (b *Breaker) Allow() bool {
	b.mu.Lock()
	defer b.mu.Unlock()
	switch b.state {
	case stateOpen:
		if time.Since(b.openedAt) < b.cooldown {
			return false // 冷却期内快速失败
		}
		b.state = stateHalfOpen // 冷却到期，只放行探测流量
		return true
	case stateHalfOpen:
		return false // 探测期间其余请求仍拒绝，避免雪崩
	default:
		return true
	}
}
```

状态迁移由一次调用结果驱动：配套的 `Report(ok bool)` 在成功时把失败计数清零并回到 `closed`，失败时累加计数，达到阈值即转 `open` 并记录 `openedAt`（判定用滑动窗口或连续失败次数，二者都要设最小样本量，避免低流量下误熔断）。与限流的区别：限流控制**速率**（不论下游是否健康都限制 QPS，保护配额与容量），熔断依据**失败率**（下游已经坏了就不要再打过去）。两者互补：限流防过载，熔断开快速失败、给下游恢复窗口；重试必须配合抖动与上限，否则会抵消熔断效果。第三方实现只给模块路径参考（如 `github.com/sony/gobreaker`，以官方最新稳定版为准），真实项目优先用成熟库或网关侧能力。

## 8. 并发安全的数据结构

**`sync.Map` 的适用与不适用**（见 <https://pkg.go.dev/sync#Map>）：

| 适用 | 不适用 |
| --- | --- |
| 读远多于写 | 写频繁或读写相当 |
| key 集合基本稳定（写一次读多次） | key 频繁增删、需要准确的 `len` |
| 各 key 之间互不相关 | 需要「检查后再写入」的复合原子操作 |
| 只增不删的注册表、元数据缓存 | 需要一致快照遍历的场景 |

普通 `map` + `RWMutex` 在多数场景更简单也更快；`sync.Map` 省的是「读路径不加锁」，代价是内存占用更高、语义更弱（长度与遍历不保证一致视图）。

**分片锁（sharded mutex）**：把一个大 map 拆成 N 个分片、每片一把锁，降低争用。

```go
type ShardedMap struct {
	shards [16]struct {
		mu sync.RWMutex
		m  map[string]int
	}
}

func (s *ShardedMap) shard(key string) *struct {
	mu sync.RWMutex
	m  map[string]int
} {
	h := fnv.New32a() // hash/fnv，见 https://pkg.go.dev/hash/fnv
	_, _ = h.Write([]byte(key))
	return &s.shards[h.Sum32()%uint32(len(s.shards))]
}
```

分片数取 2 的幂便于取模，且要覆盖真实 key 分布；跨分片的操作（如「总量」「长度」）需要额外协调，不要指望分片锁提供全局一致性。

**读写锁 vs 原子操作**：单个字长字段（计数器、标志位、指针）用 `sync/atomic` 的类型化 API；涉及多个字段的一致性用锁。`atomic.Int64`、`atomic.Bool`、`atomic.Pointer` 等类型自 **Go 1.19** 可用。

**配置热更新（copy-on-write + 原子替换）**，用 `atomic.Pointer[T]` 最干净：

```go
type Config struct {
	Timeout   time.Duration
	Endpoints []string // 只读切片，替换配置时整体重建
}
type Store struct{ cur atomic.Pointer[Config] }

func (s *Store) Load() *Config          { return s.cur.Load() }  // 读侧无锁
func (s *Store) Update(next *Config)    { s.cur.Store(next) }    // 写侧整体替换
```

刷新循环（定时构建新配置后 `Update`）用 `time.NewTimer` + `Reset` 而非 `time.After`，理由见 L11 第 3 节。规则：发布出去的 `Config` 必须视为**不可变**（读到的人可能长期持有它，原地改它就会产生数据竞争），更新一律构造新对象整体替换。这与 `sync.Once` 的「一次且永久」完全不同。

## 9. 并发模式选型表

| 场景 | 推荐模式 | 不推荐做法 |
| --- | --- | --- |
| 批量任务、要限制并发 | Worker Pool + 信号量 | 无界 `go func()` |
| 多个数据源汇入一个处理流 | Fan-in（协调者关闭输出） | 多个 goroutine 各自 `close(out)` |
| 多阶段数据处理 | Pipeline，每阶段带取消分支 | 阶段函数忽略 context |
| 上游更快、下游更慢 | 有缓冲 channel 做背压 | 无界队列 / 只放大缓冲 |
| 保护外部依赖配额 | `golang.org/x/time/rate` 令牌桶 | 每个请求各建一个限速器 |
| 热点 key 回源 | 单飞库 + 缓存 | 每个请求都穿透到数据库 |
| 初始化共享依赖 | `sync.Once` / `OnceValue` | 裸 `if inited == nil` 双重检查 |
| 下游大面积故障 | 熔断 + 快速失败 + 抖动重试 | 无限重试、固定间隔重试 |
| 需要一致快照的统计 | `Mutex`/`RWMutex` 保护整个结构 | 用多个原子变量拼一个视图 |

## 10. 综合示例：并发爬虫

需求：并发上限、URL 去重、单请求超时、失败重试（指数退避）、结果聚合、随信号优雅退出。

```go
package crawl

// import 省略：context、errors、fmt、io、net/http、os/signal、sync、time

var (
	ErrRetryable = errors.New("retryable failure")
	ErrPermanent = errors.New("permanent failure")
)

type Result struct {
	URL    string
	Status int
	Body   string
}

type Crawler struct {
	client   *http.Client
	sem      chan struct{} // 并发上限
	attempts int
	backoff  time.Duration
	mu       sync.Mutex
	seen     map[string]struct{} // 去重表
	results  []Result
}

func NewCrawler(concurrency int, client *http.Client) *Crawler {
	return &Crawler{client: client, sem: make(chan struct{}, concurrency),
		attempts: 3, backoff: 200 * time.Millisecond, seen: make(map[string]struct{})}
}

func (c *Crawler) firstVisit(u string) bool { // 去重：只处理第一次见到的 URL
	c.mu.Lock()
	defer c.mu.Unlock()
	if _, ok := c.seen[u]; ok {
		return false
	}
	c.seen[u] = struct{}{}
	return true
}

func (c *Crawler) Crawl(ctx context.Context, seeds []string) ([]Result, error) {
	var wg sync.WaitGroup
	for _, u := range seeds {
		if !c.firstVisit(u) {
			continue
		}
		select { // 先取令牌再启动，避免无界创建 goroutine
		case <-ctx.Done():
			wg.Wait()
			return c.snapshot(), ctx.Err()
		case c.sem <- struct{}{}:
		}
		wg.Go(func() { // Go 1.25+
			defer func() { <-c.sem }() // 释放令牌
			r, err := c.fetch(ctx, u)
			if err != nil {
				return // 失败不写入结果，交由调用方按需统计
			}
			c.mu.Lock()
			c.results = append(c.results, r) // results 由 mu 保护
			c.mu.Unlock()
		})
	}
	wg.Wait() // 等全部任务退出，无 goroutine 泄漏
	return c.snapshot(), ctx.Err()
}

// fetch 复用第 6 节的 retry：错误分类、指数退避、可被 ctx 取消
func (c *Crawler) fetch(ctx context.Context, u string) (Result, error) {
	var out Result
	err := retry(ctx, c.attempts, c.backoff, func(ctx context.Context) error {
		reqCtx, cancel := context.WithTimeout(ctx, 5*time.Second) // 单请求超时
		defer cancel()
		req, err := http.NewRequestWithContext(reqCtx, http.MethodGet, u, nil)
		if err != nil {
			return fmt.Errorf("%w: %v", ErrPermanent, err)
		}
		resp, err := c.client.Do(req)
		if err != nil {
			return fmt.Errorf("%w: %v", ErrRetryable, err) // 连接失败/超时：可重试
		}
		defer resp.Body.Close()
		body, _ := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		switch {
		case resp.StatusCode >= 200 && resp.StatusCode < 300:
			out = Result{URL: u, Status: resp.StatusCode, Body: string(body)}
			return nil
		case resp.StatusCode == 429 || resp.StatusCode >= 500:
			return fmt.Errorf("%w: status %d", ErrRetryable, resp.StatusCode)
		default:
			return fmt.Errorf("%w: status %d", ErrPermanent, resp.StatusCode)
		}
	})
	return out, err
}

func (c *Crawler) snapshot() []Result {
	c.mu.Lock()
	defer c.mu.Unlock()
	out := make([]Result, len(c.results))
	copy(out, c.results) // 返回副本，避免内部 slice 被并发修改
	return out
}

```

优雅退出：入口用 `signal.NotifyContext(context.Background(), os.Interrupt)` 建一个可取消的 ctx 交给 `Crawl`；收到中断后取消传播到全部在途请求，`Crawl` 返回 `ctx.Err()`，库里不要自造 `context.Background()`。

可扩展点（留作练习）：解析页面链接继续入队（去重表要改在「入队时」检查，并按域名限速）；结果写入下游时用另一个 worker 池，避免慢写拖进抓取池。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| goroutine 里直接用循环变量（Go 1.21 及更早，或变量声明在循环外） | 所有任务处理同一个元素 | 闭包捕获同一变量 | 传参或 `v := v`；Go 1.22 起循环变量每轮独立 |
| 双重 `close(ch)` | `close of closed channel` panic | 关闭权没有唯一归属 | 只由发送方关闭，多发送方先 `WaitGroup` 再关 |
| 接收方 `close(ch)` | `send on closed channel` panic | 关闭权归属错误 | 用 `done`/`ctx` 通知，不关数据 channel |
| 用 `time.Sleep` 做同步 | 测试偶发失败、线上偶发错误 | sleep 不建立 happens-before | `channel`、`WaitGroup`、`Once` |
| 无界创建 goroutine | 内存暴涨、OOM | 峰值 goroutine 数等于输入规模 | 信号量或固定 worker 池 |
| `defer` 写在循环里 | 文件描述符/锁堆积到函数返回才释放 | `defer` 只在函数返回时执行 | 抽成小函数，让 `defer` 每轮生效 |
| 忘记释放信号量令牌 | 并发额度耗尽直至永久阻塞 | 缺少 `defer func() { <-sem }()` | 取令牌后立刻 `defer` 释放 |
| 无缓冲 channel 做「通知」 | 发送方阻塞、goroutine 泄漏 | 把交接当广播 | 用 `close(done)` 或 context 广播 |
| 重试不带抖动与上限 | 下游雪崩、故障放大 | 固定间隔同步重试 | 指数退避 + jitter + 最大次数 |
| 重试非幂等写 | 重复下单、重复扣款 | 假定请求最多执行一次 | 幂等键，或先查状态再决定 |
| 原地修改已发布的配置对象 | `-race` 报警 | copy-on-write 被破坏 | 构造新对象 + `atomic.Pointer[T]` 替换 |
| 每个请求新建限速器 | 限速形同虚设 | 限速状态没有共享 | 限速器按依赖/租户维度共享 |

## 12. 动手练习

1. 实现第 1 节的 `Run` 并用 `-race` 验证；再比较固定池、`WaitGroup.Go`、信号量三种写法的可读性与取消行为。
2. 写一个三阶段 pipeline（读文件 → 解析 → 汇总），要求任一阶段出错能取消全部阶段，且不泄漏 goroutine。
3. 给第 6 节的 `retry` 增加抖动与「总预算」参数，并写测试证明 `ctx` 超时时立即返回。
4. 实现第 7 节的熔断器，补「连续 3 次失败进入 open、冷却 100ms 后半开、探测成功回 closed」的状态测试。
5. 用 `map` + `RWMutex`、分片锁、`sync.Map` 三种实现同一份读多写少的缓存，`go test -bench` 比较并解释差异。

## 13. 自检清单

- [ ] 每个池都有明确的并发上限，且令牌在 `defer` 中释放。
- [ ] 每个 channel 的关闭者唯一，且关闭发生在「没有发送方可能再发送」之后。
- [ ] 每个 `go func()` 都有取消分支或明确的退出条件。
- [ ] pipeline 的每个阶段都能独立测试，且可被单独取消。
- [ ] 我知道有界队列与无界队列的差别，背压与丢弃的选择是有意的。
- [ ] 重试有幂等前提、最大次数、指数退避与抖动。
- [ ] 我区分了可重试与不可重试错误，并写进了代码。
- [ ] 熔断、限流、重试的关系说得清楚，不会互相抵消。
- [ ] 共享状态的保护方式（锁/原子/分片）与访问模式匹配。
- [ ] 代码在 `go test -race -count=...` 下干净。

## 14. 延伸阅读

- Go 并发模式（Go Blog）：<https://go.dev/blog/pipelines>
- `sync` 包：<https://pkg.go.dev/sync>；`sync/atomic`：<https://pkg.go.dev/sync/atomic>；`context`：<https://pkg.go.dev/context>
- `golang.org/x/sync`（含 `errgroup`、`singleflight`）：<https://pkg.go.dev/golang.org/x/sync>
- `golang.org/x/time/rate`：<https://pkg.go.dev/golang.org/x/time/rate>
- `hash/fnv`：<https://pkg.go.dev/hash/fnv>；`math/rand/v2`：<https://pkg.go.dev/math/rand/v2>
- Go 1.25 发行说明（容器感知 GOMAXPROCS、`WaitGroup.Go`）：<https://go.dev/doc/go1.25>
- 书籍：《Go 程序设计语言》，Alan A. A. Donovan、Brian W. Kernighan，机械工业出版社；《Go 语言并发之道》，Katherine Cox-Buday，中国电力出版社
