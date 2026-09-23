# L08 数据访问：database/sql、sqlc 与 GORM

本篇定位：把「Go 服务如何可靠地访问关系型数据库」讲透，覆盖 `database/sql` 的抽象模型、连接池调参、超时分层、错误分类与重试、事务边界、迁移策略，以及 `database/sql`、`sqlc`、`GORM` 三者的取舍依据。
对应周次：W3–W5。前置要求：能读懂接口与结构体方法集，用过 `context` 与 `errors.Is`，写过一个基于 `net/http` 的 HTTP 服务。
本文只使用 Go 1.25 基线中确定存在的标准库 API；第三方库只给模块路径，版本以官方最新稳定版为准。

## 1. database/sql 的模型

### 1.1 五个核心类型

| 类型 | 职责 | 生命周期 | 并发安全 |
| --- | --- | --- | --- |
| `DB` | 连接池句柄，不表示某一条具体连接 | 进程级，启动建一次，常驻 | 是 |
| `Stmt` | 预处理语句句柄，绑定在创建它的 `DB` / `Conn` / `Tx` 上 | 可长期复用，随所属对象失效 | 是 |
| `Tx` | 一个事务，独占一条连接 | `BeginTx` 到 `Commit` / `Rollback` | 否 |
| `Row` | 单行结果，只能 `Scan` 一次 | 单次查询内 | 否 |
| `Rows` | 多行结果集，必须关闭 | 迭代期间 | 否 |

三条推论：`DB` 是池不是连接，可在多个 goroutine 间共享，不需要自己加互斥；事务内所有语句必须走 `Tx`，混用 `DB` 会取到另一条连接，读不到未提交数据还可能自锁；`Rows` 不关闭会把连接永久占住，这是生产环境最常见的池耗尽原因。

### 1.2 驱动与模块路径

`database/sql` 只定义抽象，具体方言由驱动实现。用法是 `go get <模块路径>` 后匿名导入，导入只为触发注册。常用驱动模块路径如下，均以官方最新稳定版为准：

| 数据库 | 模块路径 |
| --- | --- |
| MySQL | `github.com/go-sql-driver/mysql` |
| PostgreSQL | `github.com/jackc/pgx` |
| SQLite（纯 Go，无 cgo） | `modernc.org/sqlite` |

驱动注册的字符串名与 `database/sql` 适配子包路径各驱动写法不同，以官方文档为准。

### 1.3 Open 不建连接

`sql.Open` 只校验参数格式并准备好连接池，不发起网络连接，连接在第一次真正执行语句时才建立。所以启动时必须显式做一次 `PingContext`，否则配置错误要到第一个用户请求才暴露；`sql.Open` 返回的 `error` 几乎总是 nil，不能当作可用性判断；关闭服务时调用 `DB.Close`，它阻止新请求并等待在用连接归还。

## 2. 连接池

### 2.1 四个调节旋钮

| 方法 | 作用 | 设错的后果 |
| --- | --- | --- |
| `SetMaxOpenConns` | 同时打开的上限（使用中 + 空闲），0 表示不限 | 不限会被流量打爆数据库；过大则数据库侧排队 |
| `SetMaxIdleConns` | 保持空闲的连接上限 | 过小导致反复建连（TCP + TLS + 认证）；过大浪费连接额度 |
| `SetConnMaxLifetime` | 连接自创建起的最长存活时间 | 设成无限会让被中间件切断的连接继续被复用 |
| `SetConnMaxIdleTime` | 连接空闲多久后被回收 | 过长会保留已被负载均衡掐断的死连接 |

### 2.2 取值方法论

1. 取数据库侧上限：MySQL `SHOW VARIABLES LIKE 'max_connections';`，PostgreSQL `SHOW max_connections;`。
2. 扣掉预留额度：运维、备份、监控、迁移任务、其他服务占用的连接先减掉。
3. 按实例数均摊：`每实例 MaxOpenConns = 可用额度 / 实例数 × 安全系数(0.6–0.8)`。扩容时必须重算，否则实例翻倍就撞上 `too many connections`。
4. 找收益拐点：连接数超过数据库可用 CPU 并行度后，写密集场景吞吐不再增长，延迟反而上升。
5. `MaxIdleConns` 取 `MaxOpenConns` 的 1/4 到 1/2；空闲池太小会出现「每请求建连、用完即关」的锯齿。
6. `ConnMaxLifetime` 必须小于链路上中间件的最长空闲时间（负载均衡、连接代理、云数据库 idle timeout），常用 5–30 分钟；`ConnMaxIdleTime` 略小于中间件的空闲踢出时间，让客户端先主动回收。

