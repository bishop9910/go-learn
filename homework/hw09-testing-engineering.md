# 作业 09：测试工程——表格驱动、httptest、mock、fuzz 与 benchmark

| 项目 | 内容 |
| --- | --- |
| 对应周次 | W9 |
| 发放日期 | 2026-11-30 |
| 交付日期 | 2026-12-06 |
| 预计工时 | 20 小时 |
| 难度 | 进阶 |
| 对应讲义 | L06 |
| 前置作业 | hw08 |

## 1. 背景与目标

现有仓库的测试是零散用例：一个 `TestXxx` 里塞多个断言、没有用例名、失败时无法定位输入；HTTP 层靠手工起服务测；解析器没有模糊测试；性能改动没有 benchmark 支撑。本作业把测试从「零散用例」提升为可维护的测试体系。

| 目标 | 可观测结果 |
| --- | --- |
| 结构统一 | ≥3 个包采用表格驱动 + 子测试，失败输出自带用例名 |
| 边界可覆盖 | HTTP handler 的鉴权、体积、编码、方法四类边界各有用例 |
| 输入可探索 | 两个 `Fuzz` 目标，崩溃输入固化为 `testdata/fuzz/` 回归用例 |
| 性能可回归 | ≥3 个 benchmark，优化前后有 `benchstat` 对比与 `p-value` |
| 覆盖率可解释 | 按包覆盖率表 + 未覆盖分支的逐条理由 |

## 2. 需求（必须项）

**M1 表格驱动与子测试。** 至少 3 个包改造为表格驱动 + 子测试（`t.Run`），每个用例有 `name` 字段与明确期望值；用 `t.Helper()` 抽公共断言函数，使失败行号指向调用处；用 `t.Cleanup()` 管理测试资源（临时文件、启动的服务、打开的连接）。用例之间不得共享可变状态。

**M2 测试替身设计。** 为仓储层定义接口，手写 fake 作为首选方案；若引入 mock 框架（`go.uber.org/mock` 或 `github.com/stretchr/testify`，仅给模块路径，版本以官方最新稳定版为准），必须给出取舍表，列固定为 `维度 / 手写 fake / mock 框架`，维度至少包含可读性、维护成本、断言能力、对重构的敏感度、调试难度。结论要写成可判定的选型规则（什么情况下用哪种）。

**M3 httptest 覆盖。** 用 `httptest.NewRecorder` 测 handler 与完整中间件链（鉴权 → 日志 → 恢复），用 `httptest.NewServer` 测客户端逻辑（重试、超时、错误解码）。边界至少覆盖：鉴权失败、鉴权成功、请求体过大、非法 JSON、方法不允许。每条边界断言状态码与响应体结构，不只断言 `err != nil`。

**M4 并发测试。** 用 `testing/synctest`（Go 1.25 转正）写一个完整示例：`synctest.Test` 包裹测试体，`synctest.Wait` 等待所有 goroutine 阻塞，从而在虚拟时间里测试带超时的逻辑而无需真实等待。**Go 1.27 起可用 `testing/synctest.Sleep`**（等价于 `time.Sleep` + `synctest.Wait`），在 1.27 环境下用它替换手写等待并标注版本。

**M5 模糊测试。** 为日志行解析器（hw01 / hw02 的 `logline`）与 JSON 配置解析各写一个 `Fuzz` 目标，每个至少 5 个种子用例；本地跑一段 `-fuzztime`；把发现的崩溃输入固化为回归用例放进 `testdata/fuzz/` 并提交；说明 `-fuzz` 与 CI 中只跑种子用例（不加 `-fuzz`）的区别，以及为什么 CI 不能跑无界 fuzz。

**M6 基准测试与 benchstat。** 至少 3 个 benchmark：字符串拼接对比 `strings` 包的 `Builder`、JSON 编解码、锁粒度对比。全部使用 `b.ReportAllocs()`，运行时带 `-benchmem`。其中至少一个用 `b.Loop()` 编写，并说明**Go 1.26 起 `b.Loop()` 不再阻止循环体内联**（Go 1.25 上退回 `for i := 0; i < b.N; i++`）。用 `benchstat`（模块路径 `golang.org/x/perf/cmd/benchstat`，以官方最新稳定版为准）产出优化前后对比表，包含 `p-value`，并解释 `p-value` 的判定门槛。

**M7 覆盖率与说明。** 用 `go test -coverprofile` 生成报告、`go tool cover -func` 输出按函数覆盖率，整理成按包覆盖率表。对未覆盖分支逐条说明原因，区分「错误路径难以构造」「防御性分支」「外部依赖无法在单测中触发」三类。不得为刷覆盖率写空测试。

**M8 测试组织与分层。** 单元测试放 `internal/...`，集成测试放 `test/`；重型测试用 `testing.Short()` 判断并用 `-short` 跳过；CI 中按标记分层执行（快速层每次提交、重型层每日或合入主干）。README 增加「如何跑测试」章节，列出每层命令与预期耗时。

