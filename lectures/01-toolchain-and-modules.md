# L01 工具链与 Go Modules（Go 1.25–1.27）

本篇定位：把「本机装了什么、模块从哪里来、二进制怎么产出」这条链路上的每一环讲清楚，让后续所有讲义中的代码都能被稳定复现。
对应学习周：W0（开营周）与 W1（残周）。
前置要求：能在命令行运行 `go`、会写 `package main` 与 `func main`，不要求写过模块。
基线：本机 `go1.25.7`，`GOOS=windows`，`GOARCH=amd64`；语言版本上界参考 Go 1.26 与 Go 1.27 的发布说明。

---

## 1. 版本策略

| 项目 | 值 |
| --- | --- |
| 本机工具链 | `go1.25.7` |
| 目标平台 | `GOOS=windows`、`GOARCH=amd64` |
| 语言版本下界 | Go 1.25（本机可编译的全部特性） |
| 语言版本上界 | Go 1.27（2026-08 发布的当前最新版本，需显式获取工具链） |

第一条纪律：**任何标注「Go 1.26+」「Go 1.27+」的特性，都不允许在只装了 1.25 工具链的环境里声称可以编译通过。**

### 1.1 `GOTOOLCHAIN` 的取值语义

Go 1.21 起 `go` 命令可以按需下载并切换工具链，开关就是 `GOTOOLCHAIN`。

| 取值                 | 语义                                             | 典型用途           |
| ------------------ | ---------------------------------------------- | -------------- |
| `local`            | 只用本机工具链；模块要求更高版本时直接报错，绝不下载                     | 离线环境、CI 镜像版本可控 |
| `auto`             | 默认行为。按 `go.mod` 的 `go` / `toolchain` 行判断，不足则下载 | 日常开发           |
| `go1.27.0`         | 固定使用该版本；模块要求更高则报错                              | 精确复现某个版本的行为    |
| `go1.27.0+auto`    | 下界是 `go1.27.0`，模块要求更高时允许继续下载升级                 | 团队约定「至少 1.27」  |
| `C:\Go\bin\go.exe` | 直接指向某个可执行文件                                    | 多版本手工管理        |

```powershell
go env GOTOOLCHAIN
go env -w GOTOOLCHAIN=local
```

```bash
go env GOTOOLCHAIN
go env -w GOTOOLCHAIN=local
```

下载工具链依赖 `GOPROXY` 可达。受限网络下把 `GOTOOLCHAIN` 设为 `local` 并保证本地版本足够高，比让 `go` 在构建中途去下载更可预测。

### 1.2 `go` 行与 `toolchain` 行

| 行 | 含义 | 是否强制 |
| --- | --- | --- |
| `go 1.25.0` | 该模块使用的**语言版本**，决定语言特性是否可用，也决定工具链按哪个版本的标准库符号做检查 | 强制 |
| `toolchain go1.27.0` | **建议使用**的最小工具链版本；不满足时按 `GOTOOLCHAIN` 决定下载还是报错 | 是建议，不是语言约束 |

一个模块可以写 `go 1.25.0` 加 `toolchain go1.27.0`，含义是「语言按 1.25 规则，但请用 1.27 的工具链构建」。

**Go 1.25 起，更新 `go` 行不再自动写入 `toolchain` 行。** 此前提高 `go` 行时会顺带补一行 `toolchain`，导致提交里出现作者没打算引入的工具链要求；现在两行各自独立，需要固定工具链时手动添加。

### 1.3 Go 1.26 起 `go mod init` 默认写较低版本

| 创建者 | 写入的 `go` 行 |
| --- | --- |
| `1.N.X` 正式版工具链 | `go 1.(N-1).0` |
| 预发布版工具链 | `go 1.(N-2).0` |

原因：`go` 行是**下界声明**。新模块若一上来就写 `go 1.27.0`，任何还在 1.26 上的同事、CI 镜像、下游库都会被强制拉工具链，而模块本身可能压根没用 1.27 的任何特性。降低一档可最大限度保持可构建性，需要新特性时再显式抬高：

```bash
go get go@1.27.0
go mod tidy
```

---

## 2. 模块基础

### 2.1 常用子命令

