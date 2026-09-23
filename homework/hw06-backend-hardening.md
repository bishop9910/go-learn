# 作业 06：阶段二收口——Docker、接口文档、集成测试与 README

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W6 |
| 发放日期 | 2026-11-09 |
| 交付日期 | 2026-11-15 |
| 预计工时 | 18 小时 |
| 难度 | 进阶 |
| 对应讲义 | L13、L07 |
| 前置作业 | hw05 |

## 1. 背景与目标

前面的作业交付了功能，本作业交付「可复现的工程」：把 hw05 的服务收口成别人一条命令就能跑起来、且出错能自动发现的项目——配置启动即校验、依赖显式注入、镜像小而安全、文档与实现一致、集成测试真实可跑、质量门禁进 CI。完成本作业后你应能：

- 用多阶段构建交出体积更小、无构建工具链、以非 root 运行的镜像。
- 解释为什么 `depends_on` 不等于「依赖已就绪」，并用健康检查解决启动竞态。
- 用可执行的检查手段保证接口文档与实现不漂移，并把 lint、vet、测试、漏洞扫描与镜像构建串成 CI 门禁。

## 2. 需求（必须项）

**配置与依赖注入**

- **M1** 用统一的 `Config` 结构体承载配置（至少分组：server、database、redis、auth、log），提供 `Validate()` 并在启动时「先校验后监听」：任何缺失或非法字段立即退出，错误信息必须点名具体字段，不得等到第一个请求才失败（fail fast）。
- **M2** README 给出环境变量优先级表（例如默认值 < 配置文件 < 环境变量 < 命令行参数；顺序可自定但必须写明并实现一致）。
- **M3** 敏感配置（数据库口令、JWT 密钥等）只能从环境变量读取；仓库中不得出现真实值，只提供 `.env.example`。
- **M4** 依赖注入用手写构造函数逐层传入，`cmd/server/main.go` 只负责组装、启动与优雅关闭（收到信号 → 停止接收新请求 → 等待在途请求 → 关闭连接池）；禁止包级可变全局单例。

**容器化**

- **M5** 多阶段 `Dockerfile`：构建阶段编译，运行阶段只保留二进制与必要文件；编译必须带 `CGO_ENABLED=0`、`-trimpath`、`-ldflags "-s -w"`。
- **M6** 运行阶段使用非 root 用户；基础镜像不得使用 `latest` 标签，须固定到具体版本或摘要。
- **M7** `.dockerignore` 至少排除 VCS 目录、本地构建产物、`.env` 与测试缓存。
- **M8** `compose.yaml` 编排 app + MySQL + Redis + 迁移任务；app 必须等到依赖真正就绪才启动（健康检查 + 重试），不得只依赖启动顺序。
- **M9** README 给出「裸镜像 vs 多阶段镜像」体积对比表（镜像、TAG、大小、测量命令、结论）。

**接口文档**

- **M10** 用 OpenAPI/Swagger 描述接口：可选 `swaggo/swag` 或 `github.com/oapi-codegen/oapi-codegen`（模块路径，版本以官方最新稳定版为准），也可手写 `openapi.yaml`。
- **M11** 文档必须准确：每个接口的请求与响应 schema、可能的错误码、示例值、鉴权标注；**公开接口不得标注需要鉴权**。
- **M12** 必须提供「文档与实现一致」的校验手段（脚本或逐条比对记录均可），至少覆盖两点：路由集合一致、鉴权标注一致；比对结论写进 README。

**集成测试**

- **M13** 用 `net/http/httptest`（见 <https://pkg.go.dev/net/http/httptest>）配合真实 MySQL 与 Redis 跑核心链路：注册 → 登录 → 建文章 → 列表 → 详情 → 更新 → 删除。依赖可用 docker compose 或 `github.com/testcontainers/testcontainers-go`（模块路径，版本以官方最新稳定版为准）提供。
- **M14** `go test -race ./...` 全绿；`go test -cover ./...` 覆盖率不低于 60%，README 说明未覆盖部分与理由。