## 3. 需求（加分项）

- **A1 Go 1.27+ 内存假网络测试。** 用 `net/http/httptest.NewTestServer`（Go 1.27 新增）配合 `testing/synctest` 写一个测试；本地未升级到 1.27 时，该条在交付说明里标注为「升级后待办」。
- **A2 测试属性标签。** 用 `T.Attr` / `B.Attr` / `F.Attr`（Go 1.25 新增）给测试打属性，便于按属性筛选与生成报告。
- **A3 测试输出捕获。** 用 `T.Output`（Go 1.25 新增，类型为 `io.Writer`）替代直接写 `os.Stdout`，让日志断言在并行测试下稳定。
- **A4 测试产物目录。** 用 `testing.T.ArtifactDir`（Go 1.26 新增，配合 `go test -artifacts` 与 `-outputdir`）保存 profile、崩溃语料与失败请求转储。
- **A5 分配回归。** 用 `testing.AllocsPerRun` 写分配量回归测试；注意 **Go 1.25 起在有并行测试运行时它会 panic**，需保证该测试串行执行。

## 4. 技术约束

| 约束 | 说明 |
| --- | --- |
| 语言基线 | 本仓库基线 Go 1.25；使用 1.26 / 1.27 能力时必须标注版本并提供 1.25 的替代写法 |
| 依赖 | 测试依赖尽量为零；引入 mock 框架须在 README 说明理由 |
| 断言风格 | 不使用可能掩盖错误的断言封装；失败信息必须包含输入、期望、实际 |
| 网络 | 单元测试与集成测试一律不得访问真实外网 |
| 时间 | 测试不得依赖真实时钟等待，超时类逻辑用 `testing/synctest` |
| 提交 | fuzz 语料、覆盖率报告与 benchstat 原始输出必须提交 |

## 5. 交付物清单

| 编号 | 交付物 | 形式 |
| --- | --- | --- |
| D1 | 测试代码 | `internal/**/*_test.go`、`test/*_test.go` |
| D2 | 覆盖率报告 | `docs/hw09-coverage.md` + `cover.out` |
| D3 | benchstat 对比表 | `docs/hw09-benchstat.md` + `bench-old.txt` / `bench-new.txt` |
| D4 | fuzz 语料 | `testdata/fuzz/FuzzParseLine/`、`testdata/fuzz/FuzzParseConfig/` |
| D5 | 测试替身取舍表 | `docs/hw09-test-doubles.md` |
| D6 | README「如何跑测试」 | `README.md` 新增章节 |
| D7 | CI 分层配置 | `.github/workflows/test.yml` 或等价脚本 |

## 6. 验收标准（可执行命令）

```bash
go vet ./...
go test -race -count=1 ./...
go test -short ./...
go test -bench . -benchmem -count=10 ./... | tee bench-new.txt
go test -fuzz=FuzzParseLine -fuzztime=30s ./internal/logline
go test -coverprofile=cover.out ./... && go tool cover -func=cover.out
```

| 命令 | 通过条件 |
| --- | --- |
| `go test -race -count=1 ./...` | 全部通过，无 `DATA RACE` |
| `go test -short ./...` | 跳过重型测试后仍全部通过，耗时明显低于全量 |
| `-bench . -benchmem -count=10` | 每个 benchmark 有 `ns/op` 与 `B/op`、`allocs/op`；可用于 benchstat |
| fuzz 命令 | 30s 内无崩溃；若有崩溃，语料已固化且 `go test ./internal/logline` 能复现 |
| 覆盖率命令 | 输出按函数覆盖率表，与 D2 文档一致 |
| 语料提交 | `git status` 中 `testdata/fuzz/` 无未跟踪文件 |

## 7. 评分表

| 维度 | 分值 | 评分要点 |
| --- | --- | --- |
| 表格驱动与结构 | 20 | ≥3 个包改造；用例有名字；`t.Helper()` / `t.Cleanup()` 使用得当；无共享可变状态 |
| 测试替身设计 | 15 | 接口位于消费方；手写 fake 清晰；取舍表结论可判定；未滥用 mock 框架 |
| httptest 覆盖 | 15 | 中间件链被测；5 类边界齐全；断言状态码与结构而非仅错误 |
| fuzz | 15 | 两个目标各 ≥5 种子；崩溃语料固化并提交；`-fuzz` 与 CI 差异说明准确 |
| benchmark 与 benchstat | 20 | ≥3 个 benchmark；`b.Loop()` 使用正确；对比表含 `p-value` 并正确解读 |
| 覆盖率与说明 | 15 | 覆盖率表可复现；未覆盖分支逐条有理由；无刷分测试 |

## 8. 提示与思路