| 命令 | 作用 |
| --- | --- |
| `go mod init <module-path>` | 在当前目录创建 `go.mod` |
| `go mod tidy` | 增补缺失依赖、删除未使用依赖，并同步 `go.sum` |
| `go mod download` | 把依赖下载进模块缓存，不改 `go.mod` |
| `go mod verify` | 校验本地缓存内容与 `go.sum` 记录是否一致 |
| `go mod vendor` | 把依赖复制到 `vendor/`，之后构建默认使用 `vendor/` |
| `go mod edit` | 用命令行方式改 `go.mod`（脚本友好） |
| `go mod graph` / `go mod why <pkg>` | 打印依赖图 / 解释某包为何被引入 |
| `go list -m all` | 列出构建列表中的所有模块及选中版本 |

```bash
go mod edit -go=1.25.0
go mod edit -require=example.com/lib@v1.4.0
go mod edit -replace=example.com/lib=../lib
go mod edit -dropreplace=example.com/lib
```

### 2.2 `go.mod` 指令一览

| 指令 | 语义 | 引入版本 |
| --- | --- | --- |
| `module` / `go` | 声明模块路径与语言版本下界 | 1.11 |
| `require` | 声明依赖及其版本 | 1.11 |
| `replace` | 把某个模块版本替换为另一个路径/版本/本地目录 | 1.11 |
| `exclude` | 从版本选择中排除某个版本 | 1.11 |
| `retract` | 声明本模块的某些版本不应当被使用 | 1.16 |
| `toolchain` | 建议的工具链版本 | 1.21 |
| `tool` | 声明本项目使用的工具依赖（**Go 1.24 新增**） | 1.24 |
| `ignore` | 声明在包模式匹配中忽略的目录（**Go 1.25 新增**） | 1.25 |

`ignore` 解决的是「仓库里有不参与构建的目录，但 `./...` 会把它们扫进来」：

```text
module example.com/orders

go 1.25.0

ignore ./testdata/generated
ignore ./third_party
```

Go 1.25 同时新增了 `work` 包匹配模式，用于在工作区范围内做包匹配。

### 2.3 `go.sum` 的作用

`go.sum` 记录每个模块内容与 `go.mod` 文件的哈希，作用有两个：**可重复构建**（同一份 `go.mod` 在任何机器上解析出的依赖内容逐字节一致）与**防篡改**（代理返回的内容与记录不符时构建失败，而不是静默使用）。

- `go.sum` **必须提交**，不得加入 `.gitignore`。
- 不要手工编辑；用 `go mod tidy` 或 `go mod download` 让它收敛。
- `go mod verify` 只校验**本地缓存**，不联网。

### 2.4 语义化导入版本（`/v2`）

模块版本 `v2.0.0` 及以上时，**模块路径必须带主版本后缀**：

| 版本 | 模块路径 | 导入路径 |
| --- | --- | --- |
| `v0.9.0`、`v1.4.2` | `example.com/lib` | `example.com/lib` |
| `v2.0.0` | `example.com/lib/v2` | `example.com/lib/v2` |
| `v3.1.0` | `example.com/lib/v3` | `example.com/lib/v3` |

`v0` 与 `v1` 都不写后缀，因此 `v1` 是兼容承诺的起点。这条规则让同一进程里可以同时存在 `v1` 与 `v2` 两个不兼容版本。

---

## 3. 工作区：`go work`

本地同时改多个模块（阶段四多服务联调的标准形态）时，用工作区把模块关联起来，避免把本地路径 `replace` 提交进各模块的 `go.mod`。

```bash
go work init ./orders ./users ./gateway
go work use ./billing
go work sync
```

| 命令 | 作用 |
| --- | --- |
| `go work init [dirs...]` | 在当前目录创建工作区文件 |
| `go work use [dirs...]` | 把模块加入工作区 |
| `go work sync` | 把工作区的版本选择结果同步回各模块的 `go.mod` |

规则：`go.work` 通常不提交（它绑定本地路径）；处于工作区中时按 `use` 列表解析模块，各模块仍各自维护 `go.sum`；用 `GOWORK=off` 可临时关闭工作区。

---

## 4. 工具依赖：`tool` 指令（Go 1.24）

写代码时依赖第三方库用 `require`；**构建期依赖某个命令行工具**（生成器、linter）此前没有干净的表达方式，只能靠 `tools.go` 加 blank import 骗过 `go mod tidy`。

```go
//go:build tools

package tools

import (
	_ "golang.org/x/vuln/cmd/govulncheck"
)
```

新方案：

```bash
go get -tool golang.org/x/vuln/cmd/govulncheck
go tool govulncheck ./...
```

`go.mod` 中出现 `tool golang.org/x/vuln/cmd/govulncheck`。

