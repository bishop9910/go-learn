# L19 分布式基础：Raft、分布式锁、一致性哈希与限流

（本篇定位：给出分布式系统里最常被问到的四类机制——共识、锁、分片、限流——的真实边界与可落地实现。对应 W15（限流熔断与网关）与 W22–W24（分布式与一致性专题），前置为 [L16](16-microservices-and-observability.md)、[L17](17-mq-and-kubernetes.md)、[L18](18-runtime-internals.md)。语言基线 Go 1.25，涉及 1.26 / 1.27 的能力逐条标注版本。）

## 1. 现实前提：先把假设摆正

| 前提 | 具体含义 | 工程后果 |
| --- | --- | --- |
| 网络不可靠 | 延迟无上界、会丢包、会重复、会乱序，且无法区分「慢」与「已死」 | 必须设超时；超时后只能选择重试（可能重复）或放弃（可能丢失） |
| 节点会失败 | 进程崩溃、机器宕机、磁盘损坏，且失败可能同时发生 | 需要副本与多数派；单点组件最终都会成为故障源 |
| 时钟不可信 | 各机器时钟有偏移，还会回拨 | 不能用本地时间戳排序事件，跨节点因果要靠逻辑时钟或版本号 |
| 部分失败 | 请求可能已到达对端并生效，只是响应丢了 | 所有写操作都要可重试且幂等 |

CAP 的准确表述：**当网络分区发生时**，系统只能在一致性（线性一致性）与可用性之间取舍；没有分区时两者都能满足，所以「三选二」的口号是误导。PACELC 补充了另一半：**P**artition 时在 **A**vailability 与 **C**onsistency 之间取舍，**E**lse（无分区时）在 **L**atency 与 **C**onsistency 之间取舍。这解释了为什么同为「强一致」的存储，跨机房部署的写延迟必然高于单机房——一致性不是免费的，它用延迟和可用性买单。

## 2. 共识与 Raft

为什么需要共识：多个副本必须对「同一位置的日志内容」达成一致，否则不同副本会给出不同的读取结果。共识算法的目标是——只要多数派存活，就能选出唯一主、复制日志，并在任一副本恢复后把日志补齐到一致状态。

| 机制 | 关键点 |
| --- | --- |
| 任期（term） | 单调递增的逻辑时钟，用于识别过期请求与防止脑裂 |
| Leader 选举 | 候选人自增 term 并发起投票；获得多数派（N/2+1）票即当选；每个 term 内每个节点最多投一票 |
| 日志复制 | Leader 通过 `AppendEntries` 追加日志（同时充当心跳）；多数派确认后推进 `commitIndex` |
| 提交与匹配位置 | `commitIndex` 是已提交位置（多数派已复制），`matchIndex` 记录每个跟随者已复制的最高位置，用于安全推进提交 |
| 选举限制 | 候选人日志必须至少和自己一样新（term 更大，或 term 相同但索引不小于自己），否则拒绝投票——这是「已提交的日志不会被覆盖」的关键约束 |
| 快照 | 日志无限增长会拖慢重启与复制，用快照压缩已提交前缀 |
| 成员变更 | 通过单步或联合共识（joint consensus）切换配置，避免出现两个不相交的多数派 |

Raft 与 Paxos / ZAB 的关系：Raft 与 Paxos 解决的是同一类问题（多数派共识），Raft 的设计目标是**可理解性与工程可实现性**，把「选主 + 日志复制 + 安全性约束」拆成独立且易于论证的模块；ZAB（ZooKeeper 使用）着眼点是主备系统的原子广播，强调「主备顺序一致的重放」。三者都依赖多数派，区别主要在表述与工程细节，不必把它们的差异当成能力差异。

`etcd` 用 Raft 做什么：把「键值写操作」作为日志条目复制到多数派，提交后再应用到状态机（键值存储），因此读可以通过 Raft 日志顺序保证线性一致，也可以显式走只读路径换取吞吐；watch 机制建立在同一个已提交日志序列之上，这正是「watch 不会丢事件、不会乱序」的原因。

### 2.1 单机内存版 Raft 骨架

