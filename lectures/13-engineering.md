# L13 工程化：项目结构、配置、Lint 与 CI/CD

本篇定位：把「能跑的代码」变成「能被团队长期维护的工程」——项目布局与依赖方向、依赖注入、配置管理、日志约定、Lint/漏洞扫描工具链、任务编排、Git 与版本、CI 流水线、Docker 构建、十二要素落地。对应 W4–W6 与 W10。前置：熟悉 `go mod`、`go test`、`go build`，读过 [L01 工具链与模块](01-toolchain-and-modules.md)。

约定：第三方库与工具只给模块路径与官方链接，不写版本号，一律以官方最新稳定版为准；GitHub Actions 的 action 大版本以各仓库 Marketplace 最新为准。

## 1. 项目布局

```text
myservice/
├── cmd/
│   ├── api/main.go          # 一个可执行文件一个目录，main 只做装配
│   └── worker/main.go
├── internal/                # 只允许本模块内部导入
│   ├── handler/             # 协议层：HTTP/gRPC 入参与校验、错误映射
│   ├── service/             # 业务编排、事务边界
│   ├── repository/          # 存储访问（SQL/缓存/对象存储）
│   ├── domain/              # 领域模型与领域规则
│   └── config/              # 配置结构与校验
├── pkg/                     # 可被外部模块导入的公共库（可选，谨慎使用）
├── api/                     # 接口定义（proto/openapi）
├── migrations/  scripts/  testdata/
├── go.mod  go.sum  Makefile  Dockerfile  .golangci.yml
```

| 目录 | 语义 | 边界 |
| --- | --- | --- |
| `cmd/` | 可执行入口 | 只做装配与启动，不含业务逻辑 |
| `internal/` | 私有实现 | **编译期强制**：只有以 `internal` 的父目录为根的子树能导入 |
| `pkg/` | 对外公开库 | 一旦公开就要兼容，业务代码默认放 `internal/` |
| `api/` | 接口契约 | 生成代码与手写代码分目录 |

`internal` 的约束来自导入路径而不是配置：`example.com/mod/internal/x` 只能被 `example.com/mod/...` 下的代码导入，其他模块导入会直接编译失败。这就是「不承诺 API 稳定性」的机械保证，也是把业务实现藏在 `internal/` 的理由。同一模块内 `internal` 相互导入是允许的，因此它不是「模块隔离」而是「模块边界」。

**分层与依赖方向**：`handler → service → repository`，依赖只能指向内层，禁止反向。`repository` 不能 `import service`，`domain` 不依赖任何外层包（所以它不 import 数据库驱动、不 import HTTP 框架）。

```go
// internal/service/user.go
type UserRepo interface { // 接口定义在消费方
	Get(ctx context.Context, id string) (*domain.User, error)
}

type UserService struct{ repo UserRepo } // 依赖抽象，不依赖具体实现

func NewUserService(repo UserRepo) *UserService { return &UserService{repo: repo} }
```

**领域模型与 DTO 分离**：DTO 面向外部契约（`json` tag、校验规则、可加字段），领域模型面向业务规则（不变量、方法）。不要用同一个 struct 同时承担两者——改 API 字段会波及业务逻辑，反之亦然。转换写在边界层（handler 或专门的 `dto` 包）。

**接口定义在消费方还是提供方**：

| 位置 | 何时选它 | 代价 |
| --- | --- | --- |
| 消费方（Go 惯例） | 绝大多数情况：只声明自己用到的方法，便于测试打桩 | 同一实现在不同消费方会有多个小接口 |
| 提供方 | 契约必须统一发布、框架回调（如 `http.Handler`）、插件扩展点 | 接口偏大，容易随时间膨胀 |

默认「接受接口、返回结构体」：函数参数用接口，返回值用具体类型，调用方拿到能力而不是抽象。

## 2. 依赖注入

手写构造函数注入优先，`main` 就是装配清单：