**质量门禁与 CI**

- **M15** 本地门禁全部通过：`golangci-lint` 配置以 `version: "2"` 开头（v2 配置格式见 <https://golangci-lint.run>）、`go vet ./...` 无输出、`govulncheck ./...`（`golang.org/x/vuln/cmd/govulncheck`）无已知漏洞、`gofmt -l .` 输出为空。
- **M16** GitHub Actions 工作流依次执行 lint、vet、`test -race`、govulncheck、构建镜像，Go 版本用 `go-version-file: go.mod` 指定。
- **M17** README 必须写明 `GOTOOLCHAIN` 在 CI 中可能尝试下载工具链的坑与规避方式（固定工具链或显式允许下载，见 <https://go.dev/doc/toolchain>），并给出本项目的选择与理由。

**README**

- **M18** README 必须包含：架构图（`text` 或 mermaid）、目录结构说明、启动步骤、环境变量表、接口一览、验收命令、已知限制。

## 3. 需求（加分项）

- **B1** 同时产出 distroless/scratch 与 alpine 两个运行阶段并给出体积与可调试性对比。
- **B2** CI 启用依赖与构建缓存，并记录启用前后的工作流耗时。
- **B3** 用 compose profile 分离「仅依赖（本地开发时用宿主机跑 app）」与「全栈」两种启动方式，并把覆盖率与测试结果作为 CI 产物上传，覆盖率下降时让工作流失败。

## 4. 技术约束

| 编号 | 约束 |
| --- | --- |
| C1 | Go 1.25 工具链；容器内 Go 版本由构建阶段镜像决定，与 `go.mod` 一致 |
| C2 | 二进制启动不得依赖工作目录下的相对路径文件，配置全走环境变量 |
| C3 | 迁移由独立步骤执行，不得在 app 启动时隐式迁移 |
| C4 | 文档中的错误码集合必须与实现中的错误码常量一一对应 |
| C5 | 不得提交 `.env`、密钥、本地绝对路径、测试缓存与构建产物 |
| C6 | 基础镜像若不含时区数据与 CA 证书，必须显式安装并在文档说明影响 |
| C7 | CI 中的镜像构建不得依赖宿主机的本地缓存（必须能在干净 runner 上完成） |

## 5. 交付物清单

| 交付物 | 要求 |
| --- | --- |
| `Dockerfile` | 多阶段、`CGO_ENABLED=0`、`-trimpath`、`-ldflags "-s -w"`、非 root |
| `compose.yaml` | app + MySQL + Redis + 迁移，含健康检查与依赖就绪等待 |
| `.dockerignore` / `.env.example` | 排除规则齐全；示例变量可读且无真实值 |
| 接口文档 | OpenAPI 文件或注释源，含示例值与鉴权标注 |
| CI 工作流 | lint + vet + test -race + govulncheck + 镜像构建 |
| 集成测试 | 可本地与 CI 重复执行，覆盖 M13 链路 |
| README | 架构图、目录结构、启动步骤、环境变量表、接口一览、验收命令、已知限制 |
| 体积对比 | 裸镜像 vs 多阶段镜像的大小与测量命令 |

## 6. 验收标准（可执行命令）

| 命令 / 检查 | 通过标准 |
| --- | --- |
| `docker compose up -d` | app、MySQL、Redis、迁移全部就绪；`curl` 健康检查返回 200 |
| `go test -race ./...` | 全部通过，无数据竞争 |
| `go test -cover ./...` | 覆盖率不低于 60%，README 列出未覆盖部分 |
| `golangci-lint run` | 无 error 级问题；配置以 `version: "2"` 开头 |
| `govulncheck ./...` | 无已知漏洞（`golang.org/x/vuln/cmd/govulncheck`） |
| `gofmt -l .` | 输出为空 |
| `docker build -t app:verify .` | 构建成功；容器内进程以非 root 身份运行（用 `docker run --rm app:verify <查看身份的只读命令>` 证明） |
| 文档一致性检查 | 路由集合与鉴权标注全部与实现一致，比对结论记入 README |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 容器化 | 25 | 多阶段与编译参数正确；非 root；镜像标签固定；compose 依赖就绪可靠；体积对比有数据 |
| 文档准确性 | 20 | schema、错误码、示例值、鉴权标注与实现一致；公开接口无鉴权标注 |
| 集成测试 | 20 | 真实 MySQL/Redis 上跑通核心链路；`-race` 干净；覆盖率达标且有说明 |
| 工程结构与配置 | 20 | `Config` 启动即校验；优先级明确；构造函数注入；main 只做组装与生命周期 |
| 质量门禁与 CI | 15 | 四道本地门禁与 CI 全绿；处理了 `GOTOOLCHAIN` 的坑 |

