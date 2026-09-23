# 作业 03：不用框架——database/sql 用户表 CRUD 与迁移

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W2–W3 |
| 发放日期 | 2026-10-12 |
| 交付日期 | 2026-10-25 |
| 预计工时 | 18 小时 |
| 难度 | 进阶 |
| 对应讲义 | L08 |
| 前置作业 | hw02 |

## 1. 背景与目标

本作业是阶段一的数据层收口：不使用 ORM、不使用 Web 框架，用 `database/sql`（<https://pkg.go.dev/database/sql>）把一张用户表从建表、迁移、仓储、事务到分页完整走一遍。考点是 SQL 与 Go 的接缝：占位符、`ctx` 传递、`NULL` 与零值、错误分层、连接池取值。

- 迁移可重复执行、可回滚，索引有明确的服务对象；仓储依赖倒置：接口在业务侧，SQL 只在数据侧。
- 错误分层：驱动错误映射为领域错误，上层只认领域错误。
- 事务正确性由测试证明，连接池参数有推导过程。

## 2. 需求（必须项）

**M1 表结构。** 实现 `users` 表，至少包含下列列；类型自选但要在 README 说明理由。

| 列 | 约束 | 说明 |
| --- | --- | --- |
| `id` | 主键 | 自增或应用生成，二选一并说明 |
| `email` | 唯一索引 | 登录标识；大小写比较策略写进 README |
| `password_hash` | 非空 | 只存哈希；摘要算法与参数写进 README |
| `nickname` | 可空 | 用于验证 `NULL` 与空字符串的区别处理 |
| `status` | 非空，带默认值 | 取值集合在 README 中枚举 |
| `created_at` | 非空，索引 | 分页游标依赖它 |
| `updated_at` | 非空 | 应用维护或数据库维护，二选一并说明 |

M5 的余额转移允许再加余额列与流水表；额外结构必须在 README 的 ER 说明里画出关系。

**M2 迁移。** 工具二选一：`github.com/golang-migrate/migrate` 或 `github.com/pressly/goose`（均以官方最新稳定版为准，只写模块路径，不指定版本号）；也可手写 `.sql` + 自建 `migrate` 子命令。要求：

1. 每个变更同时提供向上与向下两个方向，**可回滚**。
2. 迁移**可重复执行**：重复运行已应用的迁移不报错、不产生副作用。
3. 索引至少两个：`email` 唯一索引、`created_at` 索引；README 说明每个索引服务于哪条查询，并把迁移与回滚命令写进去，交付时实际执行一次，真实输出贴进 `MIGRATION.md`。

**M3 仓储层。** 接口定义在业务侧包（不含任何 SQL 与驱动导入），MySQL 实现在数据侧包，方法至少覆盖：创建用户、按 id 查、按 email 查、更新昵称、删除（软删或硬删，二选一并说明）、列表分页。

1. 每个方法第一个参数是 `context.Context`，并把 `ctx` 传给每一次查询调用。
2. 所有 SQL 使用**占位符**，禁止拼接值。README 说明所用驱动的占位符风格；如果测试改用 SQLite 驱动（如 `modernc.org/sqlite` 或 `github.com/mattn/go-sqlite3`，以官方最新稳定版为准），必须显式说明两种驱动占位符写法的差异。
3. 不存在记录统一返回领域「未找到」错误：用 `errors.Is(err, sql.ErrNoRows)` 判定后转换，不得把 `sql.ErrNoRows` 泄漏到业务层，也不得与其他错误混判。
4. 唯一键冲突映射为领域错误 `ErrEmailExists`，采用**分层策略**：先看驱动返回错误中的结构化错误码，取不到时才回退到错误文本包含关系；README 说明为什么不能只硬编码一个错误字符串。
5. 禁止 `SELECT *`：列名显式列出且与 `Scan` 顺序一致。

**M4 连接池。** 显式设置 `SetMaxOpenConns`、`SetMaxIdleConns`、`SetConnMaxLifetime`、`SetConnMaxIdleTime`，四个参数都不得依赖默认值。README 必须写出取值**推导过程**，至少包含：数据库端 `max_connections`、应用实例数、单实例目标 QPS、单次查询平均耗时，并由这些量推出两个上界。

**M5 事务。** 实现 `TransferCredits`（或等价的余额转移）：同一事务内完成「扣减来源、增加目标、写入一条流水」，任一步失败整体回滚。