### 2.3 三类常见故障

| 故障 | 现象 | 根因 | 处置 |
| --- | --- | --- | --- |
| 连接泄漏 | 使用中连接长期顶满上限，请求排队到超时 | `Rows` 未关闭、事务未 `Rollback`、提前 return 跳过清理 | `defer rows.Close()`；事务 `defer tx.Rollback()` |
| `database is locked` | 写操作报锁等待失败（SQLite 典型） | 写事务持锁期间另一个写事务并发进入，或事务过长 | 缩短事务、串行化写路径、通过 DSN 设 busy timeout（参数名以驱动文档为准） |
| stale connection | 首次使用某连接时立刻报连接重置或 EOF | 中间件已掐断，池里仍是活连接 | 收紧 `ConnMaxLifetime`；对幂等操作做分类重试 |

连接池统计通过 `DB` 暴露的统计读取方法观察（见 <https://pkg.go.dev/database/sql>），排障优先看「使用中连接数」与「等待次数」。

## 3. 上下文与超时

规则只有一条：执行 SQL 一律使用带 `ctx` 的变体——查询多行用 `QueryContext`，单行用 `QueryRowContext`，写入与 DDL 用 `ExecContext`，开事务用 `BeginTx`，预处理用 `PrepareContext`。`ctx` 被取消时会发生三件事：排队等待连接的调用立即返回；已下发的语句向驱动发送取消；该连接可能被标记为不可用并关闭。因此取消不是免费的，高频超时会产生大量建连开销。

超时值分层，外层预算必须大于内层：handler 总预算约 2000ms（单请求，用 `context.WithTimeout` 派生），service 约 1500ms（一个业务动作，可能含多次调用），db 单条语句约 800ms（不超过 service 的一半）。

```go
func (r *UserRepo) Get(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
	defer cancel()

	var u User
	err := r.db.QueryRowContext(ctx, `SELECT id, email, name FROM users WHERE id = ?`, id).
		Scan(&u.ID, &u.Email, &u.Name)
	if err != nil {
		return nil, err
	}
	return &u, nil
}
```

`QueryContext` 返回后，`Rows` 的迭代仍受同一个 `ctx` 控制；若 `ctx` 在迭代中途取消，只有 `rows.Err()` 会返回取消原因，所以遍历结束必须检查它。

## 4. 预处理与参数化

### 4.1 必须用占位符