```go
type Role int

const (
	Follower Role = iota
	Candidate
	Leader
)

type LogEntry struct {
	Term    int
	Index   int
	Command []byte
}

type Raft struct {
	mu        sync.Mutex
	id        int
	peers     []int
	role      Role
	currentTerm int
	votedFor  int
	log       []LogEntry // log[0] 为哨兵，简化索引计算

	commitIndex int
	lastApplied int
	nextIndex   map[int]int // 仅 Leader 维护：下一个要发给该跟随者的索引
	matchIndex  map[int]int // 仅 Leader 维护：已确认复制的最高索引

	electionTimeout  time.Duration // 随机化，避免同时竞选
	heartbeatTimeout time.Duration
}

// 选举循环：随机超时触发竞选，拿到多数票即当选，收到更大 term 则退回跟随者
func (r *Raft) electionLoop() {
	for {
		time.Sleep(randomTimeout(r.electionTimeout)) // 每轮重新随机：随机化是关键
		r.mu.Lock()
		if r.role == Leader {
			r.mu.Unlock()
			continue
		}
		r.role = Candidate
		r.currentTerm++
		term := r.currentTerm
		r.votedFor = r.id
		lastIndex, lastTerm := r.lastLogInfo()
		r.mu.Unlock()

		votes := 1
		for _, peer := range r.peers {
			if peer == r.id {
				continue
			}
			// 伪代码：RequestVote(term, candidateID, lastIndex, lastTerm)
			// 对端仅当「term 未过期」且「候选人日志至少和自己一样新」时投票
			if granted, peerTerm := requestVote(peer, term, r.id, lastIndex, lastTerm); granted {
				votes++
			} else if peerTerm > term {
				r.stepDown(peerTerm)
				break
			}
		}
		if votes > (len(r.peers)+1)/2 {
			r.mu.Lock()
			if r.currentTerm == term { // 期间没被更高 term 打断
				r.role = Leader
				for _, peer := range r.peers {
					r.nextIndex[peer] = lastIndex + 1
					r.matchIndex[peer] = 0
				}
			}
			r.mu.Unlock()
		}
	}
}

// 日志复制（Leader 侧）：
// 1) 心跳周期内对每个跟随者发送 prevIndex/prevTerm + entries
// 2) 跟随者校验 prevIndex/prevTerm 不匹配则拒绝，Leader 递减 nextIndex 重试（日志回溯补齐）
// 3) 跟随者追加后回复成功，Leader 更新 matchIndex
// 4) 对 matchIndex 排序取多数派位置 N，且 log[N].Term == currentTerm，则 commitIndex = N
func (r *Raft) advanceCommit() {
	r.mu.Lock()
	defer r.mu.Unlock()
	matches := make([]int, 0, len(r.peers))
	for _, p := range r.peers {
		matches = append(matches, r.matchIndex[p])
	}
	slices.Sort(matches)
	median := matches[len(matches)/2] // 多数派已复制的最高位置
	if median > r.commitIndex && r.log[median].Term == r.currentTerm {
		r.commitIndex = median
	}
}
```

**注意**：上面的「取中位数」只在节点数固定为奇数时直观成立，真实实现需要更严谨的多数派判定；`slices` 包见 <https://pkg.go.dev/slices>。这段代码是结构示意，不是可直接上线的实现——生产环境应使用经过验证的库（`go.etcd.io/raft/v3`，以官方最新稳定版为准）或直接用 `etcd`。

测试方法：把 `requestVote` / `AppendEntries` 抽成接口，用可控的网络模拟器替换——支持「丢弃指定节点间的消息」「延迟若干毫秒」「分区成两组」；所有超时都用 `testing/synctest`（1.25 转正，`synctest.Test` / `synctest.Wait`）或注入的时钟让测试可确定地推进。必测场景：分区后旧 Leader 收到写请求不得提交、恢复后日志能被覆盖为正确内容、随机化超时能保证活跃性（反复重跑不出现「选不出主」）。

## 3. 分布式锁

基于 Redis 的最小实现：`SET key value NX PX ttl`——`NX` 保证只有第一个请求能创建，`PX` 保证即使持有者崩溃锁也会过期，value 用唯一随机值（如请求 id）以便安全释放。释放必须原子：先比对 value 再删除，用 Lua 脚本在服务端一次完成（先 `GET` 再 `DEL` 会出现「检查通过后被别人拿到锁、然后删掉别人的锁」的窗口）。Go 客户端模块路径 `github.com/redis/go-redis`（以官方最新稳定版为准）。