- 表格驱动的核心价值是「失败信息自带输入」。用例名建议包含输入摘要与期望，例如 `empty_input_returns_error`。
- `t.Cleanup()` 比 `defer` 更适合测试：它在子测试结束时也会执行，且与 `t.Helper()` 配合时错误行号更准确。
- 手写 fake 优先的原因：只需实现被测代码真正调用的方法、能精确控制返回序列、出错时调用栈就在测试里。mock 框架适合「交互次数本身是需求」的场景。
- 模糊测试先跑种子再放大：`go test ./pkg`（只跑种子）应当是 CI 的默认动作，`-fuzztime` 只在本地或定时任务里跑。
- benchmark 里防止被测代码被优化掉：把结果赋给包级变量或使用框架提供的循环机制，别写 `_ = f(x)` 就完事。
- `-count=10` 是 benchstat 能给出统计判定的前提；`-count=1` 的对比表没有 `p-value` 意义。
- 覆盖率低的包先看是否只是「getter 与错误分支」，再决定补测试还是接受现状，理由要写进 D2。

## 9. 常见坑

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 测试共享可变状态 | 单独跑通过、全量跑失败 | 包级变量或共享 fixture 被多个用例改写 | 每个用例独立构造输入；必要时用 `t.Cleanup()` 还原 |
| `t.Parallel` 与闭包变量 | 并行后断言串味、随机失败 | 闭包捕获了会被下一轮改写的变量 | Go 1.22 起循环变量每轮独立，仍需确认捕获的是当轮值；无法确认时不加 `t.Parallel` |
| 断言只判 `err != nil` | 错误类型变了测试照样过 | 断言粒度太粗 | 断言具体错误类型、状态码、响应体字段 |
| benchmark 中被测代码被优化掉 | `ns/op` 异常低且与实现无关 | 计算结果未被使用，编译期消除 | 结果赋给包级变量；用框架循环机制而非手写 `b.N` 循环 |
| fuzz 语料未提交 | 换机器后回归用例消失 | 只放在本地缓存，未纳入版本控制 | 崩溃输入写入 `testdata/fuzz/` 并提交 |
| 覆盖率刷分 | 覆盖率很高但缺陷仍漏出 | 写了不判定的调用型测试 | 以分支与边界为覆盖目标，未覆盖分支写明理由 |
| 用真实 sleep 测超时 | 测试又慢又不稳定 | 依赖真实时钟 | 用 `testing/synctest` 的虚拟时间 |

## 10. 参考实现要点

只给关键设计决策与片段，不给完整成品代码。

```go
// 决策 1：表格驱动 + 子测试（Go 1.22 起循环变量每轮独立，无需 tt := tt）
tests := []struct {
	name    string
	in      string
	want    int
	wantErr error
}{
	{"empty_input_returns_error", "", 0, ErrEmpty},
	{"valid_line_parses_level", "2026-11-23T10:00:00Z INFO start", 4, nil},
}

for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		got, err := Parse(tt.in)
		if !errors.Is(err, tt.wantErr) {
			t.Fatalf("Parse(%q) err = %v, want %v", tt.in, err, tt.wantErr)
		}
		if got != tt.want {
			t.Errorf("Parse(%q) = %d, want %d", tt.in, got, tt.want)
		}
	})
}
```

```go
// 决策 2：虚拟时间测试超时逻辑（testing/synctest，Go 1.25 转正）
synctest.Test(t, func(t *testing.T) {
	done := make(chan struct{})
	go func() {
		defer close(done)
		waitWithTimeout(context.Background(), time.Second)
	}()
	synctest.Wait() // 等待所有 goroutine 阻塞后再推进虚拟时间
})
// Go 1.27+ 可直接用 testing/synctest.Sleep 替代 time.Sleep + synctest.Wait
```

```go
// 决策 3：b.Loop（Go 1.26 起不再阻止循环体内联；1.25 用 for i := 0; i < b.N; i++）
func BenchmarkBuildLine(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		_ = buildLine(sampleRecord)
	}
}
```

```go
// 决策 4：fuzz 目标，至少 5 个种子；崩溃输入自动落盘到 testdata/fuzz/
func FuzzParseLine(f *testing.F) {
	f.Add("2026-11-23T10:00:00Z INFO start")
	f.Add("")
	f.Add("no-timestamp")
	f.Add("2026-11-23T10:00:00Z ERROR boom err=timeout")
	f.Add("\x00\xff")
	f.Fuzz(func(t *testing.T, s string) {
		_, _ = ParseLine(s) // 断言：不 panic；返回值的其他性质按解析器契约补充
	})
}
```

```go
// 决策 5：仓储接口定义在消费方，fake 只需实现被测路径
type PostStore interface {
	Get(ctx context.Context, id int64) (*Post, error)
}

type fakePostStore struct {
	getFn func(ctx context.Context, id int64) (*Post, error)
}
```

- 覆盖率表建议按 `包 / 语句覆盖率 / 主要未覆盖分支 / 理由` 四列组织，便于评审直接核对。
- benchstat 对比表建议给出 `old / new / delta / p-value`，并对 `p-value` 大于 0.05 的项明确写「差异不显著」。
- CI 分层建议用两个 job：`-short` 快速层，全量 + race 层；fuzz 用定时任务而非每次提交。
