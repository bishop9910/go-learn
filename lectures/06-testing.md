# L06 测试工程：testing / httptest / fuzz / benchmark / synctest

本篇定位：把 `go test` 从「能跑」推进到「能当工程门禁」——命令行全流程、表格驱动与生命周期、HTTP 测试、基准与性能回归、模糊测试、虚拟时钟，以及覆盖率与测试替身的取舍。对应 W2–W3（工具链与工程化）与 W9（质量与性能）。前置要求：能写 Go 包与函数，会用[L05 序列化：encoding/json 与 json/v2](05-serialization-json.md)中的 JSON 编解码。

## 1. go test 全流程

### 1.1 常用标志

| 标志 | 作用 | 实践建议 |
| --- | --- | --- |
| `-run 'TestX/sub'` | 只跑匹配的测试，`/` 分隔父子测试名 | 开发期缩小范围，注意它是正则 |
| `-v` | 输出每个测试与子测试的名字和日志 | 定位失败必开；CI 上默认噪音大 |
| `-count=1` | 禁用结果缓存 | 怀疑「改了代码还通过」时使用，也可 `-count=N` 跑多次 |
| `-short` | 设置短模式标记 | 耗时用例放在 `if testing.Short() { t.Skip(...) }` 之后 |
| `-timeout 60s` | 超时 panic 并打印所有 goroutine 栈 | 默认 10 分钟太长；不要靠加大超时掩盖死锁 |
| `-cover` / `-coverprofile=cover.out` | 语句覆盖率 / 写出覆盖率数据 | 用 `go tool cover -html=cover.out` 逐行查看 |
| `-race` | 打开数据竞争检测器 | CI 必开；本地可只对改动包开 |
| `-shuffle=on` | 随机化执行顺序 | 抓测试间共享状态；失败时会打印复现 seed |
| `-failfast` | 首个失败后停止 | 大包快速反馈；排查时不要开 |
| `-bench` / `-benchmem` | 运行基准 / 报告内存分配 | 只与 `-run` 一起用，避免顺带跑测试 |
| `-fuzz` / `-fuzztime` | 运行模糊测试 / 限制时长 | 见第 6 节 |
| `-artifacts` / `-outputdir` | 启用并有选择地输出测试产物 | 配合 `T.ArtifactDir`，见第 3 节 |
| `-json` | 机器可读事件流 | 给 CI 解析；Go 1.27 起 output 行新增 `OutputType` |

运行范围（Go 命令跨平台，bash 与 PowerShell 写法一致）：

```bash
go test ./...                              # 当前模块所有包
go test ./internal/...                     # 某个子树
go test -run 'TestUser' ./internal/user    # 单包内的匹配用例
go test -count=1 -race -shuffle=on ./...   # 提交前的一次完整检查
```

### 1.2 文件命名与包名选择

| 项 | 规则 |
| --- | --- |
| 文件名 | 必须以 `_test.go` 结尾，否则不参与测试构建 |
| `package foo`（内部测试） | 与被测代码同包，可访问非导出标识符，适合测内部实现 |
| `package foo_test`（外部测试） | 只能访问导出 API，强制从使用者视角写断言，避免测试循环导入 |
| `testdata/` | 工具链忽略该目录，专放测试数据；模糊语料放 `testdata/fuzz/` |
| 辅助代码 | 放在 `_test.go` 内，不要进正式包，避免污染生产依赖图 |

建议：对外行为用 `package foo_test`，需要构造内部状态时才用 `package foo`，一个文件只选一种。

## 2. 表格驱动与测试生命周期

### 2.1 表格驱动 + 子测试

```go
func TestParseAmount(t *testing.T) {
	tests := []struct {
		name, in string
		want     int64
		wantErr  bool
	}{
		{name: "integer", in: "100", want: 100},
		{name: "with unit", in: "100ms", wantErr: true},
		{name: "empty", in: "", wantErr: true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			got, err := ParseAmount(tt.in)
			if tt.wantErr {
				if err == nil {
					t.Fatalf("ParseAmount(%q) 期望错误，实际 nil", tt.in)
				}
				return
			}
			if err != nil {
				t.Fatalf("ParseAmount(%q) 意外错误: %v", tt.in, err)
			}
			if got != tt.want {
				t.Errorf("ParseAmount(%q) = %d, want %d", tt.in, got, tt.want)
			}
		})
	}
}
```