```go
func main() {
	cfg, err := config.Load() // 启动即校验
	if err != nil {
		log.Fatalf("config: %v", err)
	}
	db, err := repository.Open(ctx, cfg.Database)
	if err != nil {
		log.Fatalf("db: %v", err)
	}
	repo := repository.NewUserRepo(db)
	svc := service.NewUserService(repo)
	h := handler.NewUserHandler(svc)
	// ... 启动 http.Server，关闭时按相反顺序释放资源 ...
}
```

原则：构造函数返回 `(T, error)`；依赖显式列出；不用包级全局变量藏依赖；不把「容器」当参数满世界传；装配顺序即依赖顺序，测试时直接传入假实现。

| 方案 | 模块路径 | 适用场景 |
| --- | --- | --- |
| 手写构造 | 标准库 | 默认选择，依赖图小、可读性最高 |
| wire | `github.com/google/wire` | 编译期生成装配代码，零运行时反射；依赖图大且希望构建期报错 |
| fx | `go.uber.org/fx` | 运行时依赖注入 + 生命周期钩子；模块化装配、多组件启停编排 |

`wire` 与 `fx` 均不写版本号，以官方最新稳定版为准；引入前先确认手写构造函数是否已经足够——多数服务的依赖图不超过二十个节点。

## 3. 配置管理

优先级设计（从高到低）：**命令行参数 > 环境变量 > 配置文件 > 内置默认值**。越靠近部署环境的来源优先级越高，便于临时覆盖与排障；敏感项只允许来自环境变量或密钥管理服务，不进配置文件、不进 git、不打进镜像。

启动即校验（fail fast）：`Load()` 一次性读取并 `Validate()`，任何不合法配置立刻退出并打印明确原因，不要等到第一次请求才发现 DSN 为空。

`os.LookupEnv` 区分「未设置」与「显式置空」：`ok == false` 表示未设置（可用默认值），`ok == true && v == ""` 表示调用方显式要求空值（不能默默替换成默认值）。

```go
package config

// import 省略：errors、fmt、os、time

type Config struct {
	Addr            string
	ShutdownTimeout time.Duration
	LogLevel        string
	Database        DatabaseConfig
}

type DatabaseConfig struct {
	DSN             string // 敏感项：只从环境变量读取
	MaxOpenConns    int
	ConnMaxLifetime time.Duration
}

func Load() (*Config, error) {
	c := &Config{
		Addr:            getenv("APP_ADDR", ":8080"),
		ShutdownTimeout: 10 * time.Second,
		LogLevel:        getenv("APP_LOG_LEVEL", "info"),
		Database: DatabaseConfig{
			DSN:             os.Getenv("APP_DB_DSN"),
			MaxOpenConns:    getenvInt("APP_DB_MAX_OPEN_CONNS", 20),
			ConnMaxLifetime: 30 * time.Minute,
		},
	}
	if err := c.Validate(); err != nil {
		return nil, err
	}
	return c, nil
}

func getenv(key, def string) string {
	if v, ok := os.LookupEnv(key); ok { // ok=false → 未设置；ok=true && v=="" → 显式空值
		return v
	}
	return def
}
// 同理需要 getenvInt 时，也用 LookupEnv 再 strconv.Atoi，并把解析失败当作配置错误返回

func (c *Config) Validate() error {
	var errs []error
	if c.Addr == "" {
		errs = append(errs, errors.New("APP_ADDR must not be empty"))
	}
	if c.Database.DSN == "" {
		errs = append(errs, errors.New("APP_DB_DSN is required"))
	}
	if c.Database.MaxOpenConns <= 0 {
		errs = append(errs, errors.New("APP_DB_MAX_OPEN_CONNS must be > 0"))
	}
	switch c.LogLevel {
	case "debug", "info", "warn", "error":
	default:
		errs = append(errs, fmt.Errorf("unknown APP_LOG_LEVEL %q", c.LogLevel))
	}
	return errors.Join(errs...) // Go 1.20
}
```