1. 事务内每次执行都带 `ctx`。
2. 回滚的 `defer` 必须紧跟事务创建之后，不得写在错误处理分支后面；提交失败也必须返回错误。
3. 写测试证明失败时整体回滚：构造失败条件（目标不存在、余额不足或人为约束冲突），断言失败后两侧余额与流水条数**都没有变化**。

**M6 分页。** 列表查询用 `LIMIT` / `OFFSET` 实现，并说明深分页问题（偏移越大，被扫描并丢弃的行越多）。游标分页是加分项。

**M7 驱动入口。** 提供 CLI 子命令或最小 HTTP 接口驱动 CRUD 全流程。若用 HTTP，只允许 `net/http`（<https://pkg.go.dev/net/http>）标准库，禁止 Web 框架；请求体解析、参数校验、错误到状态码的映射都要写清。

**M8 测试。** 对仓储层写集成测试，至少覆盖三类用例。

| 用例 | 断言要点 |
| --- | --- |
| 重复邮箱 | 第二次创建返回 `ErrEmailExists` 且可用 `errors.Is` 判定；表中仍只有一条记录 |
| 不存在的 id | 返回领域「未找到」错误，而不是 `sql.ErrNoRows` 原始值 |
| 事务回滚 | 失败后余额与流水完全未变 |

测试需要真实数据库：用 Docker 起 MySQL（命令写进 README），或改用 SQLite 驱动并说明占位符差异。测试必须能在干净库上一次跑通，并自行准备与清理数据。

## 3. 需求（加分项）

1. **游标分页**：按 `created_at, id` 双列游标翻页，与 `LIMIT/OFFSET` 版本对比 10 万行规模下的执行计划差异。
2. **`EXPLAIN` 验证**：对「按 email 查」「按 created_at 分页」各做一次 `EXPLAIN`，把文本结果放进 README，说明索引是否被用上。
3. **用 `go.mod` 的 `tool` 指令（Go 1.24 起）管理迁移工具**，协作者无需全局安装，用 `go tool` 运行。

## 4. 技术约束

