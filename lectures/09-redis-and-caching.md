# L09 Redis 与缓存设计

本篇定位：把「缓存怎么用才不出事故」讲清楚。覆盖 Redis 客户端配置、数据结构选型、key 设计、过期与淘汰、穿透/击穿/雪崩、缓存一致性、本地多级缓存与可观测性。
对应周次：W5。前置要求：写过 HTTP 服务与数据库访问层（见 [L08 数据访问](08-database.md)），理解 `context` 取消与超时。
数据结构、淘汰策略等命名属于 Redis 服务端概念；Go 侧只给模块路径，版本以官方最新稳定版为准。
分布式锁的完整实现（可重入、续约、Redlock、fencing token）放在 L19 深入，本篇只给最小要点。

## 1. 客户端选型与连接配置

### 1.1 模块路径

| 项目 | 内容 |
| --- | --- |
| 官方推荐客户端 | `github.com/redis/go-redis/v9`，以官方最新稳定版为准 |
| 官方文档 | <https://pkg.go.dev/github.com/redis/go-redis/v9> |
| 适配版本 | 该模块的主版本路径对应 Redis 6 及以上的服务端能力；具体命令支持范围见官方文档 |

选型原则：只用一个客户端库，不要在同一个进程里混用多套连接管理；客户端必须支持连接池与 `context`，否则超时无法传导。

### 1.2 连接池与超时

| 配置项 | 作用 | 取值建议 |
| --- | --- | --- |
| `DialTimeout` | 建立连接的超时 | 数百毫秒级；跨可用区或跨机房适当放宽 |
| `ReadTimeout` | 单次读命令超时 | 与下游业务超时对齐，通常 100–500ms |
| `WriteTimeout` | 单次写命令超时 | 同上 |
| `PoolSize` | 连接池大小 | 按并发请求数与单次命令耗时估算，不是越大越好 |

其余与池相关的配置项（空闲连接下限、池满时的等待行为、最大重试次数、重试退避等）字段名与语义以官方文档为准。取值方法与数据库连接池一致：先估算「并发命令数 × 平均命令耗时」，再乘以安全系数，并保证 `进程数 × PoolSize` 不超过服务端可接受的连接数（Redis 的连接在服务端同样是资源）。

超时踩坑顺序：客户端 `ReadTimeout` 大于调用方 `ctx` 截止时间时，`ctx` 先失败，连接会被取消并可能被丢弃；反之客户端先超时，调用方还能拿到明确的错误。两者应保持「客户端略小于上游」。

### 1.3 使用注意

- 所有命令都传 `ctx`，Redis 调用同样是 I/O。
- 不要用 `KEYS`、`FLUSHALL`、`FLUSHDB` 之类的全库/全键命令做在线业务逻辑；扫描用具游标语义的迭代命令（命令名见 Redis 官方文档）。
- 批量操作用管道（pipeline）或原生多键命令降低往返，注意单次批量的大小上限，避免一次网络往返携带 MB 级数据。
- 逻辑上互相依赖的命令（读-改-写）需要保证原子性时，优先用单条命令或服务端脚本（Lua），不要用「客户端加锁 + 多次命令」模拟。
- 连接池耗尽的典型信号是命令排队时间上升而 Redis 服务端 CPU 不高，先看客户端指标再看服务端。

## 2. 核心数据结构与后端用途

| 结构 | 典型后端用途 | 注意事项 |
| --- | --- | --- |
| String | 计数器、缓存 JSON、分布式锁的 value、限流计数 | 单键体积要控制；大 value 会拉长网络与序列化时间 |
| Hash | 对象的局部字段更新（如用户资料的某个属性） | 字段多且不再需要时要么设 TTL 要么整体删除，否则会残留 |
| List | 简单队列、最新 N 条列表 | 无消费确认语义，消费者崩溃会丢消息；只适合可丢场景 |
| Set | 去重、标签、共同好友类集合运算 | 大集合的集合运算会在服务端阻塞，注意元素规模 |
| ZSet | 排行榜、延时队列、按分数范围分页 | 延时队列需配合轮询或到期扫描，本质是「轮询 + 排序」 |
| Bitmap | 签到、活跃标记、布隆过滤器的底层位图 | 偏移量决定内存占用，稀疏大偏移会浪费内存 |
| HyperLogLog | 基数估算（UV、独立 IP） | 结果是估算值，不支持取回原始元素 |
| Stream | 消息流、消费组、带确认的读取 | 见 2.1 的边界说明 |