```lua
-- 释放锁：只有持有者才能删除
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

续期与毛刺：业务执行时间可能超过 TTL，因此需要 watchdog 定期续期。矛盾在于——**不续期则锁可能提前过期导致并发进入，续期则在持有者假死（如长时间 GC）时让锁永远不释放**。工程折中：TTL 取「P99 业务耗时 × 若干倍」，续期只在业务确认仍在推进时进行，并对「续期失败」定义明确行为（通常是尽快中止业务并释放资源）。

Redis 主从切换导致的锁失效：写入在 master 上成功但尚未同步到 replica 时 master 宕机，新 master 上锁不存在，第二个持有者可以拿到同一把锁。Redlock 试图通过「多数派独立 Redis 实例」缓解，但**该方案是否真的提供安全性在业界存在长期争议**（有观点认为它依赖了不可靠的时钟假设，与基于租约的共识算法不等价）。这里如实陈述争议存在，不给出「一定安全」或「一定不安全」的绝对结论；实践建议是：对安全性要求高的场景使用基于共识的锁（`etcd` 的 lease）或直接设计成幂等，而不是依赖 Redis 锁。

`etcd` 的锁语义更完整：客户端申请一个租约（lease）并绑定一个键，键存在即持有锁；释放时撤销租约或删除键；等待者通过 watch 前一个 revision 的删除事件排队（避免惊群）；持有期间需要定期续租，否则租约到期锁自动失效。与 Redis 相比，`etcd` 的锁建立在共识之上，不会因为主从切换而凭空消失；代价是延迟更高、依赖更重。

「锁」与「幂等」的关系：**能靠幂等解决就不要用锁**。锁解决的是「同一时刻只能有一个执行者」，幂等解决的是「重复执行不产生额外副作用」。若业务本身可以设计成幂等（条件更新、唯一索引、状态机），就不需要引入锁及其全部故障模式。

## 4. 一致性哈希

普通取模 `hash(key) % N` 在节点数变化时几乎全量重映射：N 从 3 变 4 时，绝大部分键的归属都会改变，缓存全部失效、数据需要大迁移。

哈希环与虚拟节点：把哈希空间看成首尾相接的环，节点（及其多个虚拟副本）按哈希值落在环上，键归属到顺时针方向第一个节点。引入虚拟节点的目的是解决倾斜——物理节点少时，环上的位置分布可能极不均匀；每个物理节点映射成几十到几百个虚拟节点后，负载趋于均匀。

```go
type ConsistentHash struct {
	mu       sync.RWMutex
	replicas int
	ring     []uint32          // 已排序的虚拟节点哈希
	owner    map[uint32]string // 虚拟节点 → 物理节点
}

func New(replicas int) *ConsistentHash {
	return &ConsistentHash{replicas: replicas, owner: make(map[uint32]string)}
}

func (c *ConsistentHash) Add(node string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	for i := 0; i < c.replicas; i++ {
		h := hashKey(fmt.Sprintf("%s#%d", node, i))
		c.ring = append(c.ring, h)
		c.owner[h] = node
	}
	slices.Sort(c.ring) // 二分查找要求有序
}

func (c *ConsistentHash) Remove(node string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	for i := 0; i < c.replicas; i++ {
		h := hashKey(fmt.Sprintf("%s#%d", node, i))
		delete(c.owner, h)
		idx := slices.Index(c.ring, h)
		if idx >= 0 {
			c.ring = slices.Delete(c.ring, idx, idx+1)
		}
	}
}

