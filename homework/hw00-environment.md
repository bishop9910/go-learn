# 作业 00：开营准备——环境、工具链与基线自测

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W0 |
| 发放日期 | 2026-09-22 |
| 交付日期 | 2026-09-24 |
| 预计工时 | 3 小时 |
| 难度 | 入门 |
| 对应讲义 | L01 |
| 前置作业 | 无 |

## 1. 背景与目标

W0 只有 3 天，本作业不写业务代码，只做三件事：把后续 24 周要反复使用的环境固定下来、把练习仓库的目录与 module 边界一次定好、用一份可执行的基线自测暴露语法层面的知识缺口。

- 环境可复现：清楚本机工具链版本、模块缓存位置、代理与校验配置，并能逐项解释 `go env` 的含义。
- 目录可约定：练习仓库根为 `~/go-learn-work/`，每个作业一个子目录，每个子目录是独立 module。
- 基线可验证：slice 别名、map 零值、接口 nil、defer 求值时机、goroutine 泄漏五类易错语义，各有一个能复现现象的测试。
- 工具可用：静态检查与漏洞扫描在 W1（2026-10-08 复课）之前装好，或给出明确的补齐计划。

讲义参考 [L01 工具链与 Go Modules](../lectures/01-toolchain-and-modules.md)。

## 2. 需求（必须项）

**M1 环境校验。** 在 `~/go-learn-work/hw00/ENV.md` 中记录 `go version` 的完整输出，断言其中包含 `go1.25.7`；并执行 `go env GOPATH GOMODCACHE GOPROXY GOSUMDB GOFLAGS GOTOOLCHAIN`，把每一项的**本机取值**与**含义**写进下表（含义一列不得留空）。

| 变量 | 本机取值 | 含义（自己写，不超过两句） |
| --- | --- | --- |
| `GOPATH` | 待填 | 待填 |
| `GOMODCACHE` | 待填 | 待填 |
| `GOPROXY` | 待填 | 待填 |
| `GOSUMDB` | 待填 | 待填 |
| `GOFLAGS` | 待填 | 待填 |
| `GOTOOLCHAIN` | 待填 | 待填 |

**M2 目录约定。** 建立 `~/go-learn-work/` 作为练习仓库根；根目录**不**创建 module，每个作业一个子目录（`hw00`、`hw01`、…），每个子目录各自 `go mod init`，module 路径用 `example.com/golearn/<作业号>` 形式。提交一份 `tree` 输出（或目录结构截图）到 `hw00/TREE.txt`。

**M3 跑通 hello 模块。** 在 `~/go-learn-work/hw00/hello` 下创建 module，至少包含 `main.go`（打印一行固定文本）与一个 `_test.go`，并依次跑通下列命令，把每条命令的**原始输出**贴进 `hw00/hello/RUN.md`：

| 命令 | 期望结果 |
| --- | --- |
| `go mod init example.com/golearn/hw00/hello` | 生成 `go.mod`，其中 `go` 行不高于本机工具链 |
| `go build ./...` | 无输出、退出码 0 |
| `go run .` | 打印那一行文本 |
| `go test ./...` | 输出 `ok` 且退出码 0 |
| `go vet ./...` | 无输出、退出码 0 |
| `gofmt -l .` | 输出为空（有文件路径即未格式化） |

**M4 基线自测程序。** 在 `hello` module 内写 `baseline_test.go`，用 5 个子测试分别复现下列现象。每个子测试必须**先断言错误直觉会失败**，再断言正确写法通过，并在注释中写明「现象 / 根因 / 正确做法」三行。

| 编号 | 主题 | 必须复现的现象 |
| --- | --- | --- |
| B1 | slice 别名 | 对切片做切片后 `append`，原切片的可见内容被意外改写（底层数组共享） |
| B2 | map 零值 | 从 nil map 读取安全、向 nil map 写入 panic；零值 map 与已初始化 map 的行为差异 |
| B3 | 接口 nil | 把类型化的 nil 指针赋给接口后，接口自身不等于 nil |
| B4 | defer 求值时机 | 循环内 `defer` 的参数在注册时求值；对有命名返回值的函数，`defer` 能改写返回值 |
| B5 | goroutine 泄漏 | 向无缓冲/无人接收的 channel 发送后 goroutine 永久阻塞；用 `runtime`（<https://pkg.go.dev/runtime>）读取当前 goroutine 数量做前后对比 |

