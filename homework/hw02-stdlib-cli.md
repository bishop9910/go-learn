# 作业 02：标准库综合 CLI——日志清洗、JSON 配置与并发 URL 检查

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W1–W2 |
| 发放日期 | 2026-10-08 |
| 交付日期 | 2026-10-18 |
| 预计工时 | 16 小时 |
| 难度 | 进阶 |
| 对应讲义 | L03、L04、L05 |
| 前置作业 | hw01 |

## 1. 背景与目标

在 hw01 的 `logscan` 骨架上填满三个真实子命令。本作业的考点不是「能跑」，而是**标准库的边界条件**：流式读取的内存上界、文件系统的越权防护、JSON 缺失与零值的区分、并发下的取消与超时、日志的分组字段。

- `clean`：把文件系统访问限制在一个明确的根目录内，并做到大文件不整体载入内存。
- `stats`：用配置驱动统计，配置解析必须能报出到底是哪个字段写错了。
- `check`：手写有界并发 + 超时 + 重试，为后续讲义改成 Worker Pool 留好对比基线。

## 2. 需求（必须项）

**M1 `logscan clean`：流式清洗。** 参数至少包含输入路径（可多个）与输出目标，输出目标为 `-` 时写标准输出。

1. 全程流式：使用 `bufio.Scanner`（<https://pkg.go.dev/bufio>）逐行读取、`bufio.Writer` 逐行写出，**不得**整体读入内存。验收方式：用 500 MB 级输入跑一次，观察进程内存不随文件大小线性增长（记录所看指标即可）。
2. 清洗规则（至少这四条，且顺序可配置）：去掉空行与纯空白行；按正则剔除噪声行；把时间戳归一化为 RFC3339 格式；把 Windows 的 `\r\n` 归一化为 `\n`。
3. 必须显式检查扫描结束后的 `Scanner.Err()`，不得只判断 `Scan()` 的返回值。
4. 超长行必须显式处理：用 `Scanner.Buffer` 提高单行上限，或改用 `bufio.Reader`（<https://pkg.go.dev/bufio>）自行切分。必须写一个「单行超过默认上限」的测试证明你的方案有效。
5. 写出前必须 `Flush`，且 `Flush` 的错误必须被返回或记录，不得丢弃。
6. **文件系统沙箱**：用 `os.Root` 把输入与输出限制在 `--root` 指定的目录内。`os.Root` 于 **Go 1.24** 引入，**Go 1.25** 补齐了大量方法（含 `ReadFile`、`WriteFile`、`MkdirAll`、`RemoveAll`、`Rename` 等，见 <https://go.dev/doc/go1.25>）。必须在代码注释中标注版本。
7. 写一个测试：构造指向根目录之外的路径（使用 `../` 越权访问），断言操作被拒绝，且错误可以通过 `errors.Is` / `errors.As` 判定。

**M2 `logscan stats`：JSON 配置驱动。** 新增 `--config config.json`，配置文件描述清洗规则与统计维度。

1. 配置结构必须能区分**字段缺失**与**零值**：可选字段用指针（`*T`）或显式存在性标记；对「缺失即默认」「缺失即报错」两类字段分别处理，并在 README 的 schema 表里标注。
2. 必须**拒绝未知字段**：配置文件里出现 schema 之外的键时，进程以退出码 2 结束，错误信息中带上**出错的字段名**与所在层级。
3. 配置校验错误必须带字段名，例如「`rules.regex: 非法正则`」，不得只报「配置无效」。
4. 使用 `encoding/json`（<https://pkg.go.dev/encoding/json>）的 v1 API 编写。
5. 版本说明写进 README 或代码注释（二选一，必须存在）：在 **Go 1.27+** 下 `encoding/json` 已由 v2 实现支撑，行为保持但**错误文本可能变化**，因此**不得在测试里断言错误字符串全等**，只能断言错误类型与关键字段；v2 默认更严格：拒绝字符串中的非法 UTF-8、拒绝 JSON 对象中的重复键。
6. 表格驱动测试至少覆盖：字段缺失、字段为显式零值、未知字段、非法正则、重复键（在 v1 与 v2 下行为差异要能说明）、整个文件不是合法 JSON。

**M3 `logscan check`：并发 URL 检查器。** 参数：`--concurrency`、`--timeout`、`--retries`。

1. 只用 `net/http`（<https://pkg.go.dev/net/http>）标准库与 `context`，禁止引入 HTTP 框架。
2. `context` 超时覆盖单次请求；`--timeout` 作用于每个请求而不是整个批次。
3. 有界并发：用带缓冲的 channel 做信号量，或等价的固定 worker 数量；`--concurrency` 的取值必须在启动时校验（例如必须为正）。
4. 重试：仅对可重试的失败（超时、连接错误、5xx）重试，重试次数不超过 `--retries`，重试之间要有退避。
5. 结果聚合：输出每条 URL 的最终状态（成功/失败、尝试次数、耗时），失败时必须带上原因。
6. `--json` 输出：结果以单行 JSON 打印到标准输出，可被管道直接解析。
7. `resp.Body` 必须关闭；在需要复用连接的前提下必须**排空**未读内容再关闭。
8. 必须做整体取消：收到中断信号或某个不可重试的致命错误时，能终止尚未开始的请求。测试中至少有一个用例验证取消路径。