### 2.1 Stream 与专业 MQ 的边界

| 维度 | Redis Stream | 专业消息队列 |
| --- | --- | --- |
| 持久化保证 | 依赖 Redis 的持久化配置，掉电与主从切换可能丢最近数据 | 多数提供同步刷盘与多副本确认 |
| 堆积能力 | 受内存限制，堆积会挤压缓存本身的内存预算 | 磁盘级堆积，容量与内存解耦 |
| 投递语义 | 支持消费组与确认，但重试与死信需要自己实现 | 内置重试、死信、延迟、事务消息 |
| 生态 | 无独立的运维工具链与监控体系 | 有成熟的运维、监控、权限体系 |
| 适用 | 与缓存同源、量小、允许少量重复或丢失的内部事件 | 订单、支付等不可丢的业务消息 |

判断标准：如果消息丢失需要人工补偿，就不用 Stream。

## 3. key 设计规范

| 规范 | 做法 | 理由 |
| --- | --- | --- |
| 命名空间分层 | 用冒号分层，如 `app:user:profile:10086`、`app:article:detail:42` | 便于按前缀排查、批量治理与权限划分 |
| 长度控制 | 前缀用固定短标识，不要塞入完整 URL 或长参数 | 每个 key 都有额外内存与管理开销 |
| 可读性 | 名称体现业务含义与版本，如 `v1` 段 | 结构变更时可平滑切换而不冲突 |
| 编码 | 只使用 ASCII 可见字符，避免空白与需要转义的内容 | 便于日志、命令行工具与监控检索 |
| 热 key | 单个 key 的 QPS 过高时做前缀分片（`app:hot:0..N`） | 单分片是单线程处理，热 key 会打满单个实例 |
| big key | 单 value 控制在数十 KB 量级，集合元素数量设上限 | 大 value 阻塞事件循环、放大网络与序列化开销、影响主从同步 |
| 序列化 | 小对象用 JSON 便于排查；字段多、体积敏感、跨语言强约束用 protobuf | JSON 可读但冗余，protobuf 紧凑但对调试不友好 |
| TTL | 所有缓存 key 必须显式设置过期时间 | 没有 TTL 的缓存会随业务增长无限膨胀，且永远无法自愈 |
| 禁止项 | 不要把用户隐私与凭据明文写入 key 或 value | 缓存常被全量 dump、日志采集与备份，等于二次泄露 |

序列化选型的补充：JSON 在上线期便于直接查看 value 排查问题；当 value 超过数十 KB 或需要跨语言强类型契约时换 protobuf。无论选哪种，都要在 value 里带上版本号或结构标识，避免结构变更后旧数据被错误反序列化。

## 4. 过期与淘汰

### 4.1 过期命令与删除策略

`EXPIRE` / `PEXPIRE` 给已存在的 key 设置剩余生存时间（秒 / 毫秒），`EXPIREAT` / `PEXPIREAT` 设置绝对到期时间。设置 key 时也可以直接带上过期参数，避免出现「写入成功、设置 TTL 失败」的中间状态。

Redis 的过期删除是**惰性删除 + 定期删除**的组合：访问 key 时发现已过期就删除；同时后台周期性抽样删除。因此「已过期」不等于「内存已释放」，内存统计里的 expired 与实际占用会有一段时间差。

### 4.2 maxmemory-policy 取舍

内存达到 `maxmemory` 后，淘汰策略决定行为：

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| `noeviction` | 拒绝写入并报错 | 把 Redis 当持久化存储、绝不能丢数据 |
| `allkeys-lru` | 在所有 key 中按最近最少使用淘汰 | 纯缓存实例，最常用 |
| `volatile-lru` | 只在设置了 TTL 的 key 中淘汰 | 同一实例混有缓存与数据，需要保护数据 |
| `allkeys-lfu` / `volatile-lfu` | 按访问频率淘汰 | 有明显热点、访问分布长尾的场景 |
| `allkeys-random` / `volatile-random` | 随机淘汰 | 访问分布均匀、无热点 |
| `volatile-ttl` | 优先淘汰剩余生存时间短的 key | 希望尽快腾出即将过期的空间 |

选择原则：缓存实例用 `allkeys-lru`；一旦某些 key 不能丢，就拆成两个实例，而不是在同一个实例上用 `volatile-*` 反复权衡。淘汰策略是**兜底**，不是容量规划：持续淘汰说明内存不足，应该扩内存或降数据量。