Go 1.22 起循环变量每轮独立，**不需要**再写 `tt := tt`；看到这行可以直接删掉。

### 2.2 生命周期与辅助函数

| 工具 | 用途 | 关键点 |
| --- | --- | --- |
| `t.Helper()` | 把辅助函数从失败位置中摘出去 | 必须在辅助函数第一行调用，否则行号指向辅助函数 |
| `t.Cleanup(f)` | 注册清理函数，子测试结束时逆序执行 | 优先于 `defer`：`t.Fatal` 提前返回也会执行，并行语义正确 |
| `t.Parallel()` | 标记该测试可与同层其他并行测试同时运行 | 捕获循环变量之后再调用；之后不要再改共享状态 |
| `t.TempDir()` | 每个测试独立的临时目录 | 自动清理；并行时路径不冲突，禁止手工拼临时路径 |
| `t.Skip` / `t.Skipf` | 跳过并标记原因 | 用于环境不满足（无 Docker、无网络），不要掩盖失败 |

```go
func newTestStore(t *testing.T) *Store {
	t.Helper()
	s, err := OpenStore(filepath.Join(t.TempDir(), "data.db"))
	if err != nil {
		t.Fatalf("OpenStore: %v", err)
	}
	t.Cleanup(func() {
		if err := s.Close(); err != nil {
			t.Errorf("Close: %v", err)
		}
	})
	return s
}
```

### 2.3 t.Parallel 的陷阱

| 陷阱 | 后果 | 正确做法 |
| --- | --- | --- |
| `t.Parallel()` 之后修改包级变量 | 数据竞争，`-race` 报错 | 并行用例只读共享状态 |
| 并行测试里改环境变量或工作目录 | 全局状态互相踩踏 | 不在并行测试里改；或该类测试不并行 |
| 并行测试里调用 `testing.AllocsPerRun` | Go 1.25 起直接 panic | 分配计数测试不并行 |

## 3. 版本相关的新能力

| 能力 | 版本 | 说明 |
| --- | --- | --- |
| `T.Attr` / `B.Attr` / `F.Attr` | Go 1.25 | 给测试/基准/模糊目标附加键值属性，出现在 `-v` 输出中 |
| `T.Output` | Go 1.25 | 返回 `io.Writer`，内容自动归属当前测试，并行下不互相插入 |
| `testing/synctest` 转正 | Go 1.25 | `synctest.Test`、`synctest.Wait`；1.24 旧实验 API 在 1.26 移除 |
| `T.ArtifactDir` | Go 1.26 | 该测试专属的产物目录，配合 `go test -artifacts` 与 `-outputdir` |
| `B.Loop` 不再阻止循环体内联 | Go 1.26 | 见第 5 节 |
| `synctest.Sleep` | Go 1.27 | 等价于 `time.Sleep` 加 `synctest.Wait` |
| `httptest.NewTestServer` | Go 1.27 | 内存假网络中的测试服务器，配合 `testing/synctest` |

```go
func TestRenderReport(t *testing.T) {
	t.Attr("component", "report") // Go 1.25+
	out := t.Output()             // Go 1.25+
	fmt.Fprintf(out, "开始生成报告\n")
	if err := RenderReport(out); err != nil {
		t.Fatalf("RenderReport: %v", err)
	}
}
```

```go
func TestGenerateDiff(t *testing.T) {
	dir := t.ArtifactDir() // Go 1.26+
	_ = os.WriteFile(filepath.Join(dir, "diff.txt"), []byte("..."), 0o644)
}
```

`go test -run TestGenerateDiff -artifacts -outputdir=./test-artifacts ./...`：`-artifacts` 决定是否落盘，`-outputdir` 决定落盘位置。CI 里失败时把该目录作为构建产物上传，就不用再让人本地复现。