B5 只要求能稳定复现并说明泄漏原因，不要求写出通用的泄漏检测器。

**M5 工具安装。** 安装两个命令行工具，并把安装命令、版本输出、是否成功写进 `hw00/TOOLS.md`：

1. `golangci-lint`：按官方安装说明执行（<https://golangci-lint.run>）。配置文件的 v2 格式为 `version: "2"`，此格式与其他版本格式不兼容，不要把两份格式混写。
2. `govulncheck`：模块路径为 `golang.org/x/vuln/cmd/govulncheck`（以官方最新稳定版为准）。

如果网络受限装不上，**不要卡在这里**：在 `TOOLS.md` 中记录失败命令、完整报错、当时的 `GOPROXY` 取值，并写出替代方案（例如本阶段先用 `go vet ./...` 与 `gofmt -l .` 兜底）与 W1 补齐的具体日期。

**M6 学习日志。** 在 `~/go-learn-work/LEARNING-LOG.md` 写初版，表头固定为四列：

| 日期 | 学了什么 | 卡在哪 | 明天做什么 |
| --- | --- | --- | --- |
| 2026-09-22 | | | |

## 3. 需求（加分项）

1. 用 `go work`（Go 1.18 起可用）在 `~/go-learn-work/` 建工作区，把 `hw00` 下各 module 串起来，并说明工作区与「每个子目录独立 module」如何共存。
2. 记录 `GOTOOLCHAIN` 的四种取值语义（`local` / `auto` / `goX.Y.Z` / `goX.Y.Z+auto`），并用 `go env -w GOTOOLCHAIN=local` 把基线锁到本机工具链后重新跑一遍 M3 的六条命令，说明锁定的代价。
3. 用 `go version -m -json`（Go 1.25 工具变更）导出 `hello` 二进制的构建信息，把 `-json` 输出与默认输出做对比记录。
4. 用 `go doc -http`（Go 1.25 工具变更）在本地打开标准库文档，记录端口与访问结果。

## 4. 技术约束