```go
// 错误：字符串拼接，name 为 `' OR 1=1 --` 时条件恒真
q := "SELECT id FROM users WHERE name = '" + name + "'"
// 正确：值通过占位符单独传递，驱动负责转义与类型编码
row := db.QueryRowContext(ctx, `SELECT id FROM users WHERE name = ?`, name)
```

占位符风格因数据库而异，同一份 SQL 不能跨库复制：

| 数据库 | 占位符 | 说明 |
| --- | --- | --- |
| MySQL | `?` | 位置绑定 |
| PostgreSQL | `$1` `$2` | 位置编号，顺序敏感 |
| SQLite | `?` / `?NNN` / `:name` | 匿名、编号、命名三种 |

其余规则：表名、列名、`ORDER BY` 方向不能参数化，必须用白名单映射到固定字符串，绝不接收用户输入；不要在循环里逐条 `PrepareContext`，需要复用就预处理好 `Stmt` 并共享；批量写入优先用单条多值 INSERT 或数据库提供的批量接口，而不是循环 N 次 `ExecContext`。

### 4.2 NULL 与自定义类型

| 方案 | 可空表达 | 取舍 |
| --- | --- | --- |
| 指针（`*string`） | nil 表示 NULL | 表达力好，但 JSON 序列化与解引用处处判空 |
| `sql.NullString` 等 | 结构体加一个有效位 | 语义明确，但跨层传递笨重，业务层通常仍要转换 |
| 自定义类型 | 自己实现 `sql.Scanner` 与 `driver.Valuer` | 最贴业务，适合领域类型；需要处理 nil 与驱动不认识的类型 |

自定义类型的两个硬约束：`Value` 的返回值必须是驱动能识别的基础类型（整数、浮点、字符串、字节切片、时间、nil），返回结构体或切片会报错；`Scan` 的入参可能是 nil，必须显式分支。

```go
func (t *Tags) Scan(src any) error          { /* 处理 nil、[]byte、string 三种入参 */ }
func (t Tags) Value() (driver.Value, error) { /* 返回 json.Marshal(t)，t 为 nil 时返回 nil */ }
```

## 5. 错误处理

### 5.1 无结果不是错误

`QueryRowContext` 在结果集为空时把 `sql.ErrNoRows` 通过 `Scan` 抛出；`QueryContext` 在无行时不报错，只是迭代立即结束。判定一律用 `errors.Is`：

```go
err := db.QueryRowContext(ctx, `SELECT id FROM users WHERE email = ?`, email).Scan(&id)
switch {
case errors.Is(err, sql.ErrNoRows):
	return nil, ErrNotFound
case err != nil:
	return nil, fmt.Errorf("query user by email: %w", err)
}
```

不要用错误文本比对（例如 `"sql: no rows in result set"`），文本在不同版本之间不保证稳定。

### 5.2 唯一键冲突的分层判定

不要只靠硬编码错误字符串，按可靠性分三层：

| 层 | 做法 | 说明 |
| --- | --- | --- |
| 一 | 驱动导出的错误类型 / 错误码 | 最稳。MySQL 重复键错误码 1062；PostgreSQL SQLSTATE 23505；SQLite 有约束冲突的扩展错误码。驱动导出的类型名与字段名以各驱动文档为准 |
| 二 | 约束名与关键字的字符串匹配 | 兜底，用于拿不到结构化错误码的场景，例如 SQLite 的 `UNIQUE constraint failed` |
| 三 | 冲突前置查询 | 只用于改善提示信息，不能替代唯一索引；并发下必然有竞态 |

```go
func isUniqueViolation(err error) bool {
	if err == nil {
		return false
	}
	// 第一层：在此断言驱动导出的错误类型并读错误码（类型名与字段名见驱动文档）。
	// 第二层：字符串兜底。
	msg := strings.ToLower(err.Error())
	return strings.Contains(msg, "unique constraint") || strings.Contains(msg, "duplicate entry")
}
```

### 5.3 重试与幂等

可重试的错误：死锁（MySQL 1213、PostgreSQL 40001）、连接重置、明确的超时。不可重试：语法错误、约束冲突、类型不匹配。三条规则：只有幂等操作才重试，写操作靠唯一索引或业务幂等键（客户端请求 ID 落库去重）保证幂等；指数退避加随机抖动，设最大次数与总时间上限，并尊重 `ctx`；重试的是整个事务——死锁时数据库已经回滚了该事务，重试单条语句会破坏一致性。

## 6. 事务

```go
// 事务骨架（省略函数签名与参数校验）
tx, err := r.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
if err != nil {
	return fmt.Errorf("begin tx: %w", err)
}
defer tx.Rollback() // Commit 之后 Rollback 返回 sql.ErrTxDone，忽略即可

res, err := tx.ExecContext(ctx,
	`UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?`, cents, from, cents)