要点：`Validate` 返回聚合错误（`errors.Join`，Go 1.20）而不是遇到第一个问题就返回，运维一次就能看全；配置结构体只放「启动时需要确定」的值，热更新项走 L12 第 8 节的原子替换模式；配置的键名要在文档里集中列出，禁止散落在各处的魔法字符串。如需配置文件解析库，只给模块路径（如 `github.com/spf13/viper`，以官方最新稳定版为准），具体 API 以官方文档为准；能用一个 YAML/JSON 解码加环境变量覆盖解决的问题，不必引入框架。

## 4. 日志与可观测性的工程约定

详细规范见 [L03 错误处理与结构化日志](03-errors-and-logging.md) 与 L16，此处只讲落地约定：

| 约定 | 做法 |
| --- | --- |
| request_id 贯穿 | 入口生成或从上游头透传，存入 context，每条日志带同一 id，响应头回写 |
| 日志分级 | `error` 只记需要人介入的；`warn` 记可自愈的异常；`info` 记状态变更；`debug` 默认关闭 |
| 禁止敏感字段 | 密码、token、cookie、身份证、卡号、完整手机号一律不打；用字段白名单而非黑名单 |
| 字段命名统一 | `request_id`、`user_id`、`duration_ms`、`err`、`code` 全局一致 |
| 结构化输出 | 用 `log/slog`（Go 1.21）输出 JSON，禁止 `fmt.Sprintf` 拼日志正文 |
| 错误只记一次 | 顶层记录并带栈/上下文，中间层只包装不重复记，避免同一错误刷多条 |

工具链侧可用的新能力：**Go 1.25** 新增 `log/slog.GroupAttrs` 与 `slog.Record.Source`；**Go 1.26** 新增 `log/slog.NewMultiHandler`，`MultiHandler` 会依次调用所有 handler（适合同时输出到 stdout 与文件/采集端）。

## 5. 代码质量工具链

| 工具 | 作用 | 建议运行时机 |
| --- | --- | --- |
| `gofmt` / `go fmt` | 格式统一 | 保存时 / pre-commit；CI 用 `gofmt -l` 校验 |
| `go vet` | 官方静态检查 | 每次提交、CI |
| `golangci-lint` | 多 linter 聚合 | CI + 本地 |
| `staticcheck` | 更深层的正确性与简化建议 | CI（可经 golangci-lint 接入） |
| `govulncheck` | 漏洞可达性扫描 | CI 定时 + 发布前 |
| `go fix` | modernizers 批量现代化 | 升级 Go 版本后小步执行 |

`go vet` 的版本变化：**Go 1.25** 新增两个分析器——`waitgroup`（检查 `WaitGroup.Add` 位置错误）与 `hostport`（`fmt.Sprintf("%s:%d")` 拼地址在 IPv6 下不适用，建议改用 `net.JoinHostPort`）；**Go 1.27** 把 `waitgroup` 改名为 `waitgroupgo`，且 `go test` 默认执行 `stdversion` 检查（报告超出该文件生效 Go 版本的标准库符号），因此「在低版本 `go` 行下用了新标准库符号」会在测试阶段直接失败。

`.golangci.yml`（v2 配置，**键名与可用 linter 列表以 <https://golangci-lint.run> 官方文档为准**）：

```yaml
version: "2" # v2 配置必须以此开头，与 v1 不兼容

run:
  timeout: 5m
  tests: true

linters:
  default: standard
  enable:
    - staticcheck
    - errcheck
    - bodyclose
  exclusions:
    generated: lax
    rules:
      - path: _test\.go
        linters: [errcheck]

formatters:
  enable:
    - gofmt
```

v1 的配置在 v2 下会直接报错，升级时先读官方迁移说明，把「启用哪个 linter 集合、排除哪些路径」这两件事明确定下来，再逐个修告警——一次性打开全部 linter 会产生无法审阅的 diff。`staticcheck` 可作为独立命令运行，也可由 golangci-lint 托管，不要两处配置不同规则集。

