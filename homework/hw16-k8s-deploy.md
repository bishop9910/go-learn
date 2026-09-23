# 作业 16：Kubernetes 部署、Helm 与 CI/CD

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W16 |
| 发放日期 | 2027-01-18 |
| 交付日期 | 2027-01-24 |
| 预计工时 | 20 小时 |
| 难度 | 挑战 |
| 对应讲义 | L17 |
| 前置作业 | hw15 |

## 1. 背景与目标

把 hw15 的服务从「本机 `go run`」推进到「可发布、可回滚、可观测、可自动扩缩」。本作业不考 Kubernetes 概念记忆，考的是清单与 Chart 的正确性、探针与优雅退出的参数推导，以及 Go 运行时参数（容器感知 `GOMAXPROCS`、`GOMEMLIMIT`）与容器 CPU/内存 limit 的匹配。

产出方向：

- 使用 `kind`、`minikube` 或任意本地集群（安装与用法以官方文档为准，见 <https://kind.sigs.k8s.io/>、<https://minikube.sigs.k8s.io/docs/>）部署 hw15 的服务。
- 交付 `k8s/` 原始清单或 Helm Chart，二者择一并说明理由。
- 一条命令从零到 Running，一条命令完成回滚。

## 2. 需求（必须项）

1. MUST 在本地集群（`kind` 或 `minikube`）部署 hw15 的服务，并记录集群与工具版本（`kubectl version`、`helm version` 输出）。
2. MUST 交付 `k8s/` 清单，或用 Helm Chart（`Chart.yaml`、`values.yaml`、`templates/`）；选 Helm 时模板至少覆盖 `Deployment`、`Service`、`Ingress`、`ConfigMap`、`Secret`、`HPA` 六类对象。
3. MUST 提供两套覆盖值 `values-dev.yaml` 与 `values-prod.yaml`，差异至少覆盖副本数、资源 `requests`/`limits`、日志级别、`Ingress` 主机名。
4. MUST `helm lint ./chart` 与 `helm template ./chart -f chart/values-prod.yaml` 均无 error；`kubectl apply --dry-run=server -f k8s/` 通过。
5. MUST 密钥不入库：`values.yaml` 中不得出现明文口令或 token，`Secret` 由 `kubectl create secret`、外部密钥注入或 CI 变量生成；仓库内只提供 `*.example` 模板。
6. MUST 同时配置 `startup`、`readiness`、`liveness` 三类探针，并在 README 给出参数推导表：应用启动最长耗时、探测周期、单次探测开销、超时与失败阈值，推导过程须能算出最坏情况下的就绪时间。
7. MUST 复现一次「readiness 配错导致滚动更新期间 502」：记录改动内容（例如路径/端口错误或初始延迟过短）、复现命令与观测到的 502 计数，再给出修复后的对比结果。
8. MUST 为每个容器设置 CPU/内存的 `requests` 与 `limits`，并说明取值依据（压测数据或 hw10 的结论）。
9. MUST 处理 Go 1.25 起的容器感知 `GOMAXPROCS`：说明 Linux 上运行时会考虑 cgroup CPU bandwidth limit（对应 Kubernetes 的 CPU limit），且所有平台周期性更新；给出「CPU limit 与 `GOMAXPROCS` 不匹配导致 CPU 限流」的排查与验证步骤，并用 `GODEBUG=containermaxprocs=0` 做对比实验（标注 Go 1.25+）。
10. MUST 结合 `GOMEMLIMIT`（Go 1.19 起，见 <https://go.dev/doc/go1.19>）与内存 limit 给出一组经过验证的参数，说明如何避免 OOMKilled；若使用 `runtime/debug.SetMemoryLimit`，说明它与 `GOMEMLIMIT` 的关系。
11. MUST 实现优雅退出，覆盖 `terminationGracePeriodSeconds`、`preStop`、应用内 `http.Server.Shutdown`、MQ 消费者收尾与 `SIGTERM` 处理顺序，并给出时序说明。
12. MUST 提供「滚动更新期间零 5xx」验证脚本：滚动期间持续发请求，统计状态码分布，要求 5xx 计数为 0，并附一次真实运行的结果记录。
13. MUST 配置 HPA（指标类型与阈值）与 PDB，并说明 HPA 与 `requests` 的依赖关系。
14. MUST 对比 `ConfigMap`/`Secret` 的两种注入方式（环境变量 vs 挂载文件），说明热更新差异与各自适用场景。
15. MUST 处理有状态依赖：MySQL/Redis/MQ 使用 `StatefulSet`，或明确说明为何改用外部托管服务，并写出有状态工作负载的注意事项。
16. MUST 容器日志输出到 stdout 且为结构化 JSON；用 Pod 注解或 ServiceMonitor 让 Prometheus 抓取，二者择一并说明理由。
17. MUST 提供 GitHub Actions 流水线：lint → vet → test -race → 构建镜像 → 推送 → `helm upgrade --atomic`；镜像 tag 使用 git sha，不得使用 `latest`。
18. MUST 提供回滚步骤与一次演练记录：`kubectl rollout undo` 或 `helm rollback`，记录回滚耗时与验证方式。