### 4.3 TTL 抖动

同一批数据在同一时刻写入、TTL 又完全相同，就会在同一时刻集中过期，把压力瞬间全部推给数据库。做法是在基础 TTL 上叠加随机偏移：

```go
// jitter 返回基础 TTL 上叠加 ±10% 随机偏移后的时长。
func jitter(ttl time.Duration) time.Duration {
	delta := float64(ttl) * 0.1
	return ttl + time.Duration((rand.Float64()*2-1)*delta)
}
```

抖动只解决「同时失效」，不解决「同时被大量请求」；热点数据的过期时间还应错开到不同的时间窗。

## 5. 三大缓存问题

| 问题 | 判定条件 | 后果 | 处置 |
| --- | --- | --- | --- |
| 穿透 | 查询的 key 在缓存与数据库中都不存在，每次请求都落库 | 大量不存在的 key 打满数据库 | 布隆过滤器前置过滤；空值缓存并设短 TTL |
| 击穿 | 单个热点 key 过期瞬间，大量并发请求同时回源 | 数据库瞬时被打满 | 互斥重建（`singleflight`）或逻辑过期 |
| 雪崩 | 大批 key 同时过期，或缓存实例整体不可用 | 数据库被全量流量打垮 | TTL 抖动、多级缓存、限流与降级 |

### 5.1 穿透

空值缓存要设**短** TTL（如 30–60 秒），否则数据后来被创建时，客户端会持续读到「不存在」。空值缓存还会带来内存占用问题：被恶意构造的海量随机 key 会挤占内存，因此空值缓存要与「按 key 维度的请求频率限制」一起用。

布隆过滤器的定位是「大概率不存在」的判定器：判定不存在时一定不存在，判定存在时可能误判。所以它只能挡掉无意义的查询，不能替代回源逻辑。

### 5.2 击穿

互斥重建的骨架是用进程内的请求合并（`golang.org/x/sync/singleflight`）或分布式互斥，保证同一个 key 只有一个请求回源，其余请求等待并复用结果：

```go
v, err, shared := sf.Do(key, func() (any, error) {
	// 只放一个 goroutine 进来回源，并回写缓存
	return loadAndCache(ctx, key)
})
```

变体是「逻辑过期」：缓存里存 `{data, expireAt}` 且不设物理 TTL，发现逻辑过期时先返回旧值，同时异步触发一个重建任务。它牺牲短暂的一致性来换取零等待，适合对延迟敏感、可接受旧数据的读接口。

### 5.3 雪崩

三种触发路径与对应手段：TTL 同时到期——加抖动并错峰预加载；缓存集群故障——本地缓存兜住热点读、对回源做限流与熔断；突发热点——提前扩容并对单 key 做读请求的本地合并。

降级要有明确语义：缓存不可用时是「直接返回错误」还是「直连数据库但限流」。默认选择后者时要限制并发数，否则缓存层一挂，数据库会在几秒内被打垮。

## 6. 缓存一致性

### 6.1 Cache-Aside 标准流程

读：先查缓存，命中返回；未命中则查数据库，把结果写入缓存并返回（要处理「数据库也没有」的空值缓存分支）。
写：先写数据库，写成功后**删除**缓存，而不是更新缓存。

删除而不是更新缓存的原因：更新缓存需要把数据库的最新行重新序列化，并发写时两个请求的写入顺序可能与数据库提交顺序相反，留下长期脏数据；删除只做一件事，最终必然回到正确状态。

### 6.2 更新顺序对比

| 方案 | 并发风险 | 结论 |
| --- | --- | --- |
| 先更新数据库，再删缓存 | 在「删缓存成功前的读」可能读到旧值，窗口极小 | 推荐，读多写少场景下不一致窗口最小 |
| 先删缓存，再更新数据库 | 删除后到更新提交之间，读请求会把旧值写回缓存，且长期存留 | 不推荐作为默认方案 |
| 延迟双删 | 更新数据库前后各删一次，第二次延迟执行 | 用于缓解「先删再更新」的脏数据窗口，代价是多一次删除与一个延迟任务 |
| 订阅 binlog 驱动失效 | 由数据库变更事件触发删缓存，与业务代码解耦 | 适合多写入源的系统，代价是引入链路依赖与延迟 |

### 6.3 最终一致性的可接受边界