## 4. net/http/httptest

| API | 用途 | 说明 |
| --- | --- | --- |
| `httptest.NewRequest` | 构造 `*http.Request` | 不经网络，可直接塞进 handler |
| `httptest.NewRecorder` | 构造记录状态码/头/响应体的 `ResponseWriter` | 测 handler 的标准手段 |
| `httptest.NewServer` | 启动监听 loopback 的真实服务器 | 用于测客户端：超时、重试、连接复用、cookie |
| `Server.Client()` | 返回已配好地址与证书的 `*http.Client` | 比自己 `http.Get(srv.URL)` 更稳 |
| `httptest.NewTestServer` | Go 1.27 新增，内存假网络服务器 | 配合 `testing/synctest`，不占端口、不受真实时钟影响 |

```go
func TestGetUser(t *testing.T) {
	mux := NewRouter()
	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, "/users/42", nil))
	if rec.Code != http.StatusOK {
		t.Fatalf("状态码 = %d, want %d", rec.Code, http.StatusOK)
	}
	if ct := rec.Header().Get("Content-Type"); !strings.HasPrefix(ct, "application/json") {
		t.Errorf("Content-Type = %q", ct)
	}
}
```

```go
func TestFetchUser(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte(`{"id":"42","name":"alice"}`))
	}))
	defer srv.Close()
	client := srv.Client() // 已配好地址，可直接请求 srv.URL
	client.Timeout = 2 * time.Second
	u, err := FetchUser(context.Background(), client, srv.URL, "42")
	if err != nil || u.Name != "alice" {
		t.Fatalf("FetchUser: err=%v name=%q", err, u.Name)
	}
}
```

Go 1.27 起可配合 `synctest` 使用 `httptest.NewTestServer`：客户端超时、握手超时都走虚拟时钟，测试既快又稳定。具体签名见 <https://pkg.go.dev/net/http/httptest>。

## 5. 基准测试

### 5.1 基本写法

```go
func BenchmarkParseAmount(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		_, _ = ParseAmount("12345")
	}
}
```

```bash
go test -run '^$' -bench . -benchmem -cpu=1,4 ./internal/amount
```

| 元素 | 作用 |
| --- | --- |
| `b.N` | 框架决定的迭代次数；`b.Loop()` 由框架驱动循环 |
| `b.ResetTimer()` | 丢掉准备阶段计时，准备数据不计入结果 |
| `b.ReportAllocs()` | 报告每轮分配字节数与次数；等价于 `-benchmem` |
| `b.RunParallel` | 多 goroutine 并发调用，测吞吐与锁竞争 |
| `-cpu=1,2,4` | 用不同 `GOMAXPROCS` 重复运行，看是否随核数线性扩展 |

手写 `for i := 0; i < b.N; i++` 时，返回值被丢弃会让整个调用被编译器优化掉，分数虚高。要么把结果写进基准函数可见的变量（包级 sink），要么改用 `b.Loop()`：它保证循环体不被优化掉，无需 sink 变量；Go 1.26 起它不再阻止循环体内联，因此被测的小函数能正常内联，数字更接近生产路径。新写的基准一律用 `b.Loop()`。

```go
var sink int64 // 包级变量：让结果被观测，避免调用被优化掉

func BenchmarkHashParallel(b *testing.B) {
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			v, _ := ParseAmount("12345")
			sink = v
		}
	})
}
```

### 5.2 benchstat 对比

`benchstat` 属于 `golang.org/x/perf/cmd/benchstat`（以官方最新稳定版为准）：

```bash
go test -run '^$' -bench BenchmarkDecode -count=10 ./internal/codec > old.txt
go test -run '^$' -bench BenchmarkDecode -count=10 ./internal/codec > new.txt   # 改代码之后
benchstat old.txt new.txt
```

要求 `-count` ≥ 6，同一台机器、同一电源策略下采集；报告里的 `p` 值与置信区间用于判断差异是否显著，单次跑出的 5% 波动不足以作为优化结论。