| 维度 | `tools.go` + blank import | `tool` 指令（1.24+） |
| --- | --- | --- |
| 依赖声明 | 靠伪造的 import | 显式 `tool` 行 |
| 占位文件 | 需要 | 不需要 |
| 运行方式 | `go run <module-path>` | `go tool <名字>` |
| `go mod tidy` 行为 | 容易误删 | 正确保留 |

第三方工具只保证**模块路径**正确，版本以官方最新稳定版为准，不要照抄写死的版本号。

---

## 5. 构建与运行

| 命令 | 用途 | 备注 |
| --- | --- | --- |
| `go run ./cmd/orders` | 编译并立即运行 | 适合快速验证 |
| `go build -o bin/orders ./cmd/orders` | 产出可执行文件 | 生产构建入口 |
| `go install example.com/tool@latest` | 编译并安装到 `GOBIN` | 需要模块路径与版本 |
| `go test ./...` | 运行测试 | Go 1.27 起默认执行 `stdversion` vet 检查 |
| `go vet ./...` | 静态检查 | Go 1.25 新增 `waitgroup` 与 `hostport` 分析器 |
| `go fmt ./...` | 格式化 | 提交前必跑 |
| `go doc fmt.Errorf` | 查看文档 | — |
| `go version -m bin/orders` | 查看二进制的模块信息 | Go 1.25 起支持 `-m -json` |
| `go fix ./...` | 现代化改写 | Go 1.26 起重写为 modernizers 入口 |

Go 1.25 新增的两个 `vet` 检查：`waitgroup` 捕获 `WaitGroup.Add` 调用位置错误；`hostport` 捕获用字符串拼接地址（`fmt.Sprintf("%s:%d")`）的写法，IPv6 下不成立，应改用 `net.JoinHostPort`。

Go 1.27 起 `stdversion` 分析器成为 `go test` 的默认检查项，它报告「使用了超出该文件生效 Go 版本的标准库符号」——这正是本系列讲义反复标注版本的原因。Go 1.27 的 `go fix` 新增了 `atomictypes`、`embedlit`、`slicesbackward`、`unsafefuncs` modernizers，移除了 `fmtappendf`；Go 1.26 已移除旧的全部 fixer。

`go doc` 的版本差异：

| 版本 | 能力 |
| --- | --- |
| 1.25 | `go doc -http`，在本地起一个文档服务 |
| 1.27 | `go doc package@version` 查看指定版本；`-ex` 显示示例代码 |

Go 1.26 移除了 `cmd/doc` 与 `go tool doc`，统一用 `go doc`。

`go version -m bin/orders` 会打印二进制的 Go 版本、模块路径与构建选项（形如 `path` / `mod` / `build -trimpath` 若干行），字段以本机实际输出为准；Go 1.25 起加 `-json` 可得到等价的机器可读输出，适合在 CI 里做产物审计。

---

## 6. 编译选项与交叉编译

| 选项 | 作用 |
| --- | --- |
| `-trimpath` | 去掉二进制中的本地绝对路径，构建可复现、不泄漏目录结构 |
| `-ldflags "-s -w"` | 去掉符号表与 DWARF 调试信息，显著减小体积 |
| `-race` | 开启数据竞争检测，运行时开销大，只用于测试环境 |
| `-gcflags` | 传给编译器的参数，如 `-gcflags="-m"` 查看逃逸分析结论 |
| `-tags` | 传入构建标签 |

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -ldflags "-s -w" -o bin/orders-linux-amd64 ./cmd/orders
```

```powershell
$env:CGO_ENABLED="0"; $env:GOOS="linux"; $env:GOARCH="amd64"
go build -trimpath -ldflags "-s -w" -o bin/orders-linux-amd64 ./cmd/orders
Remove-Item Env:GOOS, Env:GOARCH, Env:CGO_ENABLED
```

| 变量 | 说明 |
| --- | --- |
| `GOOS` | 目标操作系统，如 `linux`、`windows`、`darwin` |
| `GOARCH` | 目标架构，如 `amd64`、`arm64` |
| `CGO_ENABLED` | 是否启用 cgo；交叉编译到非本机平台时通常设为 `0` |

`-race` 要求 `CGO_ENABLED=1`，因此不能与「交叉编译到其他平台」同时使用。

---

## 7. 代理与校验

| 变量 | 作用 | 常用值 |
| --- | --- | --- |
| `GOPROXY` | 模块代理列表，逗号分隔，`direct` 表示直连，`off` 表示禁止下载 | `https://proxy.golang.org,direct` |
| `GOSUMDB` | 校验和数据库，用于验证新模块的哈希 | `sum.golang.org`，或 `off` |
| `GOPRIVATE` | 逗号分隔的模块路径前缀/通配，命中的模块同时跳过代理与校验和数据库 | `example.com/*` |
| `GOFLAGS` | 传给所有 `go` 子命令的默认参数 | `-mod=readonly` |