## 8. 提示与思路

- **先让「一条命令起来」成立**：把 compose 与迁移跑通再优化镜像体积，顺序反了会在构建脚本上反复返工。
- **健康检查要区分两种**：容器自身的存活检查，与「依赖已就绪」的探针；前者用于重启，后者用于等待。
- **文档一致性用脚本兜底**：把路由注册处的路径与方法导出为清单，与 OpenAPI 文件比对，比人工逐条看可靠；覆盖率先覆盖核心链路（认证、文章 CRUD、缓存降级），再补边缘分支。
- **CI 与本地用同一套命令**：README 的验收命令原文照抄进工作流，避免两套标准。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 基础镜像用 `latest` | 今天能构建，明天行为变了 | 依赖浮动标签 | 固定到具体版本或摘要 |
| `CGO_ENABLED=1` | distroless/scratch 里跑不起来 | 动态链接依赖宿主机库 | 静态编译并隔离构建阶段 |
| 时区与 CA 证书缺失 | 日志时间错 8 小时、HTTPS 校验失败 | 最小镜像不含时区数据与根证书 | 显式安装或挂载，并在文档说明 |
| `depends_on` 当成就绪 | app 启动时连不上数据库而退出 | 只保证容器启动顺序，不保证服务可用 | 健康检查 + 启动重试/退避 |
| Swagger 注释与实现漂移 | 文档描述的字段或错误码已不存在 | 手工维护、无一致性校验 | 加一致性检查步骤，纳入验收 |
| 把 `.env` 提交进仓库 | 口令与密钥泄露 | 缺少忽略规则与审查习惯 | `.gitignore` + `.dockerignore` + 只留 `.env.example` |

## 10. 参考实现要点

- **多阶段 `Dockerfile` 骨架**（要点：构建与运行分离、静态编译、非 root）：

```dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /out/app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

- **compose 就绪等待**（要点：依赖探针 + app 侧重试双保险）：

```yaml
services:
  db:
    image: mysql:8.4
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 5s
      retries: 20
  migrate:
    build: .
    command: ["/app", "migrate"]
    depends_on:
      db:
        condition: service_healthy
  app:
    build: .
    environment:
      DB_DSN: app:${DB_PASSWORD}@tcp(db:3306)/app?parseTime=true
    depends_on:
      migrate:
        condition: service_completed_successfully
```

- **CI 门禁顺序**（先便宜后昂贵，失败尽快暴露）：检出 → 安装 Go（`go-version-file: go.mod`）→ `gofmt -l .` → `go vet ./...` → `golangci-lint run` → `go test -race ./...` → `govulncheck ./...` → `docker build`。
- **`GOTOOLCHAIN` 的坑**：`go.mod` 的 `go` 行高于 CI 中安装的版本时，`go` 命令会尝试下载对应工具链；离线或受限网络下会直接失败。规避方式是让 CI 安装的版本不低于 `go` 行，或显式固定 `GOTOOLCHAIN` 取值与缓存工具链。
- **配置校验的位置**：`main` 中最先调用 `Validate()`，失败时用 `log/slog` 输出结构化错误并返回非零退出码；容器编排据此重试或告警。
- **自检**：`docker compose up -d` 后不手工执行任何迁移命令即可跑通链路；README 的验收命令能原样在干净环境执行成功；镜像内不存在源码与构建缓存。