- 缓存与数据库之间是**最终一致**，不是强一致；任何「先写库再删缓存」的方案都存在一个极短的不一致窗口。
- 对「读自己的写」有强要求的场景（如刚提交就跳到详情页），不要只靠缓存最终一致，应该在写路径返回最新数据、或在读路径对该用户做短暂的缓存旁路。
- 不要在事务未提交时删缓存：删早了，其他请求会把旧值读回缓存。
- 一致性缺陷的兜底是 TTL：无论一致性逻辑写得多复杂，TTL 都应该存在，它保证系统最终能自愈。

## 7. 分布式锁基础

最小可用做法（要点，完整方案见 L19）：

1. 加锁用一条原子命令：`SET key value NX PX ttl`，`NX` 保证只有第一个请求成功，`PX` 给出自动过期时间。
2. `value` 必须是本次加锁生成的唯一随机值（如请求 ID），用于标识「锁是我的」。
3. 释放锁不能在客户端「先读后删」，必须用服务端脚本（Lua）比较 `value` 相等再删除，保证比较与删除的原子性。
4. TTL 要大于业务最长执行时间，并为「业务未完成但锁已过期」设计兜底：要么续约，要么让被保护的资源自身支持幂等与 fencing token 校验。

常见错误：

| 错误 | 后果 |
| --- | --- |
| 用 `SETNX` + 单独 `EXPIRE` 两步 | 两步之间进程崩溃会导致锁永不过期，死锁 |
| `value` 用固定字符串 | 会误删他人的锁 |
| 释放时不校验 `value` | 业务超时后锁已被他人获取，仍被本请求删除 |
| 锁过期但业务继续执行 | 两个进程同时进入临界区，等于没有锁 |
| 用锁保护跨多个 key 的复杂操作 | 锁只保证互斥，不保证原子性，中间失败仍会留下部分写入 |

## 8. 多级缓存与本地缓存

| 方案 | 适用场景 | 注意 |
| --- | --- | --- |
| `sync.Map` | 读多写少、key 集合基本稳定的进程内映射 | 没有容量上限与淘汰，写入多时性能不如普通 map 加锁 |
| LRU（`container/list` + `map` 手写，见 <https://pkg.go.dev/container/list>） | 需要按容量淘汰的本地缓存 | 自己实现要注意并发保护与淘汰时的正确性 |
| 弱引用缓存 | 希望 GC 在内存紧张时自动回收的缓存对象 | 见下方版本说明 |

本地缓存的通用约束：必须有容量上限或过期时间，否则等价于内存泄漏；多实例部署时，一个实例的本地缓存失效不会通知其他实例，所以本地 TTL 要短（秒级到十几秒），且只用于可容忍短暂不一致的数据。

**Go 1.24** 起标准库提供 `weak` 包，可用于弱引用缓存（对象只被弱引用时，GC 可以回收它，从而让缓存不阻止内存回收）。该包的具体 API 名称与语义以官方文档为准：<https://pkg.go.dev/weak>。

## 9. 可观测性与容量估算

| 指标 | 关注点 | 处置方向 |
| --- | --- | --- |
| 缓存命中率 | 命中数 / (命中数 + 未命中数)，按 key 前缀分组看 | 下降先排查 TTL 过短、key 设计变更、序列化失败 |
| 单命令耗时与慢查询 | 是否存在超过预期的命令，慢查询日志中是否有大 key 操作 | 拆大 key、改批量、换数据结构 |
| 内存与内存碎片率 | 已用内存、碎片率（`INFO` 中的碎片率字段） | 碎片持续偏高需要重启或调整分配器参数 |
| 淘汰次数 | 是否持续发生 key 淘汰 | 说明内存不足，扩容或缩数据 |
| 连接数 | 客户端连接数是否接近服务端上限 | 调小 `PoolSize` 或做连接复用 |
| 主从/持久化延迟 | 复制积压、持久化 fork 造成的延迟毛刺 | 避免大 key 与集中写入 |

容量估算方法：

1. 统计平均 key 长度、平均 value 长度、key 数量，得到原始数据量。
2. 加上 Redis 自身的每 key 开销（通常数十到上百字节）与哈希表扩容的装载因子，按原始数据量的 1.5–2 倍估算。
3. 预留 30% 以上空间给持久化 fork 时的写时复制、碎片与突增。
4. 用压测验证：按真实读写比例与 value 大小压测，观察 P99 延迟随连接数与 `PoolSize` 的变化，找到收益拐点。