if err != nil {
	return fmt.Errorf("debit: %w", err)
}
if n, err := res.RowsAffected(); err != nil || n == 0 {
	return fmt.Errorf("debit rejected for account %d", from)
}
if _, err := tx.ExecContext(ctx, `UPDATE accounts SET balance = balance + ? WHERE id = ?`, cents, to); err != nil {
	return fmt.Errorf("credit: %w", err)
}
if err := tx.Commit(); err != nil {
	return fmt.Errorf("commit: %w", err)
}
```

要点：

- `defer tx.Rollback()` 必须紧跟 `BeginTx`，在错误分支 `return` 之前注册，否则会漏掉连接释放。
- 把业务约束写进 `WHERE`（`balance >= ?`）并检查受影响行数，比「先查后改」安全。
- 隔离级别：`LevelReadCommitted` 足够多数业务；涉及「读-判断-写」的计数、库存、状态机才提升到 `LevelRepeatableRead` 或 `LevelSerializable`，级别越高冲突与死锁越多。各数据库默认级别不同（MySQL InnoDB 默认 `RepeatableRead`，PostgreSQL 默认 `ReadCommitted`），不要假设一致。
- 死锁是数据库主动检测并回滚其中一个事务，表现为一次可重试的错误，不是崩溃。
- 避免长事务：事务内不做 HTTP / RPC 调用，不发消息，不做文件 IO，不等待用户输入；网络耗时会把锁持有时间放大几个数量级。多条更新固定按主键升序执行，可显著降低死锁概率。

## 7. 迁移

| 工具 | 模块路径 | 工作方式 |
| --- | --- | --- |
| golang-migrate | `github.com/golang-migrate/migrate` | 版本号 + up/down 成对 SQL 文件，CLI 与库两种用法 |
| goose | `github.com/pressly/goose` | SQL 或 Go 函数作为迁移单元，可在同一工具内做数据回填 |

均以官方最新稳定版为准；文件命名、记录表默认名称、指令注释等细节以官方文档为准（<https://github.com/golang-migrate/migrate>、<https://github.com/pressly/goose>）。

向前兼容原则（在线服务必须遵守）：先加可空列或带默认值的列，让旧版本代码仍能写；双写过渡，新旧列同时写、读路径仍读旧列；分批回填历史数据，避免一次性大事务锁表；切换读路径到新列并观察一段时间；停止写旧列，最后再删列，删除要跨发布周期。

其他约束：禁止在同一次发布里重命名列或改类型，重命名等价于「加新列 + 双写 + 回填 + 删旧列」；发布顺序是先迁移数据库、再滚动发布应用，回滚只回滚应用版本；DDL 的隐式提交行为各数据库不同，不要假设把 DDL 放进事务就能整体回滚。

## 8. 查询生成 vs ORM

### 8.1 三方对比

| 维度 | `database/sql` | sqlc | GORM |
| --- | --- | --- | --- |
| 输入形态 | 手写 SQL + 手写扫描 | 手写 SQL，工具生成 Go 代码 | 结构体与链式 API |
| 类型安全 | 靠人工，列与字段错配只能运行时发现 | 强，参数与返回值由 SQL 推导 | 中等，链式调用可编译但语义可能不符预期 |
| 可读性 | SQL 与 Go 分离，直观 | SQL 集中在一个目录，直观 | 复杂查询退化为长链式调用 |
| 复杂查询与性能可控性 | 完全可控 | 与手写 SQL 等价 | 复杂查询需 Raw SQL 逃生舱，优化前要先看生成的 SQL |
| 学习成本 | 低，但要手写大量样板 | 中，要学生成规则与指令注释 | 低，隐式行为要单独学 |
| 适合场景 | 少量查询、教学、需要极致掌控 | 以 SQL 为中心的中大型服务 | 快速原型、以 CRUD 为主的内部系统 |

模块路径：`github.com/sqlc-dev/sqlc`、`gorm.io/gorm`，以官方最新稳定版为准。选型建议：以 SQL 为核心、复杂查询多、要求性能可控，选 `sqlc`；以实体 CRUD 为主、希望少写 SQL，选 `GORM`；为了理解原理或查询量极小，用 `database/sql`。

### 8.2 同一需求的两种骨架

需求：文章列表分页，带标签过滤，按创建时间倒序。`database/sql` 思路是动态拼 WHERE、参数仍走占位符，并另发一条 COUNT：

```go
args := []any{status}
q := `SELECT id, title, created_at FROM articles WHERE status = ?`
if tag != "" {
	q += ` AND EXISTS (SELECT 1 FROM article_tags t WHERE t.article_id = articles.id AND t.tag = ?)`
	args = append(args, tag)
}
q += ` ORDER BY created_at DESC, id DESC LIMIT ? OFFSET ?`
args = append(args, limit, offset)
rows, err := db.QueryContext(ctx, q, args...)
```

`sqlc` 思路是把 SQL 写进 `.sql` 文件并加指令注释，由工具生成类型安全的函数：

```sql
-- name: ListArticles :many
SELECT id, title, created_at FROM articles
WHERE status = $1 ORDER BY created_at DESC, id DESC LIMIT $2 OFFSET $3;