`govulncheck` 的模块路径是 `golang.org/x/vuln/cmd/govulncheck`（见 <https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck>）：它不只列出依赖里的已知漏洞，而是结合调用图判断「有漏洞的函数是否真的可达」，显著降低噪音。CI 中推荐 `go run golang.org/x/vuln/cmd/govulncheck@latest ./...`。

**`go fix` 的现代化流程**：**Go 1.26** 把 `go fix` 完全重写为 modernizers 的入口（与 `go vet` 共用同一套 analysis 框架），支持 `//go:fix inline` 做源码级内联，旧的 fixer 全部移除；**Go 1.27** 新增 modernizers `atomictypes`、`embedlit`、`slicesbackward`、`unsafefuncs`，移除 `fmtappendf`，并把 `waitgroup` 分析器改名为 `waitgroupgo`。批量现代化的纪律：

```bash
# 1) 干净基线：确保测试与 lint 全绿，工作区无未提交改动
git status --short
# 2) 小步执行：一次只跑一个 modernizer，单独成一次提交，便于 review 与回滚
go fix ./...
go test ./...
# 3) 完成后统一 gofmt 并复查 diff，确认没有语义变化
gofmt -l . && git diff --stat
```

工具类依赖（代码生成器、linter）用 **Go 1.24** 起的 `go.mod` `tool` 指令固定版本（`go get -tool` 安装、`go tool` 运行），避免「本地能生成、CI 不能」；**Go 1.25** 起发行版减少了预编译工具二进制，非构建类工具由 `go tool` 按需构建运行。

## 6. 任务编排

`Makefile` 的常见目标：

```makefile
.PHONY: build test lint run gen docker
build:  ; go build -trimpath -o bin/app ./cmd/api
test:   ; go test -race -cover ./...
lint:   ; golangci-lint run && go vet ./...
run:    ; go run ./cmd/api
gen:    ; go generate ./... && go mod tidy
docker: ; docker build -t app:dev .
```

Windows 的现实问题：本机通常没有 GNU make（需额外安装），且 recipe 默认用 POSIX shell，`rm`、`cp`、`grep`、`$(shell ...)` 在纯 PowerShell 环境会失败。跨平台团队要么统一用 WSL/容器跑 `make`，要么用 PowerShell 脚本作为等价入口：

```powershell
param([Parameter(Mandatory)][ValidateSet('build','test','lint','run','gen','docker')][string]$Task)
$ErrorActionPreference = 'Stop'
switch ($Task) {
  'build'  { go build -trimpath -o bin\app.exe ./cmd/api }
  'test'   { go test -race -cover ./... }
  'lint'   { golangci-lint run; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }; go vet ./... }
  'run'    { go run ./cmd/api }
  'gen'    { go generate ./...; go mod tidy }
  'docker' { docker build -t app:dev . }
  default  { throw "unknown task: $Task" }
}
```

Taskfile（`github.com/go-task/task`，以官方最新稳定版为准）是另一个跨平台选择：任务写在 YAML 里，自身是单个二进制，不依赖 POSIX shell。

## 7. Git 与版本

| 主题 | 规范 |
| --- | --- |
| 提交粒度 | 一个提交一个逻辑变更；重构与功能分开；能通过测试再提交 |
| 提交信息 | 首行说明「做了什么」，正文说明「为什么」，必要时附影响面 |
| `go.mod` / `go.sum` | **必须一起提交**；只动 `go.sum` 的提交通常意味着依赖漂移，要查明原因 |
| `go` 行 | 团队统一的语言基线；**Go 1.26** 起 `go mod init` 默认写低一档版本，可用 `go get go@version` 调整 |
| tag | 发布的 tag 必须是 `vX.Y.Z` 形式，且与代码状态一致 |
| 模块路径 | 主版本 ≥ 2 时模块路径与包导入路径都要以 `/vN` 结尾，tag 形如 `vN.Y.Z` |
| 发布 | `go install example.com/mod/cmd/app@v1.2.3` 要求该版本已被 tag 且可被代理拉取 |
| 不可变性 | tag 一旦被代理缓存就无法真正撤回，打错只能发布新版本 |