只按「数据量」估算是常见的容量事故来源：真正的内存占用往往是最初估算的两到三倍。

## 10. 完整示例

博客文章详情接口的 Cache-Aside：查询 → 命中返回 → 未命中用 `singleflight` 合并回源 → 回写带抖动 TTL → 更新时删除缓存。示例用接口抽象存储层，便于替换成真实 Redis 客户端。

```go
package cache

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log/slog"
	"math/rand"
	"time"

	"golang.org/x/sync/singleflight"
)

type Article struct {
	ID      int64  `json:"id"`
	Title   string `json:"title"`
	Content string `json:"content"`
}

// Store 抽象缓存读写；真实实现基于 github.com/redis/go-redis/v9。
type Store interface {
	Get(ctx context.Context, key string) ([]byte, error) // key 不存在时返回 ErrMiss
	Set(ctx context.Context, key string, val []byte, ttl time.Duration) error
	Del(ctx context.Context, keys ...string) error
}

var ErrMiss = errors.New("cache: miss")

// Repo 是回源存储（数据库）。
type Repo interface {
	GetArticle(ctx context.Context, id int64) (*Article, error)
	UpdateArticle(ctx context.Context, a *Article) error
}

type Service struct {
	store Store
	repo  Repo
	sf    singleflight.Group
	ttl   time.Duration
}

func articleKey(id int64) string { return fmt.Sprintf("app:v1:article:detail:%d", id) }

func jitter(ttl time.Duration) time.Duration {
	delta := float64(ttl) * 0.1
	return ttl + time.Duration((rand.Float64()*2-1)*delta)
}

// Get 是 Cache-Aside 的读路径。
func (s *Service) Get(ctx context.Context, id int64) (*Article, error) {
	key := articleKey(id)
	if a, ok := s.fromCache(ctx, key); ok {
		return a, nil
	}
	// 未命中：合并同一 key 的并发回源，避免热点 key 被同时穿透到数据库。
	v, err, _ := s.sf.Do(key, func() (any, error) {
		if a, ok := s.fromCache(ctx, key); ok { // 双重检查，等待期间可能已被回填
			return a, nil
		}
		a, err := s.repo.GetArticle(ctx, id)
		if err != nil {
			return nil, err
		}
		payload, err := json.Marshal(a)
		if err != nil {
			return nil, fmt.Errorf("marshal article: %w", err)
		}
		// 回写失败不影响本次返回，只记录日志。
		if err := s.store.Set(ctx, key, payload, jitter(s.ttl)); err != nil {
			slog.WarnContext(ctx, "cache set failed", "key", key, "err", err)
		}
		return a, nil
	})
	if err != nil {
		return nil, err
	}
	return v.(*Article), nil
}

func (s *Service) fromCache(ctx context.Context, key string) (*Article, bool) {
	raw, err := s.store.Get(ctx, key)
	if err != nil {
		if !errors.Is(err, ErrMiss) {
			// 缓存故障降级为回源，但要限流，避免数据库被全量流量打垮。
			slog.WarnContext(ctx, "cache get failed", "key", key, "err", err)
		}
		return nil, false
	}
	var a Article
	if err := json.Unmarshal(raw, &a); err != nil {
		// 反序列化失败说明缓存结构已过期，按未命中处理并删除脏 value。
		_ = s.store.Del(ctx, key)
		return nil, false
	}
	return &a, true
}

// Update 是写路径：先写数据库，成功后再删除缓存。
func (s *Service) Update(ctx context.Context, a *Article) error {
	if err := s.repo.UpdateArticle(ctx, a); err != nil {
		return fmt.Errorf("update article: %w", err)
	}
	if err := s.store.Del(ctx, articleKey(a.ID)); err != nil {
		// 删除失败要告警：在 TTL 到期前会持续读到旧值。
		slog.ErrorContext(ctx, "cache del failed", "id", a.ID, "err", err)
	}
	return nil
}
```

