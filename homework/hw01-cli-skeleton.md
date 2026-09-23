# 作业 01：多包 CLI 骨架——工具链、模块与错误处理

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W1 |
| 发放日期 | 2026-10-08 |
| 交付日期 | 2026-10-11 |
| 预计工时 | 8 小时 |
| 难度 | 入门 |
| 对应讲义 | L01、L02、L03 |
| 前置作业 | hw00 |

## 1. 背景与目标

W1 只有 4 天（2026-10-08 至 2026-10-11），目标是把 hw00 里零散的 `go` 命令经验收敛成一个**有边界、有测试、有退出码语义**的多包 CLI 骨架。本作业刻意只做骨架：先让包结构与错误链路立住，功能在 hw02 里填满。

- 会用 `flag` 包或标准库手写参数解析，理解子命令分发的常见写法与陷阱。
- 会按 `internal` 规则划分包边界，并说清「为什么这些包必须不可被外部导入」。
- 会写哨兵错误、自定义错误类型与 `%w` 包装链，并用 `errors.Is` / `errors.As` 判定。
- 会把错误统一翻译成进程退出码，而不是到处 `os.Exit`。

讲义参考 [L01 工具链与 Go Modules](../lectures/01-toolchain-and-modules.md) 与 [L03 错误处理与结构化日志](../lectures/03-errors-and-logging.md)。

## 2. 需求（必须项）

**M1 命令面。** 实现名为 `logscan` 的命令行程序，本阶段只支持：

| 调用 | 行为 |
| --- | --- |
| `logscan --help` | 打印总用法（含子命令列表与全局参数），退出码 0 |
| `logscan --version` | 打印版本号，退出码 0 |
| `logscan clean [flags] <输入...>` | 读取每个输入并输出统计占位结果 |
| `logscan stats [flags] <输入...>` | 读取每个输入并输出统计占位结果 |
| 未知子命令 / 未知参数 | 打印用法与错误说明到标准错误，退出码 2 |

占位结果不算数：`clean` 与 `stats` 本阶段只需遍历输入、统计行数与字节数并打印，不实现清洗与维度统计（那是 hw02 的要求）。

**M2 参数解析。** 只允许 `flag` 包或标准库手写解析，**禁止引入任何第三方 CLI 框架**。要求：

1. 全局参数与每个子命令的参数互不污染：`logscan clean --help` 与 `logscan --help` 打印不同内容。
2. 位置参数与选项可以混排，且 `logscan clean -- a-b.txt` 能处理以 `-` 开头的文件名。
3. 单个参数 `-` 表示从标准输入读取（对应 `os.Stdin`），并在输出中标注该输入来源为 `stdin`。
4. 未给出任何输入时，默认读取标准输入，并在标准错误提示这是默认行为。

**M3 多包结构。** 代码必须拆成下列包，不得把所有逻辑塞进 `main`：

| 包路径 | 职责 |
| --- | --- |
| `cmd/logscan` | 仅 `main`：调用 CLI 层、把返回的错误翻译为退出码并打印 |
| `internal/cli` | 子命令注册与分发、参数校验、把包内错误向外冒泡 |
| `internal/logline` | 单行解析：把一行原始文本解析为结构化结果 |
| `internal/version` | 版本号与构建信息，供 `--version` 使用 |

在 `README.md` 中用一段话说明：`internal` 目录的可见性规则是什么、为什么这三个包不该被外部模块导入、如果把它们提到顶层会产生什么后果。参考 <https://go.dev/doc/modules/layout>。

**M4 错误处理与退出码。** 这是本作业的核心评分项。

1. 在 `internal/logline` 定义至少 2 个**哨兵错误**（例如「行过长」与「字段缺失」），用 `errors.Is` 判定。
2. 定义至少 1 个**自定义错误类型**（实现 `error` 接口，带字段），用 `errors.As` 取出字段。
3. 错误从 `internal/logline` 到 `internal/cli` 再到 `cmd/logscan` 至少经过 **3 层 `%w` 包装**，每层补充一层上下文（文件路径、行号、子命令名）。
4. `main` 中统一翻译：`0` 成功、`1` 运行期错误、`2` 用法错误。除 `main` 外任何包都不得直接终止进程。
5. 包装链必须可被验证：写一个测试，构造三层包装后的错误，断言 `errors.Is` 与 `errors.As` 都能穿透。