要点：`/vN` 规则不是「tag 加个 N」，而是**模块路径本身带 `/vN`**（`example.com/mod/v2`），否则 `go install ...@v2.0.0` 会失败。预发布版本用 `v1.2.3-rc.1`。私有模块需要配置代理与凭据，见 <https://go.dev/ref/mod>。

## 8. CI/CD

最小可用流水线（GitHub Actions，`action 大版本以官方最新为准`）：

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod   # 工具链版本与 go.mod 对齐，避免各跑各的
          cache: true               # 缓存模块与构建缓存
      - name: gofmt
        run: |
          unformatted=$(gofmt -l .)
          test -z "$unformatted" || { echo "$unformatted"; exit 1; }
      - name: vet
        run: go vet ./...
      - name: lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
      - name: test
        run: go test -race -cover ./...
      - name: vulncheck
        run: go run golang.org/x/vuln/cmd/govulncheck@latest ./...
      - name: build
        run: CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o bin/app ./cmd/api
      - uses: actions/upload-artifact@v4
        with:
          name: app
          path: bin/app
```

逐段解释：`checkout` 取代码；`setup-go` 用 `go-version-file: go.mod` 保证 CI 工具链与本地一致，`cache: true` 缓存模块下载与构建缓存；`gofmt` 步骤用非空输出判定失败（`gofmt -l` 只列文件名）；`vet` 抓静态问题（1.25 起的 `waitgroup`/`hostport`、1.27 的 `waitgroupgo` 与 `stdversion`）；`lint` 跑聚合检查；`test` 用 `-race` 让竞态在 CI 暴露；`vulncheck` 做可达性漏洞扫描；最后产出静态二进制并上传为 artifact。

**`GOTOOLCHAIN` 的坑**：当 `go.mod` 的 `go` 行高于本机/CI 的 Go 版本时，`go` 命令默认会尝试**下载**对应工具链再执行。网络受限的 CI（离线镜像、只允许内网代理）会因此直接失败，报错文本里出现工具链下载相关提示。对策：

- `setup-go` 用 `go-version-file: go.mod`，让安装的版本与要求一致；
- 受限环境显式设 `GOTOOLCHAIN=local`，让版本不匹配时立刻报清晰错误，而不是长时间尝试下载；
- 把工具链预装进构建镜像，或用 `go.mod` 的 `toolchain` 行/`go get go@version` 统一团队版本；
- 提升 `go` 行属于团队级变更，要在同一提交里更新文档与镜像，避免「本地能构建、CI 不能」。

## 9. Docker 构建

多阶段 `Dockerfile`：

```dockerfile
FROM golang:1.25 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/api

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/app /app
USER nonroot:nonroot
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/app", "-health-check"]
ENTRYPOINT ["/app"]
```

要点：

| 项 | 原因 |
| --- | --- |
| 先 `COPY go.mod go.sum` 再 `go mod download` | 依赖不变时复用缓存层，改代码不必重下依赖 |
| `CGO_ENABLED=0` | 产出静态二进制，运行镜像不需要 libc |
| `-trimpath` | 去掉构建机绝对路径，构建可复现且不泄漏目录结构 |
| `-ldflags "-s -w"` | 去掉符号表与调试信息，体积更小 |
| `distroless/static:nonroot` | 无 shell、无包管理器，攻击面最小，且自带非 root 用户 |
| `USER nonroot:nonroot` | 不以 root 运行；用 alpine 时需自行 `adduser` |
| `HEALTHCHECK` | 供编排系统探活；distroless 无 shell，必须用二进制自带的健康检查子命令 |
| `.dockerignore` | 排除 `.git`、`bin`、`*.md`、`.env`、`testdata`，防止把密钥与无关文件送进构建上下文 |

`.dockerignore` 最小内容：`.git`、`.gitignore`、`bin/`、`*.md`、`.env*`、`testdata/`。

| 运行镜像 | 典型体积 | 适用 |
| --- | --- | --- |
| `golang:1.25`（直接当运行镜像） | 数百 MB ~ 1GB | 仅本地调试，不要上生产 |
| `alpine` | 约 10 MB 级（+ 二进制） | 需要 shell 排障、需要 `ca-certificates` |
| `gcr.io/distroless/static:nonroot` | 数 MB 级（+ 二进制） | 推荐：静态二进制、无 shell、非 root |

镜像体积的差异主要来自基础镜像；真正要控制的是「不要把编译工具链带进运行层」，以及在 CI 中固定基础镜像的 digest、用 `docker buildx` 产出多平台镜像。

## 10. 十二要素应用与 Go 的对应

| 要素 | 在 Go 服务里的落地 |
| --- | --- |
| 代码库 | 一个服务一个仓库（或 monorepo 内一个模块），用 `go.mod` 划边界 |
| 依赖 | 显式声明在 `go.mod`/`go.sum`，不依赖系统已装库 |
| 配置 | 环境变量优先，见第 3 节；敏感项走密钥管理 |
| 后端服务 | MySQL/Redis/MQ 都当可替换的附加资源，通过接口注入 |
| 构建、发布、运行 | CI 构建静态二进制 → 镜像 → 部署，三者严格分离 |
| 进程 | 无状态、可随时重启；优雅关闭见 [L11 并发原语与 context](11-concurrency-and-context.md) 第 6 节 |
| 端口绑定 | 服务自己监听端口（`:8080`），不依赖外部 Web 服务器 |
| 并发 | 横向扩展进程；容器感知 `GOMAXPROCS`（Go 1.25）避免容器内按宿主机核数调度 |
| 易处理 | 快速启动、随时可杀；用 `signal.NotifyContext` 处理终止信号 |
| 开发/生产一致 | 同一镜像跑各环境，差异只在配置；`-trimpath` 保证构建可复现 |
| 日志 | 写 stdout 的结构化日志，由平台采集，不做本地滚动文件 |
| 管理进程 | 迁移、一次性任务用独立 `cmd/` 入口或 Job，不塞进主进程启动逻辑 |

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 业务逻辑写进 `main.go` | 无法测试、无法复用 | 入口承担了装配以外的职责 | `main` 只做装配，逻辑放 `internal/service` |
| 把实现放 `pkg/` | 一次重构就要发大版本 | 误以为 `pkg/` 是「公共代码」 | 默认 `internal/`，确需共享再公开 |
| repository 反向依赖 service | 循环依赖、无法编译或无法测试 | 依赖方向设计错误 | 由消费方定义接口，依赖指向内层 |
| 用同一个 struct 兼作 DTO 与领域模型 | 改 API 破坏业务、校验规则泄漏 | 边界没有分离 | DTO 与领域模型分开，边界处转换 |
| 包级全局 `var db *sql.DB` | 测试相互污染、无法替换实现 | 隐式依赖 | 构造函数注入 |
| 启动时懒读配置 | 上线后才发现 DSN 为空 | 没有 fail fast | 启动即 `Load()` + `Validate()` |
| 用 `os.Getenv` 判断「是否设置」 | 显式空值被默认值覆盖 | 无法区分未设置与空值 | `os.LookupEnv` |
| 敏感配置写进配置文件或镜像 | 密钥泄漏 | 把配置当代码管理 | 只从环境变量/密钥管理读取 |
| 跳过 `go.sum` 提交 | CI 校验失败或依赖漂移 | 认为 `go.sum` 是生成物 | `go.mod` 与 `go.sum` 一起提交 |
| 模块 `/v2` 但路径不带 `/v2` | `go install ...@v2` 失败 | 误解 `/vN` 规则 | 模块路径与 tag 同时带版本 |
| CI 不固定 Go 版本 | 本地能过、CI 挂 | `go` 行升级后未同步 CI | `go-version-file: go.mod` |
| 受限网络里依赖工具链自动下载 | 构建卡住后失败 | `GOTOOLCHAIN` 默认行为 | 预装工具链或设 `GOTOOLCHAIN=local` |
| 一次开启全部 linter | 上千条告警、无人修 | 迁移策略错误 | 先定基线，按目录/规则分批收紧 |
| 运行镜像用 `golang` 基础镜像 | 镜像数百 MB、含编译器 | 没做多阶段构建 | builder + distroless/alpine 两阶段 |
| 容器以 root 运行 | 逃逸风险放大 | 默认用户是 root | `USER nonroot` 或自建用户 |
| `docker build` 不带 `.dockerignore` | 构建慢、密钥进上下文 | 忽略上下文体积 | 显式写 `.dockerignore` |
| 只用 `make` 且假设 POSIX shell | Windows 上任务跑不起来 | 忽略平台差异 | 提供 PowerShell 脚本或 Taskfile |

## 12. 动手练习

1. 为一个既有小项目重排目录：`cmd/internal/pkg` 三分，画出依赖方向图，确认没有任何反向依赖。
2. 为配置写 `Load()` + `Validate()`，用表驱动测试覆盖：未设置、显式空值、非法值、多个错误同时聚合。
3. 在项目里接入 `go vet`、`golangci-lint`（v2 配置）、`govulncheck`，把「首次全绿的基线」写进 README。
4. 用 `go fix` 对一个旧包做一次现代化（先跑一个 modernizer，单独提交），记录 diff 与测试结果。
5. 写一份 GitHub Actions 工作流，让它在 PR 上跑格式、vet、lint、`-race` 测试、漏洞扫描与构建。
6. 写多阶段 `Dockerfile` 与 `.dockerignore`，对比「直接 `FROM golang`」与「distroless」两种镜像的体积与运行用户。

## 13. 自检清单

- [ ] `main` 只做装配，业务逻辑在 `internal/`，且依赖方向单向。
- [ ] 我知道 `internal` 的可见性由导入路径决定，并把它用在了业务实现上。
- [ ] DTO 与领域模型分离，转换发生在边界层。
- [ ] 接口定义在消费方，返回值用具体类型。
- [ ] 所有依赖都通过构造函数显式注入，没有包级全局状态。
- [ ] 配置在启动时一次性加载并校验，失败立即退出；敏感配置只来自环境变量或密钥管理，日志里不会出现。
- [ ] 代码通过 `gofmt`、`go vet`、`golangci-lint`（v2 配置）与 `govulncheck`。
- [ ] 构建与测试任务在 Windows 与 Linux 上都有可执行入口。
- [ ] `go.mod` 与 `go.sum` 一起提交，`go` 行与 CI 工具链一致；发布 tag 与模块路径 `/vN` 规则一致，`go install pkg@version` 能成功。
- [ ] 镜像为多阶段构建、静态二进制、非 root、带健康检查，并有 `.dockerignore`。

## 14. 延伸阅读

- Go Modules 参考：<https://go.dev/ref/mod>；模块版本与 `/vN`：<https://go.dev/blog/v2-go-modules>
- `internal` 目录语义（`go` 命令文档）：<https://pkg.go.dev/cmd/go>
- 配置与日志标准库：<https://pkg.go.dev/os>、<https://pkg.go.dev/log/slog>
- golangci-lint 官方文档（v2 配置与迁移）：<https://golangci-lint.run>
- `govulncheck`：<https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck>
- wire：<https://pkg.go.dev/github.com/google/wire>；fx：<https://pkg.go.dev/go.uber.org/fx>
- GitHub Actions 上的 Go：<https://docs.github.com/actions>
- 十二要素应用：<https://12factor.net/zh_cn/>
- Go 1.25 发行说明：<https://go.dev/doc/go1.25>；Go 1.26：<https://go.dev/doc/go1.26>；Go 1.27：<https://go.dev/doc/go1.27>
- 书籍：《Go 程序设计语言》，Alan A. A. Donovan、Brian W. Kernighan，机械工业出版社
- 书籍：《演进式架构》，Neal Ford、Rebecca Parsons、Patrick Kua，中国电力出版社