## 3. 需求（加分项）

1. 用 `kustomize` 或 Helm library chart 消除 dev/prod 重复片段。
2. 用垂直压测或 VPA 建议值把 `requests` 调到贴近真实使用量。
3. 在容器内用 `GODEBUG=gctrace=1` 结合 RSS 观察 GC 行为，输出一组内存 limit 调参对照表。
4. 增加 NetworkPolicy 与安全上下文（非 root、只读根文件系统）。
5. 用 `helm unittest` 或 OPA/conftest 对渲染结果做断言。
6. 把零 5xx 验证接入 CI，随流水线自动执行。

## 4. 技术约束

- Go 版本基线为 Go 1.25（本机 `go1.25.7`）。标注 Go 1.26/1.27 的能力不得声称在 1.25 可用。
- 第三方依赖只写模块路径并注明「以官方最新稳定版为准」，不得编造第三方库版本号。
- 集群对象只用官方稳定 API 版本（`apps/v1`、`networking.k8s.io/v1`、`autoscaling/v2`），不得使用已废弃版本。
- 不得把密钥、`.env`、kubeconfig 提交进仓库。
- 性能与运行时数据必须给出采集命令与原始输出，不得只写结论。

## 5. 交付物清单

| 路径 | 内容 |
| --- | --- |
| `chart/` 或 `k8s/` | Helm Chart 或原始清单 |
| `chart/values-dev.yaml`、`chart/values-prod.yaml` | 两套环境覆盖值 |
| `.github/workflows/` | CI 流水线（构建、推送、`helm upgrade --atomic`、回滚） |
| `scripts/zero-5xx.sh` | 滚动更新零 5xx 验证脚本 |
| `README.md` | 部署步骤、参数推导表、故障复现记录、回滚演练记录 |
| `docs/runtime.md` | 探针推导、资源配置、运行时时序说明 |

## 6. 验收标准（可执行命令）

```bash
helm lint ./chart
helm template ./chart -f chart/values-prod.yaml > /tmp/rendered.yaml
kubectl apply --dry-run=server -f /tmp/rendered.yaml
kubectl rollout status deploy/<name> --timeout=120s
bash scripts/zero-5xx.sh --url http://localhost:8080/healthz --duration 60s
kubectl rollout undo deploy/<name>
```

PowerShell 环境：

```powershell
helm lint ./chart
helm template ./chart -f chart/values-prod.yaml | Out-File -Encoding utf8 rendered.yaml
kubectl apply --dry-run=server -f rendered.yaml
kubectl rollout status deploy/<name> --timeout=120s
```

判定标准：`helm lint` 无 error；`--dry-run=server` 返回 0；`rollout status` 在超时前成功；零 5xx 脚本输出的 5xx 计数为 0；`rollout undo` 后 revision 回到上一版本。

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 清单/Chart 质量 | 25 | 六类对象齐全、values 分层清晰、渲染可复现、模板无硬编码 |
| 探针与优雅退出 | 25 | 三类探针参数有推导、502 复现有记录、零 5xx 验证有数据 |
| 资源与运行时参数 | 20 | requests/limits 有依据、容器感知 GOMAXPROCS 描述正确、GOMEMLIMIT 组合有验证 |
| CI/CD | 15 | 流水线完整、git sha 打标、atomic 升级、回滚可执行 |
| 文档与演练 | 15 | README 可照做、故障与回滚演练记录完整 |

## 8. 提示与思路