预告：L11 会把这里的手写并发改造成 Worker Pool 与 `golang.org/x/sync/errgroup`（以官方最新稳定版为准），请在本作业的 README 里写下「当前实现与 Worker Pool 的差别」，为那次重构留锚点。

**M4 日志。** 全程使用 `log/slog`：

1. 支持 `text` 与 `json` 两种 handler，用 `--log-format` 选择；日志写标准错误，不污染标准输出。
2. 每条日志必须带 `run_id`：进程启动时生成一次，所有日志（含子命令内部）都继承该字段。
3. **至少使用一个 Go 1.25 的 slog 新能力**（标注版本）：`slog.GroupAttrs` 或 `slog.Record.Source`（见 <https://go.dev/doc/go1.25>）。使用 `slog.Record.Source` 时说明 Source 的获取代价。

**M5 测试。** 三个子命令各自有单元测试；另加一个用 `net/http/httptest`（<https://pkg.go.dev/net/http/httptest>）的集成测试，把 `check` 指向测试服务器，覆盖成功、超时、5xx 重试、取消四类路径。至少写 2 个 benchmark，且每个都调用 `b.ReportAllocs()`。

**M6 文档与样例。** `README.md` 包含：三个子命令的完整用法、`config.json` 的 schema 表格（字段、类型、是否必填、缺失时的行为）、退出码表、版本标注汇总（`os.Root` Go 1.24/1.25、slog 能力 Go 1.25、JSON v2 Go 1.27+）；仓库内提供 `testdata/sample.log` 与 `testdata/config.json` 各至少一份。

## 3. 需求（加分项）

1. 若使用 Go 1.26+ 工具链，用 `errors.AsType`（**Go 1.26** 新增）替代 `errors.As` 做类型化错误判定，并对比两者的写法差异（必须标注版本，不得声称在 1.25 可编译）。
2. `clean` 支持 `--in-place`，通过临时文件 + `os.Root` 下的 `Rename` 原子替换，并说明 Windows 上替换已打开文件的行为差异。
3. `check` 增加「同一主机串行、不同主机并行」的限速策略，避免把单个目标打挂。
4. 用 `bytes.Buffer.Peek`（**Go 1.26** 新增）改进超长行切分逻辑，并标注版本。

## 4. 技术约束

- 语言基线：**Go 1.25**（本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`）。所有 Go 1.26 / 1.27 的能力只能出现在标注了版本的加分项与说明文字里。
- 运行时依赖只用标准库（`bufio`、`encoding/json`、`flag`、`log/slog`、`net/http`、`net/http/httptest`、`os`、`regexp`、`context`、`sync`、`time`）。第三方框架一律禁止。
- module 路径可沿用 hw01 的 `logscan`；本作业视为 hw01 的延续，允许在 hw01 代码上扩展，但必须保证 hw01 的验收命令仍然通过。
- 所有源文件 UTF-8、无 BOM；`gofmt -l .` 必须为空。
- 测试不得依赖外网：所有 HTTP 交互走 `httptest` 服务器。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `~/go-learn-work/hw02/logscan/internal/cli/*.go` | 三个子命令的参数与分发 |
| `~/go-learn-work/hw02/logscan/internal/clean/*.go` | 流式清洗与 `os.Root` 沙箱 |
| `~/go-learn-work/hw02/logscan/internal/config/*.go` | 配置解析、校验、未知字段拒绝 |
| `~/go-learn-work/hw02/logscan/internal/check/*.go` | 并发检查器、超时、重试、聚合 |
| `~/go-learn-work/hw02/logscan/internal/*/\*_test.go` | 单元测试与 benchmark |
| `~/go-learn-work/hw02/logscan/internal/check/integration_test.go` | `httptest` 集成测试 |
| `~/go-learn-work/hw02/logscan/testdata/sample.log` | 样例日志（含 CRLF 与超长行） |
| `~/go-learn-work/hw02/logscan/testdata/config.json` | 样例配置 |
| `~/go-learn-work/hw02/logscan/README.md` | 用法 + schema 表 + 版本标注汇总 |
| `~/go-learn-work/hw02/logscan/DEMO.md` | 三条端到端演示命令与真实输出 |

## 6. 验收标准（可执行命令）

在 `~/go-learn-work/hw02/logscan` 下执行：

```bash
go vet ./...                                  # 无输出，退出码 0
gofmt -l .                                    # 输出为空
go test -race -cover ./...                    # 全部 PASS，无 DATA RACE
go test -bench . -benchmem ./...              # ≥2 个 benchmark，均打印 allocs/op
```

三条端到端演示命令（写入 `DEMO.md`，含实际输出）：

```bash
go run ./cmd/logscan clean --root testdata -o out.log testdata/sample.log
go run ./cmd/logscan stats --config testdata/config.json testdata/sample.log
go run ./cmd/logscan check --concurrency 4 --timeout 2s --retries 2 --json \
  https://example.invalid/ https://example.com/