```bash
go env -w GOPROXY=https://proxy.golang.org,direct
go env -w GOSUMDB=sum.golang.org
go env -w GOPRIVATE=example.com/*
go env -w GOFLAGS=-mod=readonly
```

私有仓库域名放进 `GOPRIVATE`，`go` 就不会把它发给公共代理，也不会拿去查公共校验和库。`GOFLAGS` 是全局持久配置，也是「下载失败」类问题最常见的隐形来源；`go env -u GOFLAGS` 可撤销。

---

## 8. 依赖图与漏洞

```bash
go mod graph                      # 完整依赖图（边为 "A B@v"）
go list -m all                    # 构建列表中的全部模块与选中版本
go list -m -u all                 # 标注可升级的模块
go mod why example.com/lib/sub    # 解释某包为什么被引入
go mod why -m example.com/lib     # 解释某模块为什么被引入
```

漏洞扫描用第三方工具 `govulncheck`：

```bash
go get -tool golang.org/x/vuln/cmd/govulncheck
go tool govulncheck ./...
```

它的模块路径是 `golang.org/x/vuln/cmd/govulncheck`，属于 Go 官方维护但**不是标准库**，版本以官方最新稳定版为准。

---

## 9. 完整示例：从零建立多包模块

```text
orders/
├── go.mod
├── cmd/orders/main.go
└── internal/pricing/pricing.go
```

`internal/` 的语义由工具链强制：该目录下的包只能被以其父目录为根的子树中的代码导入，外部模块无法导入。

`go.mod`：

```text
module example.com/orders

go 1.25.0
```

`internal/pricing/pricing.go`：

```go
package pricing

import "errors"

// ErrEmptyItems 表示传入的条目列表为空。
var ErrEmptyItems = errors.New("pricing: empty items")

// Item 描述一个计价条目，Price 以分为单位，避免浮点误差。
type Item struct {
	Name     string
	Price    int64
	Quantity int
}

// Total 返回所有条目的总价（单位：分）。
func Total(items []Item) (int64, error) {
	if len(items) == 0 {
		return 0, ErrEmptyItems
	}
	var sum int64
	for _, it := range items {
		sum += it.Price * int64(it.Quantity)
	}
	return sum, nil
}
```

`cmd/orders/main.go`：

```go
package main

import (
	"fmt"
	"os"

	"example.com/orders/internal/pricing"
)

func main() {
	items := []pricing.Item{
		{Name: "keyboard", Price: 19900, Quantity: 1},
		{Name: "cable", Price: 1900, Quantity: 3},
	}

	total, err := pricing.Total(items)
	if err != nil {
		fmt.Fprintln(os.Stderr, "compute total:", err)
		os.Exit(1)
	}

	fmt.Printf("total(cents)=%d\n", total)
}
```

```bash
mkdir orders && cd orders
go mod init example.com/orders
mkdir -p cmd/orders internal/pricing   # 写入上面的两个 .go 文件
go mod tidy
go vet ./... && go fmt ./...
go run ./cmd/orders
go build -trimpath -ldflags "-s -w" -o bin/orders.exe ./cmd/orders
go version -m bin/orders.exe
```

预期输出 `total(cents)=25600`。

---