// Get 返回 key 归属的物理节点：环上顺时针第一个虚拟节点
func (c *ConsistentHash) Get(key string) string {
	c.mu.RLock()
	defer c.mu.RUnlock()
	if len(c.ring) == 0 {
		return ""
	}
	h := hashKey(key)
	// 手写二分：找环上第一个 >= h 的虚拟节点
	lo, hi := 0, len(c.ring)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		if c.ring[mid] < h {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	if lo == len(c.ring) {
		lo = 0 // 回绕到环首
	}
	return c.owner[c.ring[lo]]
}
```

`hashKey` 可用标准库的 `hash/fnv` 或 `hash/crc32`（见 <https://pkg.go.dev/hash>）；`slices` 包见 <https://pkg.go.dev/slices>。

键迁移比例测试：准备 10 万个键，先记录 3 个节点下的归属分布，再加入第 4 个节点，统计归属发生变化的键的比例。理想情况下应约为 `1/(N+1)`（约 25%），而取模方案接近 100%（约 75% 以上）。同时统计各节点的键数量占比，验证最大偏差是否在可接受范围（例如不超过平均值的 ±20%）；若偏差过大，增加虚拟节点数后复测。

## 5. 限流

| 算法 | 突发流量 | 实现要点 | 适用 |
| --- | --- | --- | --- |
| 固定窗口 | 窗口边界处可能通过 2 倍配额 | 每窗口一个计数器，按时间取整分桶 | 实现最简单，对精度要求低 |
| 滑动窗口-日志 | 精确 | 记录每次请求时间戳，剔除窗口外的记录 | 单机低 QPS 场景，内存随请求量增长 |
| 滑动窗口-计数 | 近似精确 | 当前窗口计数 + 上一窗口计数按比例加权 | 分布式限流的主流做法，内存固定 |
| 漏桶 | 不允许突发，输出恒定 | 队列 + 固定速率出队，队列满则拒绝 | 保护下游恒定处理能力 |
| 令牌桶 | **允许突发**（桶中有存量） | 按速率补充令牌，取到令牌才放行 | 通用 API 限流，最常用 |

```go
// 令牌桶骨架：容量 burst，按 rate 个/秒补充
type TokenBucket struct {
	mu       sync.Mutex
	capacity float64
	tokens   float64
	rate     float64 // 每秒补充
	last     time.Time
}

func (b *TokenBucket) Allow() bool {
	b.mu.Lock()
	defer b.mu.Unlock()
	now := time.Now()
	b.tokens = min(b.capacity, b.tokens+now.Sub(b.last).Seconds()*b.rate)
	b.last = now
	if b.tokens >= 1 {
		b.tokens--
		return true
	}
	return false
}

// 固定窗口：临界问题的来源——窗口边界两侧各放行一次配额
type FixedWindow struct {
	mu     sync.Mutex
	limit  int
	count  int
	window time.Time
}

func (w *FixedWindow) Allow() bool {
	w.mu.Lock()
	defer w.mu.Unlock()
	now := time.Now().Truncate(time.Second) // 按秒分桶
	if now.After(w.window) {
		w.window, w.count = now, 0
	}
	if w.count >= w.limit {
		return false
	}
	w.count++
	return true
}
```

固定窗口的临界问题：limit 为 100/s 时，第 0.99 秒来 100 个请求、第 1.01 秒再来 100 个请求，两秒内的瞬时速率达到 200/s，而两个窗口各自都没超限。滑动窗口计数用「上一窗口计数 × 剩余比例 + 当前窗口计数」逼近真实速率。

分布式限流：把令牌桶状态放进 Redis，用 Lua 脚本在服务端原子地「读取 — 按时间差补充 — 尝试扣减」，避免「读-改-写」竞态；再配合本地令牌桶做两级限流——**本地桶按实例数分摊总配额并承担突发吸收**（不依赖网络、无额外延迟），Redis 桶作为全局精确上限（有网络开销，放在关键入口）。只做本地限流会把总量放大到 `实例数 × 单实例配额`；只做分布式限流则每个请求都要访问 Redis，Redis 抖动会直接变成业务故障。

| 手段 | 目的 | 触发条件 | 失败时的行为 |
| --- | --- | --- | --- |
| 限流 | 控制进入系统的请求速率 | 超过配额 | 返回 429，快速失败 |
| 熔断 | 停止调用已故障的依赖 | 依赖错误率/超时率超阈值 | 直接走降级逻辑，不再发起调用 |
| 降级 | 牺牲非核心功能保住核心链路 | 资源紧张或依赖不可用 | 返回缓存/默认值/精简结果 |
| 隔离 | 限制故障影响范围 | 按资源池或并发数划分 | 只有某个池被拖满，其他池不受影响 |

## 6. 熔断器

三态机：**关闭**（正常放行，同时统计）→ 错误率或超时率达到阈值 → **打开**（直接失败，不发请求）→ 经过冷却期 → **半开**（放行少量试探请求）→ 试探成功则回到关闭，失败则重新打开并延长冷却。

滑动窗口统计：用「按时间分桶的计数环」而不是全量日志，内存固定；至少统计请求数、失败数、超时数，并按最小请求量门槛避免低流量下的误判（1 个请求失败就是 100% 错误率）。

与重试的相互作用是设计重点：**重试会放大故障**。依赖已经过载时，重试把请求量乘以重试次数，雪上加霜。正确组合是：重试只针对幂等且可恢复的错误（连接失败、明确的可重试错误码），带指数退避与抖动，并设置总重试预算（如请求量的 10%，用令牌桶实现）；熔断在依赖持续失败时直接切断，而不是让重试继续冲击。另外必须避免「重试风暴的自我强化」：重试成功率的提高会让熔断器误以为依赖恢复，从而放行更多流量。

## 7. 分布式事务与一致性机制

| 机制 | 思路 | 主要问题 |
| --- | --- | --- |
| 2PC | 协调者先准备、再提交，参与者持有锁直到决议 | 协调者故障导致参与者长期阻塞；准备阶段锁住资源，吞吐差 |
| 3PC | 在准备与提交之间插入预提交，减少阻塞窗口 | 仍然无法完全避免不一致，且多一轮通信 |
| TCC | 每个操作实现 try / confirm / cancel 三段 | 业务改造成本高，cancel 必须幂等，空回滚与悬挂要处理 |
| Saga | 长事务拆成一串本地事务 + 补偿事务 | 无隔离性，中间状态对外可见；补偿必须幂等 |
| 本地消息表 / Outbox | 业务表与消息表同事务写入，中继投递 | 依赖中继的至少一次语义 + 消费端幂等（见 [L17](17-mq-and-kubernetes.md)） |
| 可靠事件 | 事件存储作为事实来源，下游按事件重放构建视图 | 事件 schema 演进与重放成本 |

只描述机制与取舍，不引入具体框架。实践中优先级是：**能用单库事务就不用分布式事务 → 能接受最终一致就用 Outbox + 幂等 → 必须强一致时用共识系统（如 `etcd`）承载关键状态 → 其余情况才考虑 TCC/Saga**。

## 8. 分布式 ID

| 方案 | 单调性 | 唯一性来源 | 问题与适用 |
| --- | --- | --- | --- |
| 数据库自增 | 单调递增 | 单点数据库 | 简单但单点瓶颈；分库分表后需要步长或号段 |
| UUID | 无序 | 随机数（碰撞概率极低） | 无需协调；但无序导致索引插入随机、体积大。**Go 1.27 起标准库新增 `uuid` 包**用于生成与解析，具体函数名以 <https://pkg.go.dev/uuid> 为准 |
| 雪花算法 | 趋势递增 | 时间戳 + 机器号 + 序列号 | 性能好、有序；强依赖时钟，**时钟回拨**会造成重复 ID，必须显式处理 |
| 号段模式 | 趋势递增 | 数据库批量发号 + 内存自增 | 折中方案：数据库压力降低几个数量级，重启会浪费一段号；需要处理多实例号段不重叠 |

时钟回拨的处理策略：回拨幅度小则自旋等待追平（设置最大等待上限）；回拨幅度大则拒绝发号并告警，或用「备用机器号/更高位版本号」切换工作位；绝不能忽略回拨继续用旧时间戳发号。运维上应保证机器时间同步（NTP 且监控偏移），并把回拨事件作为告警。

## 9. 高并发瓶颈排查方法论

分层定位，逐层排除：

| 层 | 先看什么 | 常见瓶颈 | 对应手段 |
| --- | --- | --- | --- |
| 入口（LB / 网关） | QPS、连接数、TLS 握手耗时、带宽 | 连接数上限、证书握手、单实例带宽打满 | 连接复用、HTTP/2、横向扩容、就近接入 |
| 应用 | CPU 使用率与 throttling、goroutine 数、GC 停顿、锁竞争 | P 不足、GC 频繁、锁粒度过粗、同步阻塞调用 | 调 `GOMAXPROCS`/`GOMEMLIMIT`、减少分配、细分锁、异步化 |
| 依赖（RPC / 缓存 / MQ） | 各依赖的 P99、错误率、重试次数、连接池占用 | 单个依赖拖尾、连接池过小、重试风暴 | 超时与熔断、连接池调优、限流重试预算 |
| 存储 | 慢查询、锁等待、连接数、磁盘 IO、主从延迟 | 缺索引、大事务、热点行、全表扫描 | 索引与 SQL 改写、分片、读写分离、批量合并 |

排查顺序建议：先确认「是不是所有请求都慢」（全局问题）还是「只有某类请求慢」（局部问题）；再看依赖延迟与自身 CPU 的对应关系——自身 CPU 不高而延迟高，通常是等待（依赖、锁、连接池）；自身 CPU 打满则先看分配与 GC。任何结论都要有一个可复现的度量支撑，避免凭感觉加缓存。

## 10. 参考项目思路

| 项目 | 关键设计点 | 容易做错的地方 |
| --- | --- | --- |
| 分布式缓存（GeeCache 类） | 一致性哈希分片 + 本地 LRU + 单飞（singleflight）防击穿 + 节点间回源 | 忘了防击穿导致缓存失效瞬间数据库被打穿；一致性哈希不加虚拟节点导致倾斜 |
| 简化 RPC | 编解码 + 服务注册发现 + 负载均衡 + 超时与重试 + 拦截器 | 只做序列化不做超时与取消传递，链路无法止损 |
| 秒杀系统 | 库存预扣减放在缓存/Redis 用 Lua 原子执行；下单异步化写入 MQ，前端轮询结果；多层限流（本地 + 网关 + 分布式）；防超卖靠条件更新或原子扣减 | 用「先查库存再扣减」造成超卖；同步下单让数据库承受全部瞬时流量；没有对同一用户的重复提交做幂等 |

库存扣减的原子性是核心：`DECR`（Lua 中「读 + 判断 + 扣减」）或 SQL 的 `UPDATE ... WHERE available >= n`，两者都靠「判断与修改在同一个原子操作内」避免超卖；异步下单意味着「扣减成功」与「订单创建成功」不是同一事务，必须允许失败并回补库存。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 把 CAP 简化成「三选二」 | 设计讨论失去依据 | 忽略了分区是前提、无分区时无取舍 | 用 CAP + PACELC 表述，明确分区时与常态下的取舍 |
| 用本地时间戳排序跨节点事件 | 事件顺序错乱、偶发丢更新 | 时钟偏移与回拨 | 用逻辑时钟、版本号或共识系统的 revision |
| 所有超时都用同一个固定值 | 有的依赖频繁超时，有的拖很久 | 未按依赖的延迟分布设定 | 超时按各依赖的 P99 设定，并逐层递减 |
| 超时后无限重试 | 故障期间请求量倍增 | 重试没有预算与上限 | 指数退避 + 抖动 + 重试预算 + 熔断 |
| 用 Redis 锁保护关键资金操作 | 主从切换后出现双写 | 锁在故障切换中失效 | 关键路径设计成幂等，或用共识系统承载锁 |
| 手写 Raft 直接上线 | 出现已提交日志被覆盖等严重问题 | 正确性论证不足、缺少 fuzz 与分区测试 | 使用经过验证的实现（`go.etcd.io/raft/v3`）或 `etcd` |
| 用随机化超时之外的固定超时做选举 | 反复出现选票瓜分、长时间无主 | 同时竞选导致平票 | 选举超时随机化，并保证心跳间隔远小于它 |
| 一致性哈希不加虚拟节点 | 某个节点承担大部分流量 | 物理节点在环上分布不均 | 每节点几百个虚拟节点，并实测分布偏差 |
| 只做本地限流 | 集群总流量是配额的实例数倍 | 配额未全局核算 | 本地限流分摊 + 分布式限流做全局上限 |
| 只做分布式限流 | Redis 抖动直接放大为业务故障 | 每个请求都依赖 Redis | 两级限流，本地桶承担突发吸收与降级 |
| 熔断与重试互不知情 | 依赖恢复前被重试再次打挂 | 重试与熔断独立配置 | 熔断打开时禁止重试，重试预算独立计量 |
| 雪花算法不处理时钟回拨 | 产生重复 ID，数据错乱 | 依赖单调时钟 | 拒绝发号/切换工作位并告警，同时监控时钟偏移 |
| 用带锁的分布式事务替代幂等设计 | 吞吐低、故障面大 | 把幂等问题当并发问题解 | 优先条件更新与唯一约束 |
| 每条缓存都设相同 TTL | 缓存集中失效造成穿透 | TTL 未加随机抖动 | TTL 加随机偏移，热点数据逻辑过期 |

## 12. 动手练习

1. **Raft 选举与分区**：实现第 2 节的骨架，注入网络模拟器，跑三个场景（正常选主、分区后旧主收到写、恢复后日志收敛），每个场景都重复 100 次并统计「选出主所需的最大轮数」。
2. **随机化超时的必要性**：把选举超时改成固定值，观察并统计平票导致的选举失败率；再改回随机化，对比同样次数下的失败率。
3. **Redis 锁的安全释放**：实现 `SET NX PX` + Lua 释放，写一个测试脚本模拟「业务超时后锁过期、原持有者再去释放」的场景，验证 Lua 脚本不会误删他人锁；去掉 value 比对再跑一次，观察误删。
4. **一致性哈希迁移比例**：用 10 万键、3 个节点、`replicas=1/20/150` 三组参数，分别统计加入第 4 个节点后的键迁移比例与各节点负载最大偏差，画出偏差随虚拟节点数的变化。
5. **限流算法对比**：对固定窗口、滑动窗口计数、令牌桶三种实现打同一组「稳定速率 + 周期性突发」的流量，记录实际放行速率的峰值与平滑度，说明各自在什么业务下更合适。
6. **分布式限流**：把令牌桶搬进 Redis + Lua，跑两个应用实例，验证总放行速率等于全局配额而不是两倍；再手动让 Redis 变慢（如加大延迟），观察本地限流的兜底效果。
7. **熔断与重试**：搭一个「可注入延迟与错误率」的假依赖，分别在「只重试」「只熔断」「两者组合且熔断打开时禁止重试」三种配置下压测，记录依赖侧收到的请求量倍数。
8. **ID 方案对比**：分别用号段模式与雪花算法生成 100 万个 ID，记录 QPS、ID 有序程度（相邻 ID 的差值分布）与索引插入性能差异；再模拟时钟回拨，验证你的处理策略。

## 13. 自检清单

- [ ] 能准确表述 CAP 与 PACELC，不再使用「三选二」的说法。
- [ ] 所有跨节点写操作都幂等，且重试有退避、抖动与预算。
- [ ] 能解释 Raft 的 term、多数派、选举限制各解决什么问题。
- [ ] 知道已提交日志不可被覆盖是由选举限制保证的，并能说明原因。
- [ ] 有网络分区与随机超时的测试，且能重复运行验证活跃性。
- [ ] 分布式锁有唯一 value、原子释放与明确的续期失败行为。
- [ ] 能说明 Redis 锁在主从切换下的失效场景，以及为什么高风险场景不应依赖它。
- [ ] 一致性哈希使用虚拟节点，并实测过迁移比例与负载偏差。
- [ ] 限流是两级结构，本地桶与全局配额的换算关系明确。
- [ ] 熔断的统计有最小请求量门槛，且明确「熔断打开时不重试」。
- [ ] 分布式 ID 有明确的时钟回拨处理策略与告警。
- [ ] 能按「入口 → 应用 → 依赖 → 存储」分层定位瓶颈，并给出对应的度量。
- [ ] 任何一致性方案都能说出它的不一致窗口与补偿方式。

## 14. 延伸阅读

- Raft 论文与可视化讲解：<https://raft.github.io/>
- `etcd` 官方文档（租约、事务、watch）：<https://etcd.io/docs/>
- Redis 分布式锁的官方说明与争议：<https://redis.io/docs/manual/patterns/distributed-locks/>
- 一致性哈希原始论文（Karger 等，1997）：<https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf>
- Google SRE 手册中的过载保护与重试预算章节（书名：Site Reliability Engineering，作者：Betsy Beyer 等，出版社：O'Reilly Media）
- 《Designing Data-Intensive Applications》，作者：Martin Kleppmann，出版社：O'Reilly Media
- Go 官方 Release Notes：<https://go.dev/doc/go1.25>、<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>；Go 1.27 新增 `uuid` 包见 <https://pkg.go.dev/uuid>
- 相关讲义：[L16 微服务架构与可观测性](16-microservices-and-observability.md)、[L17 消息队列、容器化与 Kubernetes 部署](17-mq-and-kubernetes.md)、[L18 运行时原理](18-runtime-internals.md)