- 语言基线：**Go 1.25**（本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`）。加分项 3 标注 Go 1.24 起可用。
- 数据库访问只用 `database/sql` 与所选驱动；禁止 ORM 与 Web 框架。第三方只写模块路径 + 「以官方最新稳定版为准」。
- SQL 只出现在迁移文件或数据侧常量中，业务侧不得出现 SQL 字符串。
- 时间统一用 `time.Time`：README 说明时区策略与驱动连接参数如何配套；源文件 UTF-8、无 BOM，`gofmt -l .` 必须为空，密码只存哈希（测试数据也不得出现明文口令）。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `~/go-learn-work/hw03/userstore/migrations/*.sql` | 向上与向下迁移文件 |
| `~/go-learn-work/hw03/userstore/internal/domain/*.go` | 领域类型与领域错误（含 `ErrEmailExists`） |
| `~/go-learn-work/hw03/userstore/internal/store/*.go` | 仓储接口 + MySQL 实现 + 错误映射 |
| `~/go-learn-work/hw03/userstore/internal/store/store_test.go` | 三类必需集成测试 + 回滚测试 |
| `~/go-learn-work/hw03/userstore/cmd/userstore/main.go` | CLI 或 HTTP 入口 |
| `~/go-learn-work/hw03/userstore/README.md` + `MIGRATION.md` | ER 与索引说明、连接池推导、迁移命令；迁移执行与回滚的真实输出 |

## 6. 验收标准（可执行命令）

```bash
go vet ./...                       # 无输出，退出码 0
gofmt -l .                         # 输出为空
go test -race ./...                # 集成测试全部 PASS，无 DATA RACE
```

迁移执行与回滚（示例，实际命令以所选工具为准，必须写进 README）：

```bash
go run ./cmd/userstore migrate up      # 应用全部迁移
go run ./cmd/userstore migrate up      # 重复执行：必须成功且无副作用
go run ./cmd/userstore migrate down 1  # 回滚最后一步
```

人工验收：README 的连接池取值有推导链；`MIGRATION.md` 是真实粘贴。

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| schema 与迁移 | 20 | 列与约束齐全（8）；双向可回滚（6）；可重复执行（3）；两个索引有查询依据（3） |
| 仓储与错误分层 | 25 | 接口与实现分离、无 SQL 泄漏（8）；`ctx` 与占位符全覆盖、无 `SELECT *`（7）；未找到与唯一冲突映射正确且可 `errors.Is`（10） |
| 事务正确性 | 15 | 回滚 `defer` 位置正确（5）；回滚测试能证明数据未变（10） |
| 连接池与文档 | 10 | 四个参数显式设置（4）；推导含实例数、QPS、`max_connections`（6） |
| 分页 | 10 | `LIMIT/OFFSET` 正确并说明深分页问题 |
| 测试 | 20 | 三类必需用例齐全（12）；测试可重复运行、数据自清理（4）；`go test -race` 通过（4） |

## 8. 提示与思路

- 先写 SQL、用数据库客户端手工验证，最后才写 Go；直接在 Go 里调 SQL 会把 SQL 错误和 Go 错误混在一起排查。错误映射集中放在数据层最外沿的一个函数里：结构化错误码是唯一可靠依据，字符串匹配只作兜底。
- 连接池取值思路：单实例连接上界约为数据库 `max_connections` 除以应用实例数，再乘安全系数；`MaxIdleConns` 与 `MaxOpenConns` 不宜相差过大，否则连接反复创建销毁。
- 事务测试要同时断言两侧余额与流水条数，只看一侧会漏掉部分提交。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| 忘记 `defer rows.Close()` | 连接被长期占用，`MaxOpenConns` 用尽后请求全部阻塞 | 查询成功后立刻 `defer`，并在循环中检查迭代错误 |
| `Scan` 类型不匹配 | 报「converting NULL to string」或数值溢出 | 可空列用可空类型或指针；列顺序与 `Scan` 参数严格对齐 |
| `sql.ErrNoRows` 与其他错误混判 | 把连接失败当成「用户不存在」，返回 200 空对象 | 只在确认无行时用 `errors.Is` 匹配，其他错误原样上抛再映射 |
| `defer tx.Rollback()` 写在错误处理之后 | 提前 `return` 时事务未回滚，连接泄漏 | 事务创建后立即注册回滚；提交成功后再回滚是无害的 |
| `time.Time` 时区不一致 | 同一记录读出的时间相差 8 小时 | 明确存 UTC，驱动连接参数与应用解析统一时区 |
| 连接池过大 | 单实例压垮数据库，其他服务被拒连 | 按推导公式设上界，并压测观察数据库端连接数 |
| `SELECT *` | 加列后 `Scan` 数量不匹配 | 显式列出列名 |

## 10. 参考实现要点

只给关键设计与片段，不给完整成品。

表与索引（示意）：`id` 主键、`email` 唯一索引、`created_at` 索引；`nickname` 可空以验证 `NULL` 与空串的差异；`status` 带默认值；`created_at` 与 `updated_at` 用带小数秒的日期时间类型。余额转移所需的余额列与流水表另行加上，并在 README 里画清关系。

```sql
CREATE TABLE users (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  email         VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  nickname      VARCHAR(64)  NULL,
  status        TINYINT      NOT NULL DEFAULT 1,
  created_at    DATETIME(6)  NOT NULL,
  updated_at    DATETIME(6)  NOT NULL,
  UNIQUE KEY uk_users_email (email),
  KEY idx_users_created_at (created_at)
);
```

显式列名 + 占位符 + 只对「无行」做 `errors.Is` 判定：

```go
const qGetByEmail = `SELECT id, email, password_hash, nickname, status, created_at, updated_at FROM users WHERE email = ?`

err := r.db.QueryRowContext(ctx, qGetByEmail, email).Scan(&u.ID, &u.Email, &u.PasswordHash,
	&u.Nickname, &u.Status, &u.CreatedAt, &u.UpdatedAt)
if errors.Is(err, sql.ErrNoRows) {
	return nil, domain.ErrUserNotFound // 不向业务层泄漏 sql.ErrNoRows
}
if err != nil {
	return nil, fmt.Errorf("query user by email: %w", err)
}
```

事务骨架：`BeginTx` 后立刻 `defer tx.Rollback()`，事务内一律用带 `ctx` 的执行与查询方法，最后 `Commit` 并检查其错误；错误映射函数 `mapDriverError` 是唯一把驱动信息翻译成 `domain.ErrEmailExists` 等领域错误的地方。游标分页（加分项）的排序键必须唯一且稳定，因此用 `created_at` 与 `id` 双列游标。