## 10. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `GOFLAGS` 留有 `-mod=vendor` 但仓库无 `vendor/` | `go build` 报找不到 vendor 目录 | `GOFLAGS` 是全局持久配置，容易忘记 | `go env GOFLAGS` 检查；`go env -u GOFLAGS` 清除或执行 `go mod vendor` |
| `GOPROXY=off` 或代理不可达 | 下载依赖报 EOF / connection refused | 网络策略与配置不一致 | `go env GOPROXY` 确认；受限网络配可达代理，或依赖全走 `vendor/` |
| 手工编辑 `go.sum` 或手工合并冲突 | `missing go.sum entry` / `checksum mismatch` | 破坏了哈希记录 | `git checkout go.sum && go mod tidy`；永不手工编辑 |
| 本地 `replace=../lib` 忘记删除就提交 | 别人克隆后构建失败，或发布版本不生效 | `replace` 会被完整带进模块 | 本地联调用 `go.work`；提交前 `git diff go.mod` 检查 |
| 有 `toolchain go1.27.0` 但 CI 镜像只有 1.25 | CI 在构建中途下载工具链失败或超时 | `toolchain` 行触发按需下载 | CI 显式设 `GOTOOLCHAIN=local` 并升级镜像，或去掉 `toolchain` 行 |
| `go get lib@latest` 顺手升级 | 引入非预期甚至不兼容的版本 | `@latest` 不受当前版本约束 | 指定版本：`go get lib@v1.4.2` |
| 把 `go.sum` 加进 `.gitignore` | 各机器能构建但内容可能不同 | 丢失了可重复构建的锚点 | `go.sum` 必须提交 |
| `go mod tidy` 在错误的目录执行 | 删掉别的模块的依赖 | 多模块仓库中未切到目标目录 | 进到含 `go.mod` 的目录再执行 |
| `ignore` 写成绝对路径 | 目录仍被 `./...` 扫到 | `ignore` 使用相对于模块根的路径模式 | 用 `./` 开头的相对路径，改完用 `go list ./...` 验证 |
| 有 `vendor/` 时改 `go.mod` 却不重新 vendor | 构建使用旧代码 | vendor 模式下依赖以目录内容为准 | 改完依赖后重新 `go mod vendor` |

---

## 11. 动手练习

1. **版本实验**：把 `GOTOOLCHAIN` 依次设为 `local`、`auto`，在 `go.mod` 中写 `go 1.27.0`，记录两次 `go build` 的差异。
2. **go.sum 破坏与恢复**：手工改动 `go.sum` 中一行，运行 `go mod verify` 与 `go build` 并记录现象，再用 `go mod tidy` 恢复。
3. **多包模块**：把第 9 节示例扩为三层 `cmd/orders` → `internal/service` → `internal/pricing`，让 `service` 返回带上下文的错误（L03 展开）。
4. **工作区联调**：再建模块 `example.com/users`，用 `go work` 让 `orders` 导入其本地代码；然后删除 `go.work`，观察构建失败并说明原因。
5. **工具依赖**：用 `go get -tool golang.org/x/vuln/cmd/govulncheck` 固定漏洞扫描器，再写出 `tools.go` 旧方案做对照。
6. **交叉编译**：为 `linux/amd64` 与 `windows/amd64` 各产出一个带 `-trimpath -ldflags "-s -w"` 的二进制，比较体积并解释差异。

---

## 12. 自检清单

- [ ] 能说清 `GOTOOLCHAIN` 四个取值（`local` / `auto` / `go1.27.0` / `go1.27.0+auto`）的差异
- [ ] 能区分 `go` 行与 `toolchain` 行的语义，并知道 Go 1.25 起二者不再联动
- [ ] 知道 Go 1.26 起 `go mod init` 写入较低版本，会用 `go get go@version` 调整
- [ ] `require` / `replace` / `exclude` / `retract` / `ignore` / `tool` 各能说出一个使用场景
- [ ] 能解释 `go.sum` 为什么不进 `.gitignore`，并知道 `go mod verify` 只校验本地缓存
- [ ] 能写出 `/v2` 模块的模块路径与导入路径
- [ ] 能用 `go work init/use/sync` 搭建多模块本地联调
- [ ] 能用 `tool` 指令管理工具依赖，并说明它相对 `tools.go` 的优势
- [ ] 会用 `-trimpath` 与 `-ldflags "-s -w"`，知道 `-race` 与交叉编译不能同时使用
- [ ] `GOPROXY` / `GOSUMDB` / `GOPRIVATE` / `GOFLAGS` 各能说出作用与常见误配
- [ ] 能用 `go mod why` 与 `go list -m all` 定位「这个依赖是谁引进的」
- [ ] 能独立从零建立、构建、审计一个多包模块

---

## 13. 延伸阅读

- Go 1.25 发布说明：<https://go.dev/doc/go1.25>
- Go 1.26 发布说明：<https://go.dev/doc/go1.26>
- Go 1.27 发布说明：<https://go.dev/doc/go1.27>
- 模块参考：<https://go.dev/ref/mod>
- 工具链管理：<https://go.dev/doc/toolchain>
- `govulncheck`：<https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck>
- 书籍：*Go 语言设计与实现*，左书祺，人民邮电出版社
- 相关讲义：[L02 语言精要与易错点](02-language-essentials.md)、[L03 错误处理与结构化日志](03-errors-and-logging.md)