## 6. 模糊测试

```go
func FuzzParseAmount(f *testing.F) {
	for _, seed := range []string{"100", "-5", "", "9999999999999999999999"} {
		f.Add(seed)
	}

	f.Fuzz(func(t *testing.T, in string) {
		got, err := ParseAmount(in)
		if err != nil {
			return
		}
		// 不变式：解析成功的结果，格式化回去必须能再次解析
		if _, err := ParseAmount(strconv.FormatInt(got, 10)); err != nil {
			t.Fatalf("round-trip 失败: %d", got)
		}
	})
}
```

| 概念 | 说明 |
| --- | --- |
| `f.Add(...)` | 种子语料，参数类型必须与 `f.Fuzz` 回调一致 |
| `f.Fuzz(fn)` | 模糊目标；变异输入反复调用 `fn`，任何 panic 或 `t.Fatal` 都算失败 |
| `testdata/fuzz/` | 失败输入自动写入该目录，必须提交进版本库 |
| 普通 `go test` | 只重放种子与已入库语料，不做变异，日常构建不受影响 |
| `-fuzz=FuzzParseAmount` / `-fuzztime=30s` | 开启变异并限制时长，CI 上按包分配预算 |

一个包同时只能有一个模糊目标在跑；`f.Fuzz` 回调失败时要把输入本身放进错误信息，否则复现还得翻语料文件。

## 7. testing/synctest

`testing/synctest` 在 Go 1.25 转正，提供「虚拟时钟气泡」：气泡内 goroutine 一旦全部阻塞，虚拟时钟直接跳到下一个唤醒点。于是测试带 `time.Sleep`、超时、重试退避的代码，真实耗时接近 0。

| API | 版本 | 作用 |
| --- | --- | --- |
| `synctest.Test` | Go 1.25 转正 | 在气泡中运行一个测试函数 |
| `synctest.Wait` | Go 1.25 转正 | 阻塞到气泡内其他 goroutine 都进入阻塞状态 |
| `synctest.Sleep` | Go 1.27 | 等价于 `time.Sleep` 加 `synctest.Wait` |

被测代码（真实运行要等 300ms）：

```go
func retryDo(ctx context.Context, attempts int, do func() error) error {
	var err error
	for i := 0; i < attempts; i++ {
		if err = do(); err == nil {
			return nil
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(100 * time.Millisecond):
		}
	}
	return err
}
```

虚拟时钟测试，不真的等待，但断言虚拟耗时：

```go
func TestRetryDo(t *testing.T) {
	// 子测试一：两次退避后成功，断言虚拟耗时精确等于 200ms。
	synctest.Test(t, func(t *testing.T) {
		start := time.Now()
		calls := 0
		err := retryDo(context.Background(), 3, func() error {
			calls++
			if calls < 3 {
				return errors.New("transient")
			}
			return nil
		})
		if err != nil || calls != 3 {
			t.Fatalf("err=%v calls=%d, want nil/3", err, calls)
		}
		if elapsed := time.Since(start); elapsed != 200*time.Millisecond {
			t.Errorf("虚拟耗时 = %v, want 200ms", elapsed)
		}
	})

	// 子测试二：上下文在第二次退避前超时，虚拟时钟推进但不真的等待。
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
		defer cancel()
		err := retryDo(ctx, 10, func() error { return errors.New("always") })
		if !errors.Is(err, context.DeadlineExceeded) {
			t.Fatalf("错误 = %v, want DeadlineExceeded", err)
		}
	})
}
```

要点：气泡内 `time.Sleep`、`time.After`、`context` 超时都走虚拟时钟，断言可精确到毫秒且不因 CI 机器慢而抖动；若气泡内 goroutine 阻塞在真实网络上，虚拟时钟无法推进，会直接超时，所以 synctest 只用于纯时钟与内存场景；Go 1.27 起把「睡一会儿并等所有 goroutine 就绪」写成一行 `synctest.Sleep(d)`，此前需要 `time.Sleep(d)` 后跟 `synctest.Wait()`；与 `httptest.NewTestServer`（Go 1.27）组合，可把服务端、客户端与超时全部放进同一气泡。