-- name: CountArticles :one
SELECT count(*) FROM articles WHERE status = $1;
```

生成的方法直接返回结构体切片与总数，参数与列的对应关系由生成代码保证。代价是可选的标签过滤通常要拆成多条查询、在 Go 侧选择调用哪一条；换来的是没有手写 `Scan` 的参数顺序风险。

## 9. 索引与查询优化基础

### 9.1 B+ 树、最左前缀与回表

关系型数据库的索引绝大多数是 B+ 树：数据只存在叶子节点，叶子之间用链表相连，非叶子节点只存键用于导航。由此得到两个性质：范围扫描沿叶子链表顺序读，效率高；树高通常 3–4 层就能覆盖千万级行，一次等值查找只需几次页读取（命中缓冲池更快）。联合索引 `(a, b, c)` 能用于 `a`、`a+b`、`a+b+c` 的前缀等值匹配，`b` 单独作为条件用不上这个索引，这就是最左前缀匹配。二级索引的叶子存的是索引列加上主键值，不含其余列；查询需要的列如果全在索引里就是覆盖索引，无需访问主数据，否则要按主键回到主数据结构取整行，称为回表。回表行数多时，优化器可能直接放弃索引改走全表扫描。

### 9.2 EXPLAIN 的通用读法

不同数据库与版本的输出列名不同，但要看的信息一致：访问方式（全表扫描 / 索引扫描 / 索引等值查找 / 范围扫描，判断是否走了预期路径）、实际使用的键（是否命中建好的联合索引）、预估扫描行数（与实际返回差距过大通常意味着统计信息过期或索引选择性差）、额外动作（是否用临时表、是否触发文件排序，这两项在大结果集上代价明显）、是否覆盖索引。读法顺序是：先看访问方式是否退化，再看键是否命中，最后看额外动作。

### 9.3 联合索引顺序与深分页

等值条件列放前面，范围条件列放最后；等值列之间选择性高（区分度大）的放前面；`ORDER BY` 的列顺序与索引顺序一致时可直接利用索引顺序，避免额外排序；索引不是越多越好，每个索引都要在写入时维护，写密集表的索引数量要克制。

`LIMIT 20 OFFSET 100000` 的问题是数据库要先扫描并丢弃前 100000 行。两种优化：游标分页，用上一页最后一条记录的排序键作为起点，条件是 `(created_at, id) < (?, ?)` 配 `ORDER BY created_at DESC, id DESC LIMIT ?`，前提是排序键唯一稳定（所以带上主键做 tie-break），代价是不能随机跳页；延迟关联，先在覆盖索引上分页拿到主键集合，再用主键 JOIN 回主表取整行，把回表次数压到页大小。另外，`SELECT *` 会多读不需要的列、无法使用覆盖索引、增加网络与反序列化开销，大字段（TEXT / BLOB）还可能触发额外溢出页读取；结构变化会静默影响扫描逻辑，`Scan` 的参数个数也必须跟着改，因此应明确列出列名。

## 10. 完整示例

下面用 `database/sql` + `modernc.org/sqlite` 完成用户表的增删改、事务转账与游标分页；单行查询与 `sql.ErrNoRows` 判定见 5.1。驱动相关的部分只出现在 `sql.Open` 的驱动名与占位符风格上，金额一律用整数「分」存储以避免浮点误差。

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "modernc.org/sqlite" // 纯 Go 驱动，注册名 "sqlite"
)

const schema = `
CREATE TABLE IF NOT EXISTS users (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    email      TEXT    NOT NULL UNIQUE,
    name       TEXT    NOT NULL,
    balance    INTEGER NOT NULL DEFAULT 0,
    created_at TEXT    NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_users_created ON users(created_at DESC, id DESC);
`

type User struct {
	ID        int64
	Email     string
	Name      string
	Balance   int64
	CreatedAt string
}

func createUser(ctx context.Context, db *sql.DB, email, name string) (int64, error) {
	res, err := db.ExecContext(ctx,
		`INSERT INTO users (email, name, balance, created_at) VALUES (?, ?, 0, ?)`,
		email, name, time.Now().UTC().Format(time.RFC3339Nano))
	if err != nil {
		return 0, fmt.Errorf("insert user %s: %w", email, err) // 唯一键冲突判定见 5.2
	}
	return res.LastInsertId()
}

// transfer 在同一事务内完成「扣款 + 入账」，任一步失败整体回滚。
func transfer(ctx context.Context, db *sql.DB, from, to, cents int64) error {
	tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	res, err := tx.ExecContext(ctx,
		`UPDATE users SET balance = balance - ? WHERE id = ? AND balance >= ?`, cents, from, cents)
	if err != nil {
		return fmt.Errorf("debit: %w", err)
	}
	if n, err := res.RowsAffected(); err != nil || n == 0 {
		return fmt.Errorf("account %d has insufficient balance", from)
	}
	if _, err := tx.ExecContext(ctx,
		`UPDATE users SET balance = balance + ? WHERE id = ?`, cents, to); err != nil {
		return fmt.Errorf("credit: %w", err)
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}

// listUsersAfter 使用游标分页，避免大 OFFSET 的扫描开销。
func listUsersAfter(ctx context.Context, db *sql.DB, afterID int64, limit int) ([]User, error) {
	rows, err := db.QueryContext(ctx,
		`SELECT id, email, name, balance, created_at FROM users WHERE id > ? ORDER BY id ASC LIMIT ?`,
		afterID, limit)
	if err != nil {
		return nil, fmt.Errorf("list users: %w", err)
	}
	defer rows.Close()
	var out []User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Email, &u.Name, &u.Balance, &u.CreatedAt); err != nil {
			return nil, fmt.Errorf("scan user: %w", err)
		}
		out = append(out, u)
	}
	if err := rows.Err(); err != nil { // ctx 取消或网络中断只在这里体现
		return nil, fmt.Errorf("iterate users: %w", err)
	}
	return out, nil
}

// must 仅用于示例，把启动与演示路径上的错误直接终止。
func must[T any](v T, err error) T {
	if err != nil {
		log.Fatal(err)
	}
	return v
}

func main() {
	ctx := context.Background()
	db := must(sql.Open("sqlite", "file:demo.db?cache=shared")) // 池参数按第 2 节设置
	defer db.Close()
	if err := db.PingContext(ctx); err != nil { // Open 不建连接，必须显式探活
		log.Fatal(err)
	}
	must(db.ExecContext(ctx, schema))

	a := must(createUser(ctx, db, "a@example.com", "alice"))
	b := must(createUser(ctx, db, "b@example.com", "bob"))
	must(db.ExecContext(ctx, `UPDATE users SET balance = 1000 WHERE id = ?`, a))
	must(transfer(ctx, db, a, b, 300))
	must(db.ExecContext(ctx, `DELETE FROM users WHERE id = ?`, b)) // 删除（D）
	for _, u := range must(listUsersAfter(ctx, db, 0, 20)) {
		fmt.Printf("id=%d email=%s balance=%d\n", u.ID, u.Email, u.Balance)
	}
}
```

换数据库时只改两处：`sql.Open` 的驱动名与导入路径，以及 SQL 里的占位符风格（MySQL 用 `?`，PostgreSQL 用 `$1` `$2`）。生产使用 MySQL 或 PostgreSQL 时，连接池参数必须按第 2 节重新推导。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `sql.Open` 的 error 当成连通性检查 | 启动正常，第一个请求报错 | `Open` 不建连接 | 启动时 `PingContext` |
| 不设置连接池参数 | 高峰打爆数据库，或空闲期锯齿建连 | 默认值不适合生产 | 按 `max_connections` 与实例数推导四个参数 |
| `Rows` 不 `Close` | 池内连接被占满，请求排队超时 | 提前 return 跳过了关闭 | 查询后立刻 `defer rows.Close()` |
| 遍历后不检查 `rows.Err()` | 结果静默截断，少数据无报错 | 错误只在 `Err()` 暴露 | 循环结束必查 `rows.Err()` |
| 字符串拼接 SQL | SQL 注入 | 把值当语法拼接 | 占位符传参，标识符用白名单 |
| 用错误字符串判定「无结果」 | 升级 Go 或驱动后判定失效 | 错误文本无兼容保证 | `errors.Is(err, sql.ErrNoRows)` |
| 只做字符串匹配判唯一冲突 | 换数据库或换语言环境后失效 | 文本会变 | 优先驱动错误类型与错误码，字符串仅兜底 |
| 事务里发 HTTP 请求 | 锁等待飙升，死锁变多 | 把网络延迟算进锁持有时间 | 网络调用放到事务外 |
| 事务内混用 `db` 与 `tx` | 读不到本事务数据，甚至自锁 | 两者取到不同连接 | 事务内所有语句走 `tx` |
| `defer tx.Rollback()` 写在错误分支之后 | 出错路径漏掉回滚，连接不释放 | 注册时机太晚 | `BeginTx` 成功后立刻 defer |
| 深分页用大 `OFFSET` / 长期 `SELECT *` | 页越深越慢；加列后行为变化且无法覆盖索引 | 扫描并丢弃 offset 行；依赖列顺序与数量 | 游标分页或延迟关联；显式列出列名 |
| 大表一次性加非空列 | 迁移期间锁表，服务不可用 | 不向前兼容 | 先加可空列，回填后再收紧约束 |

## 12. 动手练习

1. 实现上面的示例，把 `SetMaxOpenConns` 从 8 改成 1，观察并发请求下的排队现象。
2. 故意删掉 `rows.Close()`，在循环里加 `time.Sleep`，用池统计观察使用中连接数持续顶满。
3. 把 `transfer` 的扣减条件从 `WHERE ... AND balance >= ?` 改成先 `SELECT` 再 `UPDATE`，构造并发转账，说明为什么会出现负余额。
4. 给文章表建 `(status, created_at, id)` 联合索引，用 `EXPLAIN` 对比带 `status` 过滤与不带该过滤的查询，解释差异。
5. 用 `golang-migrate` 或 `goose` 写一对 up/down 迁移，再补「加列 + 双写 + 回填」的第二对迁移，并用 `sqlc` 重写同一个分页查询。

## 13. 自检清单

- [ ] 进程启动时对数据库做了 `PingContext` 健康检查。
- [ ] 四个连接池参数有推导依据，并写进配置而不是散落在代码里。
- [ ] 所有 SQL 调用都传了 `ctx`，handler / service / db 三层超时逐层收紧。
- [ ] 每个 `Rows` 都 `Close`，每次迭代后都检查了 `rows.Err()`。
- [ ] 所有值都走占位符；表名与列名来自白名单映射。
- [ ] 「无结果」用 `errors.Is(err, sql.ErrNoRows)` 判定，唯一键冲突按「驱动错误码优先、字符串兜底」分层。
- [ ] 事务统一 `BeginTx` + `defer tx.Rollback()`，事务内没有网络调用。
- [ ] 迁移全部向前兼容，删列与改类型至少晚一个发布周期。
- [ ] 慢查询用 `EXPLAIN` 看过访问方式与使用索引，深分页已改成游标分页或延迟关联。

## 14. 延伸阅读

- 标准库文档：<https://pkg.go.dev/database/sql>、<https://pkg.go.dev/database/sql/driver>、<https://pkg.go.dev/context>
- Go 1.26 新增 `database/sql.ConvertAssign`；Go 1.27 新增 `database/sql/driver` 的 `RowsColumnScanner` 接口：<https://go.dev/doc/go1.26>、<https://go.dev/doc/go1.27>
- sqlc 文档：<https://docs.sqlc.dev>；GORM 文档：<https://gorm.io/docs/>
- golang-migrate：<https://github.com/golang-migrate/migrate>；goose：<https://github.com/pressly/goose>
- 书籍：《Go 语言高级编程》，柴树杉、曹春晖，人民邮电出版社
- 书籍：《数据密集型应用系统设计》，Martin Kleppmann，中国电力出版社