- 先让容器在集群外跑通（镜像可起、健康检查路径可用），再写清单；顺序颠倒会把时间浪费在排查 YAML 上。
- 探针参数推导可这样组织：启动耗时 P99（由启动日志或 `startup` 首次成功时间得到）→ `startup` 的失败阈值 × 周期必须大于该值；`readiness` 的周期决定流量注入延迟；`liveness` 必须显著比 `readiness` 宽松，避免把一次慢请求误判成死锁。
- `preStop` 通常加一段等待，让 Service 端点摘除先于进程退出完成；应用侧先停止接收新请求（`Shutdown`），再等待在途请求与消费者收尾。
- 容器感知 `GOMAXPROCS` 只在设置了 CPU limit 时才有意义；limit 远小于宿主核数而未正确对齐时，会出现 P 数多于可用 CPU 时间的限流抖动。对比实验：同一负载分别用默认与 `GODEBUG=containermaxprocs=0` 运行，比较 P99 与容器节流指标。
- 内存方面：`GOMEMLIMIT` 设为容器内存 limit 的一定比例，为栈、非堆内存与运行时元数据留出余量，同时观察 RSS 与 `runtime/metrics`（见 <https://pkg.go.dev/runtime/metrics>）中的堆指标。只有 `GOMEMLIMIT` 而没有内存 limit 会导致 GC 过于频繁；只有 limit 没有 `GOMEMLIMIT` 则可能被 OOMKilled。

## 9. 常见坑

| 坑 | 现象 | 正确做法 |
| --- | --- | --- |
| 镜像 tag 用 `latest` | 回滚定位不到版本，拉取策略行为不可预期 | 用 git sha 打标，配合固定 tag |
| `imagePullPolicy` 配错 | 本地构建的镜像反复拉取失败 | 本地集群先 `kind load`/`minikube image load`，策略与 tag 匹配 |
| 认为 Secret 的 base64 是加密 | 仓库泄露即等于密钥泄露 | RBAC + 外部密钥注入，仓库只留 `*.example` |
| 探针超时过短 | 偶发重启、滚动更新卡住 | 按最坏启动时间与探测开销推导 |
| 未设 `requests` | Pod Pending、HPA 算不出利用率 | 每个容器都设 `requests` |
| `limits` 设得过低 | CPU 被节流、频繁 OOMKilled | 压测取值并与 `GOMEMLIMIT` 配合 |
| 没有 `preStop` | 端点摘除前进程已退出，连接被重置 | `preStop` + `Shutdown` + 合理宽限期 |
| HPA 与 `requests` 脱节 | 扩缩不生效或剧烈震荡 | 阈值基于 `requests` 百分比，配合稳定窗口 |
| 把集群当单机 | 在集群里跑 `docker compose`，或在 Pod 内装数据库 | 有状态依赖用 `StatefulSet` 或外部托管服务 |
| 只跑 `helm template` | 渲染通过但集群拒绝 | 补 `--dry-run=server` 做服务端校验 |

## 10. 参考实现要点

- `values.yaml` 只放结构与默认值，环境差异放 `values-dev.yaml`/`values-prod.yaml`；模板中可变量一律走 `{{ .Values.* }}`，不写字面量。
- 探针关键字段（数值按你的推导填，此处只给结构）：

```yaml
startupProbe:
  httpGet: { path: /healthz, port: http }
  periodSeconds: 2
  failureThreshold: 30
readinessProbe:
  httpGet: { path: /readyz, port: http }
  periodSeconds: 5
  timeoutSeconds: 2
livenessProbe:
  httpGet: { path: /healthz, port: http }
  periodSeconds: 10
  failureThreshold: 6
```

- 优雅退出的应用侧骨架：

```go
srv := &http.Server{Addr: ":8080", Handler: mux}
go func() { _ = srv.ListenAndServe() }()

ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
defer stop()
<-ctx.Done() // 收到 SIGTERM 后开始收尾

shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx) // 先停新连接，再等在途请求
consumer.Stop(shutdownCtx)    // 再收尾消费者，保证在途消息处理完
```

- CI 阶段划分：checkout → setup-go（缓存模块）→ `go vet ./...` → `go test -race ./...` → 构建镜像并以 `${{ github.sha }}` 打标 → 推送 → `helm upgrade --atomic --wait --timeout 5m`；失败分支执行 `kubectl rollout undo` 或 `helm rollback`。
- 运行时参数：容器内用 `GOMEMLIMIT` 设内存软上限，CPU 交给容器感知 `GOMAXPROCS` 自动对齐；需要人工干预时用 `GODEBUG=containermaxprocs=0` 关闭容器感知并显式设定，作为对照组。
- 零 5xx 脚本要点：并发若干 worker 持续请求目标地址，统计状态码直方图；滚动期间出现 5xx 或连接被拒即判失败；输出请求总数、状态码分布与 P99 延迟。