## 8. 测试替身

优先级：接口 + 手写 fake > 内存实现 > 生成式 mock。生产代码先定义窄接口（3–5 个方法），测试里手写实现，能表达「调用几次」「返回什么错误」这类断言，零依赖且可读。

| 工具 | 模块路径 | 适用 |
| --- | --- | --- |
| gomock | `go.uber.org/mock` | 需要严格断言调用次数与参数时序 |
| mockery | `github.com/vektra/mockery` | 从接口批量生成 mock，减少样板 |
| testify | `github.com/stretchr/testify` | 断言与 suite 组织 |

以上均以官方最新稳定版为准，不锁定版本号。引入前先问：这个接口用 30 行手写 fake 是不是更清楚？是的话就别加依赖。

```go
type UserStore interface {
	Get(ctx context.Context, id string) (*User, error)
}

type fakeUserStore struct { // 只实现测试真正需要的行为
	users map[string]*User
	err   error
}

func (f *fakeUserStore) Get(ctx context.Context, id string) (*User, error) {
	if f.err != nil {
		return nil, f.err
	}
	if u, ok := f.users[id]; ok {
		return u, nil
	}
	return nil, ErrNotFound
}
```

## 9. 分配断言与覆盖率

```go
func TestBuildKeyNoAlloc(t *testing.T) {
	if got := testing.AllocsPerRun(100, func() { _ = buildKey("users", "42") }); got != 0 {
		t.Fatalf("buildKey 分配次数 = %v, want 0", got)
	}
}
```

Go 1.25 起 `testing.AllocsPerRun` 在有并行测试运行时 panic：该测试不要调用 `t.Parallel()`，也不要被并行子测试包裹；需要隔离时单独放一个测试文件，用 `go test -run TestBuildKeyNoAlloc` 单跑。零分配断言优先挑热路径上的关键函数。

覆盖率：`go test -cover ./...` 看趋势，`go test -coverprofile=cover.out ./...` 加 `go tool cover -html=cover.out` 看逐行。陷阱：覆盖率不等于正确性（只调用不断言照样 100%）；语句覆盖率不反映 `a && b` 的真值组合；为凑数字去测 getter 会掩盖边界与错误路径；门禁更适合写成「新增代码覆盖率不得低于 X」而不是「全仓必须 Y%」。

测试金字塔在 Go 后端项目的落地：

| 层 | 占比 | 写法 | 运行成本 |
| --- | --- | --- | --- |
| 单元测试 | 约 70% | 纯函数与领域逻辑，表格驱动，无 IO | 毫秒级，保存即跑 |
| 组件测试 | 约 20% | 真实 handler + `httptest` + 内存存储 + `synctest` 虚拟时钟 | 百毫秒级，提交前跑 |
| 集成测试 | 约 10% | 真数据库/真容器，用 `-short` 之外的开关与 `t.Skip` 守卫 | 秒到分钟级，CI 单独阶段 |

反金字塔（大量依赖 mock 的单测 + 极少真实集成）会让「测试全绿但服务起不来」成为常态：mock 复现的是你对依赖的想象，不是依赖本身。