退出码语义必须写进 README 表格：

| 退出码 | 含义 | 触发示例 |
| --- | --- | --- |
| `0` | 成功 | `logscan --version` |
| `1` | 运行期错误 | 输入文件不存在、行解析失败 |
| `2` | 用法错误 | 未知子命令、未知参数、缺少必需参数 |

**M5 单元测试与覆盖率。** 对 `internal/logline` 写**表格驱动测试**，至少 8 个用例，必须覆盖：空行、只有空白字符的行、行尾 `\r\n`（Windows 换行）、行尾 `\n`、超长行、字段数不足、字段数超出、含非 UTF-8 字节的行。同时给出覆盖率报告：

```bash
go test -cover ./...
go test -coverprofile=cover.out ./... 
```

**M6 文档。** `README.md` 至少包含：一句话定位、安装/构建方式、两个子命令的用法与全部参数、退出码表（M4）、`-` 与默认 stdin 的行为说明、包结构表（M3）。

## 3. 需求（加分项）

1. **并发处理多输入文件**：用 `sync.WaitGroup.Go`（**Go 1.25** 新增；本机 1.25.7 可用）为每个输入文件启动一个 goroutine，结果按输入顺序聚合。必须在代码注释中标注该 API 的版本，并写一段说明：与「`wg.Add(1)` + `go func(){...}()`」的写法相比，`sync.WaitGroup.Go` 帮你消除了哪一个容易写错的步骤。
2. **结构化日志**：全程使用 `log/slog`，默认输出到标准错误；提供 `--log-level` 控制级别（至少 `debug` / `info` / `warn` / `error` 四档，非法值按用法错误处理）。日志不得污染标准输出，`-json` 输出必须仍可被管道解析。
3. **`-json` 输出**：两个子命令支持 `-json`，把统计结果以单行 JSON 打印到标准输出。

## 4. 技术约束