要点回顾：读路径不写「无条件回源」的分支，缓存故障时也只降级一次回源；`singleflight` 只在进程内合并，多实例部署时若热点 key 仍会打穿，需要额外的分布式互斥；删除缓存失败必须告警，因为它是唯一的一致性修复动作。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 缓存 key 不设 TTL | 内存持续增长、数据永远不更新 | 把缓存当数据库 | 所有缓存 key 显式设 TTL |
| 大批 key 同一时刻过期 | 数据库周期性被打满 | TTL 无抖动 | TTL 叠加随机偏移并错峰 |
| 把空结果不缓存 | 不存在的 key 每次都落库 | 未处理穿透 | 空值缓存并设短 TTL，配合布隆过滤器 |
| 回源无合并 | 热点 key 过期瞬间数据库连接被打满 | 未处理击穿 | 用 `singleflight` 或分布式互斥 |
| 写路径更新缓存而非删除 | 并发写后长期脏数据 | 更新顺序不确定 | 先写库再删缓存 |
| 先删缓存再更新数据库 | 旧值被读回并长期驻留 | 删除与提交之间存在窗口 | 先更新数据库再删缓存 |
| 在事务未提交时删缓存 | 其他请求把旧值写回 | 删除时机早于提交 | 事务提交成功后删除 |
| 用 `KEYS` 做线上扫描 | 服务端阻塞、请求超时 | 全键扫描是 O(N) | 用游标迭代或按命名空间维护索引 |
| 单 key 存 MB 级数据 | 延迟毛刺、主从同步滞后 | big key | 拆分结构或减小 value，设置元素上限 |
| 热 key 不做分片 | 单实例 CPU 打满 | 单分片单线程 | 按前缀分片并做本地合并 |
| 用 `SETNX` + `EXPIRE` 两步加锁 | 进程崩溃导致锁永不释放 | 非原子操作 | 单条 `SET ... NX PX` |
| 释放锁不校验唯一值 | 误删他人锁 | 先读后删非原子 | Lua 脚本比较后删除 |
| 缓存故障直接全量回源 | 缓存一挂数据库立刻被打垮 | 无降级与限流 | 降级时限制回源并发并熔断 |

## 12. 动手练习

1. 用 `github.com/redis/go-redis/v9` 实现 `Store` 接口，命中时返回缓存值，未命中时返回 `ErrMiss`，并给所有写入设置带抖动的 TTL。
2. 在本地用一个带统计的假 `Store` 测出读路径命中率，并构造「同一 key 100 个并发请求」的场景，对比有无 `singleflight` 时的回源次数。
3. 把 5.1 的空值缓存实现出来，再写一个测试证明「数据后来被创建时，缓存能在短 TTL 内自动纠正」。
4. 用 `EXPIRE` 给一个已有 key 设置过期时间，分别在过期前后读取并观察返回值与内存统计的变化。
5. 估算你正在做的项目的缓存容量：统计 key 数量与平均 value 长度，按 1.5–2 倍加开销、再预留 30% 余量，与压测结果对比。

## 13. 自检清单

- [ ] 所有缓存 key 都显式设置了 TTL，且热点数据的 TTL 有抖动。
- [ ] key 命名遵循「应用:版本:业务:标识」的分层规范。
- [ ] 单 value 体积与集合元素数量有上限，没有 big key。
- [ ] 热 key 有分片或本地合并方案。
- [ ] 穿透有布隆过滤器或空值缓存，空值 TTL 足够短。
- [ ] 击穿有 `singleflight` 或分布式互斥，且有双重检查。
- [ ] 雪崩有 TTL 抖动、多级缓存与降级限流预案。
- [ ] 写路径统一「先更新数据库、提交成功后再删缓存」，删缓存失败会告警。
- [ ] 淘汰策略与实例用途匹配（缓存实例用 `allkeys-lru`）。
- [ ] 客户端连接池与超时按上游超时设置，所有命令都传 `ctx`。
- [ ] 命中率、慢查询、内存碎片率、淘汰次数都有监控。
- [ ] 分布式锁使用单条 `SET NX PX` 加锁、Lua 校验后释放，TTL 覆盖业务最长耗时。

## 14. 延伸阅读

- Redis 客户端（Go）：<https://pkg.go.dev/github.com/redis/go-redis/v9>
- 请求合并：<https://pkg.go.dev/golang.org/x/sync/singleflight>
- 标准库链表（手写 LRU 用）：<https://pkg.go.dev/container/list>
- Go 1.24 的 `weak` 包：<https://pkg.go.dev/weak>
- Go 1.25 的 `net/http.CrossOriginProtection`（缓存放行时需要区分来源时同样要理解 Fetch metadata）：<https://go.dev/doc/go1.25>
- Redis 官方命令与配置文档：<https://redis.io/docs/latest/>
- 书籍：《数据密集型应用系统设计》，Martin Kleppmann，中国电力出版社