## 10. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 只打印不失败的断言 | 用例永远通过 | 忘了 `t.Errorf`/`t.Fatalf` | 用 `t.Errorf` 系列或断言库 |
| 在非测试 goroutine 里 `t.Fatalf` | 报「Fatal 从非测试 goroutine 调用」 | `Fatal` 只能在测试自身 goroutine 调 | 用 channel 把错误传回主 goroutine |
| 辅助函数没有 `t.Helper()` | 失败行号指向辅助函数 | 框架识别不出辅助帧 | 辅助函数第一行写 `t.Helper()` |
| 用 `defer` 而非 `t.Cleanup` 清理 | 并行子测试下顺序错乱 | `defer` 绑定函数返回而非子测试结束 | 用 `t.Cleanup` |
| 手工拼临时目录 | 并行用例互相覆盖文件 | 共享路径 | 用 `t.TempDir()` |
| `t.Parallel()` 后改包级变量 | `-race` 报数据竞争 | 共享可变状态 | 每个用例独立状态 |
| 并行用例里用 `testing.AllocsPerRun` | Go 1.25 起 panic | 该函数禁止与并行测试共存 | 不并行，或单独包运行 |
| 基准里丢弃返回值 | 分数好得不真实 | 调用被编译器优化掉 | 用 `b.Loop()` 或写入 sink |
| 模糊失败语料不入库 | 修了还会再犯 | `testdata/fuzz/` 未提交 | 把语料文件提交进版本库 |
| 所有依赖都 mock | 单测全绿、联调全崩 | mock 与真实依赖行为不一致 | 关键路径用真实组件 |

## 11. 动手练习

1. 为一个「金额解析 + 格式化」小包写完整测试：表格驱动、子测试、`-race`、`-shuffle=on`、`-count=1` 全绿，且错误分支都被覆盖。
2. 用 `httptest.NewRecorder` 测一个带路径参数与 JSON 校验的 handler，再用 `httptest.NewServer` 从客户端侧测同一功能，比较两种写法的断言差异。
3. 给一个字符串处理函数写基准，分别用 `for i := 0; i < b.N; i++` 与 `b.Loop()` 实现，用 `-benchmem` 对比并解释结果差异。
4. 写一个 `FuzzX` 目标，跑 `-fuzztime=30s` 找出至少一个 panic，把失败语料留在 `testdata/fuzz/`，修复后确认 `go test` 能回归。
5. 给第 7 节的 `retryDo` 补一个用例：上下文在第一次退避期间被取消，断言返回取消错误而非业务错误，且整个测试真实耗时小于 100ms。
6. 给一个 handler 加产物输出，用 `T.ArtifactDir` 落盘响应快照，并用 `-artifacts -outputdir=./test-artifacts` 确认文件生成。

## 12. 自检清单

- [ ] 能默写提交前的检查命令：`go test -count=1 -race -shuffle=on ./...`。
- [ ] 知道 `-short`、`-failfast`、`-timeout`、`-coverprofile` 各自解决什么问题。
- [ ] 能解释 `package foo` 与 `package foo_test` 的取舍。
- [ ] 表格驱动测试都用 `t.Run` 子测试，命名能定位到具体输入。
- [ ] 已按 Go 1.22 之后的语义清理循环变量捕获（没有多余的 `tt := tt`）。
- [ ] 资源清理走 `t.Cleanup`，临时文件走 `t.TempDir()`。
- [ ] 并行测试不共享可变状态，环境变量类测试没有并行化。
- [ ] 至少一个基准用 `b.Loop()` 写，并用 `-count` + `benchstat` 做过对比。
- [ ] 至少一个 `FuzzX` 目标，失败语料已提交进 `testdata/fuzz/`。
- [ ] 时间相关逻辑用 `testing/synctest` 测，测试里没有真实等待。
- [ ] 覆盖率只当趋势指标，关键路径有真实断言与边界用例。
- [ ] 能指出项目里单元、组件、集成三层各有哪些测试。

## 13. 延伸阅读

- Go 1.25 Release Notes：<https://go.dev/doc/go1.25>
- Go 1.26 Release Notes：<https://go.dev/doc/go1.26>
- Go 1.27 Release Notes：<https://go.dev/doc/go1.27>
- `testing`：<https://pkg.go.dev/testing>
- `testing/synctest`：<https://pkg.go.dev/testing/synctest>
- `net/http/httptest`：<https://pkg.go.dev/net/http/httptest>
- `go test` 命令与标志：<https://pkg.go.dev/cmd/go#hdr-Test_packages>
- `golang.org/x/perf/cmd/benchstat`：<https://pkg.go.dev/golang.org/x/perf/cmd/benchstat>
- 书籍：《Go语言实战》，William Kennedy，人民邮电出版社