- 语言基线：**Go 1.25**（本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`）。任何 Go 1.26 / 1.27 专属能力都不得在本作业中当作 1.25 可用。
- 只使用标准库与命令行工具，不引入第三方依赖（M5 的两个工具属于开发期工具，不算运行时依赖）。
- 源文件 UTF-8 编码；不要用 BOM。文件行尾 CRLF 与 LF 都可以，但 B4/B1 之外的解析类需求要在日志里说明你用的是哪一种。
- 命令示例：Go 命令跨平台，直接写即可；仅 PowerShell 专有的写法要单独标注。
- 所有命令的输出必须是**真实粘贴**，不允许手写「预期输出」充当结果。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `~/go-learn-work/LEARNING-LOG.md` | 学习日志（M6 表头，至少 3 行真实记录） |
| `~/go-learn-work/hw00/ENV.md` | `go version` 输出 + `go env` 六项含义表 |
| `~/go-learn-work/hw00/TREE.txt` | 目录结构 `tree` 输出或截图 |
| `~/go-learn-work/hw00/TOOLS.md` | 两个工具的安装过程、版本、或失败与补齐计划 |
| `~/go-learn-work/hw00/hello/go.mod` | hello 模块定义 |
| `~/go-learn-work/hw00/hello/main.go` | 最小可运行入口 |
| `~/go-learn-work/hw00/hello/baseline_test.go` | B1–B5 五个子测试 |
| `~/go-learn-work/hw00/hello/RUN.md` | M3 六条命令的原始输出 |

## 6. 验收标准（可执行命令）

以下命令均在 `~/go-learn-work/hw00/hello` 下执行，逐条给出实际输出：

```bash
go version                 # 输出包含 go1.25.7
go env GOTOOLCHAIN         # 输出 current setting 与可选值说明
go vet ./...               # 无输出，退出码 0
gofmt -l .                 # 输出为空
go test -v ./...           # B1–B5 五个子测试全部 PASS
```

附加人工验收：

- `ENV.md` 六项 `go env` 都有本机取值与含义，含义不能是复制粘贴的同一句话。
- `baseline_test.go` 每个子测试的注释都写了「现象 / 根因 / 正确做法」。
- `TOOLS.md` 中若标记为失败，必须同时存在替代方案与补齐日期。

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 环境校验完整 | 20 | `go version` 断言正确；六项 `go env` 取值真实、含义准确、无互相矛盾 |
| 模块跑通 | 20 | 目录约定符合 M2；M3 六条命令全部有原始输出；`gofmt -l .` 与 `go vet ./...` 干净 |
| 基线自测 5 题 | 40 | B1–B5 每项 8 分：能复现现象（4 分）+ 说明根因与正确做法（4 分） |
| 日志与目录规范 | 20 | 日志四列齐全且至少有 3 天记录（10 分）；`TREE.txt` 与交付物清单一致（10 分） |

## 8. 提示与思路

- 先跑命令、后写文档。`ENV.md` 里的「含义」一列建议用自己的话写，写完隔一天再读一遍，读不通就是没懂。
- B3 接口 nil 的关键是区分「接口的动态类型与动态值」两个字段，用 `fmt.Printf("%T %v", v, v)` 打印出来最直观。
- B4 的坑分两半：`defer` 的**参数**在注册时求值，而函数体在返回前才执行。请分别写一个循环内 `defer` 和一个命名返回值被 `defer` 修改的例子。
- B5 建议先写「发送到无缓冲 channel 但无人接收」的最小例子，再用 goroutine 数量前后对比。测试结束时用超时控制避免整个 `go test` 挂死。
- `go env -w` 的写入是持久的，改错值要能改回来；锁定 `GOTOOLCHAIN=local` 后如果某个模块要求更高版本会直接报错，这是预期行为，不要误判为环境损坏。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| Windows 路径与磁盘 | `GOPATH` 落在系统盘，模块缓存迅速占满；路径含空格导致个别工具异常 | 先看 `go env GOPATH`，必要时改到独立数据盘并用 `go env -w` 持久化 |
| 代理与校验打架 | `GOPROXY` 指向不可达地址时任何下载都失败，报错却像「模块不存在」 | 同时检查 `GOPROXY` 与 `GOSUMDB`，把当前取值写进 `TOOLS.md` 备查 |
| 用 IDE 代替命令行 | IDE 一键运行成功，但说不清 `go build` 与 `go run` 的差别 | 本作业所有结论必须来自命令行输出 |
| `gofmt -l .` 在错误目录执行 | 输出为空，误以为已格式化 | 先确认当前目录含 `go.mod`，再执行 |
| 忽略 CRLF | 后续作业里按行解析时多出 `\r`，测试直接失败 | 现在就在日志里记下自己仓库的行尾约定 |
| 泄漏测试拖死测试进程 | `go test` 长时间不返回 | 给可疑测试加超时，或把阻塞点改成可取消的等待方式 |

## 10. 参考实现要点

`hello/main.go` 只需要一个最小入口，重点是后面的自测：

```go
package main

import "fmt"

func main() { fmt.Println("hw00 hello") }
```

B4 的关键片段（另外四项按同样结构写成一个子测试表）：

```go
func TestDeferEvaluation(t *testing.T) {
	// 现象：循环里 defer 打印的全是同一个值
	// 根因：defer 的参数在注册时求值，不是执行时
	// 正确做法：用闭包捕获或把值作为参数显式传入
	t.Run("args evaluated at defer time", func(t *testing.T) { /* 略 */ })
	t.Run("named result modified by defer", func(t *testing.T) { /* 略 */ })
}
```

把 B1–B5 组织成子测试表，每项一条记录：主题、复现函数、根因注释。日志模板直接照 M6 四列建表，不要另起格式。