```

人工验收：越权测试用例存在且断言了错误可判定；配置未知字段报错信息里能看到字段名。

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| `clean` | 25 | 流式且有内存上界证据；四条清洗规则正确；`Scanner.Err()` 与 `Flush` 错误被处理；超长行方案有效；`os.Root` 沙箱与越权测试通过 |
| `stats` | 20 | 缺失与零值可区分；未知字段被拒绝且报错含字段名；测试覆盖六类场景；版本说明存在 |
| `check` | 25 | 有界并发正确；单请求超时；重试与退避；结果聚合与 `--json`；`Body` 关闭与排空；取消路径有测试 |
| 日志与错误处理 | 10 | 两种 handler；`run_id` 全链路；至少用到一个 Go 1.25 slog 能力并标注版本 |
| 测试与 benchmark | 10 | 单元测试 + `httptest` 集成测试；≥2 个 benchmark 且用 `b.ReportAllocs()` |
| 文档 | 10 | README 六项齐全；schema 表准确；样例文件可用；演示命令可复现 |

## 8. 提示与思路

- 未知字段检测建议在解码器层面打开「未知字段报错」开关，再叠加一次基于 schema 的显式校验：前者抓到多余键，后者负责带字段名的业务校验错误。
- 区分缺失与零值的实践：把可选字段声明为指针，`nil` 表示缺失，`&0` 表示显式零值。测试里两种输入都要有。
- 并发检查器先写「一次只跑一个」的正确版本，再把并发度做成参数；并发 bug 大多是共享 map 与忘记取消。
- 重试只对幂等的 GET 做，且重试次数用尽后要区分「超时」与「5xx」，因为排查方向完全不同。
- 排空 `resp.Body` 的目的是让底层连接可复用；如果响应体极大而你不关心内容，排空要有上限，否则等于把大文件读进内存。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| `bufio.Scanner` 默认 64 KB 行上限 | 遇到超长行直接停止扫描，且 `Err()` 非 nil，容易被当成文件读完了 | 提高 `Scanner.Buffer` 上限或改用 `bufio.Reader`，并写测试 |
| 忘记 `Flush` | 输出文件缺尾部内容，测试偶发失败 | `defer` 里 `Flush` 且检查其错误 |
| `os.Root` 与绝对路径混用 | 越权本该被拒绝却报「路径不存在」，或直接绕过根目录 | 所有路径都相对根目录解析，越权测试用 `../` 构造 |
| JSON 未知字段未拒绝 | 用户写错键名，程序静默按默认值跑 | 打开未知字段报错，并把字段名带进错误 |
| 并发里共享 map | `go test -race` 报 DATA RACE，或结果条目丢失 | 每个 goroutine 写独立下标，或聚合阶段统一合并 |
| 忘记 `context` 取消 | 某个请求卡住，整个命令不返回 | 用带超时的 `context`；在 `select` 里同时监听取消 |
| `http.Client` 没设超时 | 单个请求挂数分钟 | 用 `context` 超时并同时设置客户端级超时兜底 |

## 10. 参考实现要点

只给关键设计决策与片段，不给完整成品。

清洗主循环的结构（流式 + 显式错误检查 + 显式 Flush）：

```go
sc := bufio.NewScanner(rootIn) // rootIn 由 os.Root 解出的文件句柄
sc.Buffer(make([]byte, 0, 64*1024), 4*1024*1024) // 显式提高单行上限
w := bufio.NewWriter(dst)
for sc.Scan() {
	line, err := rules.Apply(sc.Text())
	if err != nil {
		return fmt.Errorf("clean line: %w", err)
	}
	if _, err := w.WriteString(line + "\n"); err != nil {
		return fmt.Errorf("write line: %w", err)
	}
}
if err := sc.Err(); err != nil { // 必须显式检查
	return fmt.Errorf("scan input: %w", err)
}
if err := w.Flush(); err != nil { // 必须显式检查
	return fmt.Errorf("flush output: %w", err)
}
```

并发检查器的骨架：带缓冲 channel 作信号量实现有界并发，每个 goroutine 只写 `results[i]`（按序聚合，聚合阶段不加锁），并在 `select` 中同时监听 `ctx.Done()` 与信号量获取，实现取消路径。

配置校验的关键决策：结构体里可选字段用 `*T`，校验函数返回的错误必须带上字段路径（如 `rules.regex`），未知字段则在解码阶段就失败。版本说明单独成段，写清 Go 1.27+ 下 `encoding/json` 由 v2 支撑带来的两点严格化与错误文本变化，以及为什么测试不能断言错误字符串。