- 语言基线：**Go 1.25**（本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`）。加分项 1 标注为 Go 1.25；如果某段代码只在更高版本可用，必须显式标注版本且不得混入必须项。
- 运行时依赖只用标准库。开发期可用 hw00 装好的 `golangci-lint`（v2 配置格式 `version: "2"`，见 <https://golangci-lint.run>）。
- module 路径自定（建议 `example.com/golearn/hw01/logscan`），`go.mod` 中的 `go` 行不得高于本机工具链。
- 所有源文件 UTF-8、无 BOM；提交前必须 `gofmt -w` 过一遍。
- `internal/logline` 的解析函数必须**无副作用**：不读文件、不写标准输出、不记日志，纯输入到输出，便于表格驱动测试。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `~/go-learn-work/hw01/logscan/go.mod` | 模块定义 |
| `~/go-learn-work/hw01/logscan/cmd/logscan/main.go` | 入口与退出码翻译 |
| `~/go-learn-work/hw01/logscan/internal/cli/*.go` | 子命令分发与参数校验 |
| `~/go-learn-work/hw01/logscan/internal/logline/*.go` | 行解析、哨兵错误、自定义错误类型 |
| `~/go-learn-work/hw01/logscan/internal/version/*.go` | 版本信息 |
| `~/go-learn-work/hw01/logscan/internal/logline/parse_test.go` | ≥8 个表格驱动用例 |
| `~/go-learn-work/hw01/logscan/README.md` | 用法、退出码表、包结构说明 |
| `~/go-learn-work/hw01/logscan/DEMO.md` | 3 条演示命令与真实输出 |
| `~/go-learn-work/hw01/logscan/cover.out` | 覆盖率报告文件 |

## 6. 验收标准（可执行命令）

在 `~/go-learn-work/hw01/logscan` 下执行：

```bash
go vet ./...                                  # 无输出，退出码 0
go test -race ./...                           # 全部 PASS，无 DATA RACE
gofmt -l .                                    # 输出为空
go build -o bin/logscan ./cmd/logscan         # 生成可执行文件
go test -cover ./...                          # 打印覆盖率，internal/logline 不低于 80%
```

`DEMO.md` 中必须包含 3 条端到端演示命令及**预期输出**（预期输出可以手写，但必须与真实运行结果一致）：

| 演示命令（PowerShell 示例） | 预期要点 |
| --- | --- |
| `./bin/logscan --version` | 打印版本号，退出码 0 |
| `Get-Content sample.log \| ./bin/logscan clean -` | 输入来源标注为 `stdin`，输出行数与字节数 |
| `./bin/logscan stats missing.log; echo $LASTEXITCODE` | 标准错误有带文件路径的错误链，退出码 1 |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 功能 | 25 | 两个子命令可用；`--help` / `--version` / 未知参数退出码正确；`-` 与默认 stdin 行为符合 M2 |
| 多包结构与 `internal` 边界 | 15 | 四个包职责无重叠；`main` 只做翻译；README 能说清 `internal` 规则 |
| 错误处理与退出码 | 20 | 哨兵错误 ≥2、自定义类型 ≥1、`%w` 链 ≥3 层；`errors.Is`/`errors.As` 穿透测试存在；0/1/2 语义一致 |
| 测试与覆盖率 | 20 | 表格驱动 ≥8 用例且含边界；`go test -race` 通过；覆盖率达标 |
| 文档与规范 | 20 | README 六项齐全；退出码表准确；`gofmt`/`go vet` 干净；演示命令可复现 |

## 8. 提示与思路

- `flag` 包的默认行为是「遇到第一个非选项参数就停止解析」。做子命令分发时，先在 `os.Args[1]` 上取出子命令名，再交给该子命令自己的参数集解析，可以避免大量边界问题；需要混排时再显式处理剩余参数。
- 退出码翻译只放在 `main`：`err := cli.Run(...)`，然后用 `errors.Is` 判断是否为用法错误，选择 `2` 还是 `1`。
- `%w` 链的层次建议：行解析层（行号/字段）→ 文件处理层（文件路径）→ 子命令层（子命令名）。每层只补充本层知道的信息。
- 表格驱动测试用结构体切片，字段至少包含：名称、输入、期望输出、期望错误（用哨兵错误比较，以便 `errors.Is` 生效）。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| `flag` 子命令解析顺序 | 全局参数写在子命令后面失效，或被当成子命令参数 | 明确「全局参数必须在子命令之前」，或把全局参数在每个子命令参数集里再注册一次 |
| `os.Exit` 绕过 `defer` | 输出缓冲未 Flush、临时文件未删除、日志未落盘 | 只在 `main` 里 `os.Exit`，其余层返回错误 |
| 错误只打日志不返回 | 上层拿到 nil，退出码变成 0 | 打日志与返回错误二选一，或都做但必须返回 |
| `internal` 路径写错 | 编译报「use of internal package not allowed」 | 导入路径必须包含 `internal` 的完整前缀，且导入方在其父目录树内 |
| Windows 换行 `\r\n` | 行尾多出 `\r`，字段比较失败、行数统计偏差 | 解析入口先做行尾归一化，并为此写一个专门用例 |
| stdin 被读两次 | 第二次读取得到 0 字节，结果看似「空文件」 | stdin 只能消费一次，需要复用就先读到内存或落临时文件 |

## 10. 参考实现要点

只给关键设计决策与骨架片段，不给完整成品。

- 退出码分类：让「用法错误」成为可识别的哨兵错误（如 `cli.ErrUsage`），`main` 中用 `errors.Is` 判断一次，命中即 `2`，其余非 nil 错误为 `1`，nil 为 `0`。
- `internal/logline` 的解析函数保持纯函数：`func Parse(line string) (Record, error)`，不读文件、不写输出、不记日志，错误链才可测。
- 三层包装，每层只加本层已知的上下文：

```go
return fmt.Errorf("parse field 2: %w", ErrMissingField) // 第 1 层：行解析
return fmt.Errorf("read %s: %w", path, err)             // 第 2 层：文件处理
return fmt.Errorf("run clean: %w", err)                 // 第 3 层：子命令
```

加分项 1 的并发骨架（`sync.WaitGroup.Go` 为 Go 1.25 新增）：结果切片按输入长度预分配，每个 goroutine 只写自己的下标，聚合阶段不必加锁。

```go
var wg sync.WaitGroup
for _, path := range paths {
	wg.Go(func() { /* 处理单个输入 */ })
}
wg.Wait()
```
