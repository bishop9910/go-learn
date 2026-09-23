# L04 标准库核心：io / os / strings / strconv / bytes / bufio / time

本篇定位：把后端日常最常用的七个标准库包讲到能凭记忆写出正确代码，重点是流式 I/O 的组合方式、`os.Root` 的目录穿越防护、字符串拼接的性能差异、`bufio` 的缓冲时机与时间的单调时钟语义。
对应学习周：W2。前置要求：完成 [L02 语言精要与易错点](02-language-essentials.md) 与 [L03 错误处理与结构化日志](03-errors-and-logging.md)。
版本纪律：`io/fs.ReadLinkFS` 与 `os.Root` 的多数方法需要 **Go 1.25+**；`io.ReadAll` 的性能改进与 `bytes.Buffer.Peek` 需要 **Go 1.26+**；`strings.CutLast` 与 `bytes.CutLast` 需要 **Go 1.27+**。

## 1. `io`：接口组合与拷贝

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `io.Reader` | `Read([]byte) (int, error)` | 唯一方法，一切围绕它组合 |
| `io.Writer` | `Write([]byte) (int, error)` | 写入全部字节或返回错误 |
| `io.Closer` | `Close() error` | 资源释放；`io.ReadCloser` 是 Reader 与 Closer 的组合 |

`Read` 的契约只有一句话：**返回读取到的字节数与遇到的错误；`n > 0` 时即使 `err != nil` 也必须先处理这 n 个字节。** **`io.EOF` 表示「正常读完」，不是错误**，判定必须用 `errors.Is(err, io.EOF)` 而不是 `==`（包装过的 EOF 用 `==` 判定不出来）。

```go
buf := make([]byte, 4096)
for {
	n, err := r.Read(buf)
	if n > 0 {
		// 先处理这 n 个字节
	}
	if err != nil {
		if errors.Is(err, io.EOF) {
			break
		}
		return err
	}
}
```

`io.ReadFull` 把「必须读满 len(buf) 个字节」表达清楚：读满返回 nil，读了一部分就结束返回 `io.ErrUnexpectedEOF`，解析定长头部时比手写循环可靠。

| 函数 | 作用 |
| --- | --- |
| `io.Copy(dst, src)` | 从 src 拷到 dst 直到 EOF，返回字节数；内部对 `WriterTo` / `ReaderFrom` 有优化 |
| `io.ReadAll(r)` | 读到 EOF，返回全部内容 |
| `io.TeeReader(r, w)` | 读 r 的同时把读到的内容写一份到 w |
| `io.MultiWriter(ws...)` | 一次写入复制给多个 Writer，任一失败即返回错误 |
| `io.LimitReader(r, n)` | 限制最多读 n 字节，防止超大输入 |

```go
limited := io.LimitReader(r.Body, 10<<20)         // 最多 10MiB
tee := io.TeeReader(limited, auditFile)           // 读的同时写审计文件
hash := sha256.New()                              // crypto/sha256
_, err := io.Copy(io.MultiWriter(dst, hash), tee) // 同时落到 dst 与哈希
```

`io.ReadAll` 在 **Go 1.26** 起更快、内存占用约为原来的一半，但语义没变：**把整个流读进内存**。对未知大小的输入必须先套 `io.LimitReader`。处理大文件用 `io.Copy`，内存占用与文件大小无关。

## 2. `io/fs`：文件系统抽象与嵌入

`fs.FS`（<https://pkg.go.dev/io/fs>）把「文件系统」抽象成接口，让代码可以在真实磁盘、`embed.FS`、内存测试替身之间切换。路径必须满足 `fs.ValidPath`：相对路径、以斜杠分隔、不含 `.` 与 `..`、不以斜杠开头、根目录写作 `.`（`fs.ValidPath("a/b.txt")` 为 true，`"/a/b.txt"` 与 `"../a.txt"` 都为 false）。

```go
//go:embed templates/*.html
var templates embed.FS

entries, err := fs.ReadDir(templates, "templates") // 列目录
sub, err := fs.Sub(templates, "templates")         // 挂到子路径
data, err := fs.ReadFile(sub, "index.html")        // 读文件

err = fs.WalkDir(templates, ".", func(path string, d fs.DirEntry, err error) error {
	if err != nil || d.IsDir() {
		return err // 返回非 nil 即终止遍历；目录返回 nil 表示继续下钻
	}
	fmt.Println(path, d.Name())
	return nil
})
```

`WalkDir` 回调返回非 nil 即终止遍历；返回 `fs.SkipDir` 跳过当前目录，返回 `fs.SkipAll` 结束整个遍历。`//go:embed` 是**编译期**指令，路径相对于当前源文件所在目录，不能出现 `..`；只用指令不用包名时写 `import _ "embed"`。被嵌入的文件成为二进制的只读数据，`embed.FS` 的时间戳为零值，依赖修改时间做缓存失效的逻辑对它无效。

**Go 1.25 起**新增 `io/fs.ReadLinkFS` 接口，用来表达「这个文件系统支持读取符号链接」；此前的 `fs.FS` 抽象没有读链接的能力。同一版本中 `os.DirFS` 与 `os.Root.FS` 都实现了它（可断言后调用其读链接能力），而 `embed.FS` 不支持（嵌入数据里没有链接概念）；接口的精确方法集以官方文档为准。

## 3. `os`：路径、进程与受限根目录

| 函数 | 说明 |
| --- | --- |
| `os.Open(name)` | 只读打开，返回 `*os.File`（实现了 `io.ReadCloser`） |
| `os.Create(name)` | 创建/截断为只写 |
| `os.OpenFile(name, flag, perm)` | 通用形式 |
| `os.ReadFile` / `os.WriteFile` | 一次读/写全部内容，内部自动 Close |
| `os.CreateTemp` / `os.MkdirTemp` | 创建唯一命名的临时文件/目录 |
| `os.Rename` / `os.RemoveAll` | 重命名、递归删除（路径不存在时返回 nil） |

flag 组合：`os.O_RDONLY` / `O_WRONLY` / `O_RDWR` 与 `os.O_CREATE` / `O_TRUNC` / `O_APPEND` / `O_EXCL` 按位或。**Go 1.26 起**，Windows 上 `os.OpenFile` 的 flag 支持 Windows 专有标志。

`os.Exit(n)` **立即终止进程，不执行任何 defer，也不等待其他 goroutine**，因此 `main` 只应调用一个返回 error 的 `run()`，由它保证 defer 生效：

```go
func main() {
	if err := run(); err != nil { // main 只做一件事
		fmt.Fprintln(os.Stderr, "fatal:", err)
		os.Exit(1)
	}
}

func run() error {
	f, err := os.Create("out.txt")
	if err != nil {
		return err
	}
	defer f.Close() // 正常返回路径上一定会执行
	return nil
}
```

| 项 | 说明 |
| --- | --- |
| `os.Args` | 命令行参数切片，`os.Args[0]` 是程序路径，业务参数从 `[1]` 开始 |
| `os.Getenv(key)` | 不存在时返回空字符串，无法区分「未设置」与「设为空」 |
| `os.LookupEnv(key)` | 返回 `(value, ok)`，需要区分时必须用它 |
| `os.Stdin` / `os.Stdout` / `os.Stderr` | 三个已打开的文件，类型均为 `*os.File` |

需要解析复杂命令行时用 `flag` 包（<https://pkg.go.dev/flag>），不要手写 `os.Args` 的下标逻辑。**`os.Root`（重点）**：拼接用户输入构造路径（`filepath.Join(base, userInput)`）无法阻止目录穿越——`userInput` 为 `../../etc/passwd` 或 Windows 上的 `..\..\config.yaml` 时，`filepath.Join` 只做字符串清理。`os.Root`（Go 1.24 引入）把「根目录」变成句柄，此后所有路径解析都在该根内进行，`..` 与符号链接都会被限制住。

```go
root, err := os.OpenRoot("data") // 这里只用受信任的常量路径
if err != nil {
	return err
}
defer root.Close()

f, err := root.Open(userPath) // 路径可来自用户输入：root 保证不会逃出 data/
if err != nil {
	return err
}
defer f.Close()
```

**Go 1.25 补齐的方法**（1.24 只提供打开/创建/删除一类基础操作）：

| 方法 | 用途 |
| --- | --- |
| `ReadFile` / `WriteFile` | 读入 / 写出整个文件 |
| `MkdirAll` / `RemoveAll` | 递归创建 / 递归删除 |
| `Rename` / `Link` / `Symlink` / `Readlink` | 重命名、创建硬链接与符号链接、读取链接目标 |
| `Chmod` / `Chown` / `Lchown` / `Chtimes` | 权限、所有者、时间戳（`Lchown` 不跟随符号链接） |

同一版本中 `os.DirFS` 与 `os.Root.FS` 实现了 `io/fs.ReadLinkFS`，因此可以把 `os.Root` 暴露成 `fs.FS` 供 `fs.WalkDir` 使用，同时保留不逃出根目录的保证。选择判据：路径含用户输入且必须限制在某个目录内时用 `os.Root`；路径完全由代码常量组成时直接用 `os` 的函数；要把目录当 `fs.FS` 交给别的库时，`os.DirFS` 无逃逸防护、`os.Root.FS` 有防护。

**Windows 上的异步 I/O（Go 1.25）**：`os.NewFile` 现在支持传入以异步方式打开的文件句柄，使这类句柄能被运行时正确纳入 poller 管理；手工从 `syscall` 层拿到句柄再包装时，需确保句柄确实以 `FILE_FLAG_OVERLAPPED` 打开。

## 4. `strings` 与 `bytes`

字符串不可变，`s += x` 每次都会分配新字符串并复制全部旧内容，循环 n 次的复制量是 O(n²)；`strings.Builder` 把复制摊到多次扩容上。

```go
// 反模式：O(n²)
var s string
for _, item := range items {
	s += item.Name + ","
}

// 正确：O(n)
var b strings.Builder
b.Grow(len(items) * 8) // 可选：预估容量，减少扩容
for _, item := range items {
	b.WriteString(item.Name)
	b.WriteByte(',')
}
s = b.String()
```

两条纪律：**不要复制 `Builder` 值**（传参用 `*strings.Builder`，不要放进切片或 map 值，否则副本共享底层 buffer）；`String()` 之后视为已定稿。benchmark 思路是写两个 `Benchmark` 函数分别跑 `+` 与 `Builder`，用 `go test -run '^$' -bench . -benchmem` 看 `allocs/op` 与 `B/op`，比只比时间更能说明问题。

| 函数 | 用途 | 注意 |
| --- | --- | --- |
| `strings.Cut(s, sep)` | 切成 `(before, after, found)` | 替代 `Index` + 两次切片；`found` 必须检查 |
| `strings.Fields(s)` | 按空白切分，忽略连续空白 | 与 `Split(s, " ")` 语义不同，后者保留空串 |
| `strings.Split` / `SplitN` | 按分隔符切分 | `SplitN(s, sep, -1)` 等价于 `Split` |
| `strings.Replace(s, old, new, n)` | 替换前 n 个 | `n < 0` 表示全部 |
| `strings.ReplaceAll` | 替换全部 | 是 `Replace` 的 `n = -1` 简写 |
| `strings.TrimSpace` / `TrimPrefix` / `TrimSuffix` | 去空白 / 去前后缀 | `TrimPrefix` 不匹配时原样返回 |
| `strings.EqualFold` | 忽略大小写比较 | 不要用 `ToLower` 后比较（会分配） |

```go
key, value, found := strings.Cut(line, "=")
if !found {
	return fmt.Errorf("bad line %q: missing '='", line)
}
```

**Go 1.27 起**新增 `strings.CutLast`，从最后一个分隔符处切分——解析「前缀相同、后缀才是关键」的字符串时不必先 `LastIndex` 再切片。`bytes` 包（<https://pkg.go.dev/bytes>）提供 `[]byte` 版本的同类操作（`bytes.Cut`、`bytes.Fields` 等），避免字符串与字节切片来回转换；**`bytes.CutLast` 需要 Go 1.27+**。

**Go 1.26 起**新增 `bytes.Buffer.Peek`，可以在**不消费**数据的前提下查看接下来的 n 个字节，适合「先看头部判断格式，再决定怎么解析」：

```go
// 需要 Go 1.26 及以上
head, err := buf.Peek(4)
if err != nil && !errors.Is(err, io.EOF) {
	return err
}
if bytes.HasPrefix(head, []byte("GZIP")) {
	// 仍可按 gzip 解析，数据未被消费
}
```

`Peek(n)` 的返回值在下次读写前有效，**不要长期持有**；它不推进读指针，因此 `Peek` 之后仍需正常读取。

## 5. `strconv`：与 `fmt` 的分工

热路径一律用 `strconv`：`fmt.Sprintf` 要走反射与接口装箱，`strconv` 是直接的数值转换；日志、错误文本等冷路径用 `fmt` 更可读。

```go
n, err := strconv.Atoi("42") // 最常用：十进制 int
if err != nil {
	return fmt.Errorf("parse count: %w", err)
}
big, err := strconv.ParseInt("7fffffffffffffff", 16, 64) // 指定进制与位宽
f, err := strconv.ParseFloat("3.14", 64)
s := strconv.FormatFloat(f, 'f', 2, 64) // FormatFloat 用 'f' 定点输出
q := strconv.Quote("a\nb")
raw, err := strconv.Unquote(q) // 与 Quote 互逆
```

- `Atoi` 等价于 `ParseInt(s, 10, 0)`，位宽是平台相关的 `int`；要固定宽度用 `ParseInt` 或 `Itoa` 的对应形式。
- `ParseFloat` 的第二个参数只能是 32 或 64；`FormatFloat` 的格式字符 `'f'`（定点）、`'e'`（科学计数）、`'g'`（自动）语义不同，金额展示用 `'f'` 并显式给精度。
- 浮点不能精确表示十进制小数，**金额一律用整数最小单位（分）**，不要用 `float64`。
- `strconv.ParseBool` 只接受 `1/t/T/TRUE/true/True/0/f/F/FALSE/false/False`，其他输入都报错。

## 6. `bufio`：缓冲与扫描

```go
sc := bufio.NewScanner(f)
sc.Buffer(make([]byte, 0, 64*1024), 1024*1024) // 初始 64KiB，上限 1MiB

for sc.Scan() {
	line := sc.Text() // 返回的字节可能被下一次 Scan 复用
	_ = line
}
if err := sc.Err(); err != nil { // 必须检查
	return fmt.Errorf("scan: %w", err)
}
```

| 坑 | 现象 | 处理 |
| --- | --- | --- |
| 默认单行上限 64KiB | 超长行时 `Scan` 返回 false，`Err()` 返回 `bufio.ErrTooLong` | 用 `sc.Buffer(buf, max)` 提高上限，或改用 `bufio.Reader` |
| 忘记检查 `sc.Err()` | 读取中途的真实 I/O 错误被当成「正常读完」 | 循环结束后必须检查 |
| 长期持有 `sc.Text()` 的结果 | 内容被下一次读取覆盖 | 需要保留时 `strings.Clone(sc.Text())` |

`Scanner` 默认按行切分（`bufio.ScanLines`），其他内置切分函数为 `ScanWords`、`ScanRunes`、`ScanBytes`。输入是「记录」而非「行」时，用 `bufio.Reader` 的按分隔符读取更直接：`chunk, err := r.ReadString('\n')`，返回的这一段与 err 一起处理。

`bufio.Writer` 把多次小写入合并成少数几次系统调用，**数据在 `Flush` 之前不一定落到底层**：

```go
w := bufio.NewWriterSize(f, 64*1024)
for _, line := range lines {
	if _, err := w.WriteString(line); err != nil {
		return err
	}
}
if err := w.Flush(); err != nil { // 必须检查：写失败往往只在 Flush 时暴露
	return err
}
```

规则：`Flush` 必须在返回前调用且返回值必须检查；缓冲区满时自动 Flush，因此中途的写错误会在 `Write` 系列调用上返回；完全依赖 `defer w.Flush()` 就会丢掉错误。

## 7. `time`

| 类型/函数 | 说明 |
| --- | --- |
| `time.Time` | 时间点（墙上时钟 + 可选的单调读数） |
| `time.Duration` | 时间段，底层是 `int64` 纳秒 |
| `time.Now()` | 当前时间 |
| `t.Add(d)` / `t.Sub(u)` | 时间点加减时间段 |
| `t.Truncate(d)` | 向下取整到 d 的整数倍 |
| `t.Format(layout)` / `time.Parse(layout, s)` | 格式化与解析 |
| `time.LoadLocation(name)` | 加载时区 |

```go
start := time.Now()
elapsed := time.Since(start)
fmt.Printf("elapsed=%dms\n", elapsed.Milliseconds())
now := time.Now().Truncate(time.Minute)
fmt.Println(now.Format(time.RFC3339))
sh, err := time.LoadLocation("Asia/Shanghai")
_ = now.In(sh) // 转换到目标时区
```

- layout 不是格式串，而是「参考时间」的字面写法（`2006-01-02 15:04:05`、`time.RFC3339`）；写错会得到静默的错误输出。
- `t.Truncate` 作用于绝对时间，对带时区的值按 UTC 边界截断；要按本地日历截断（例如「今天零点」）先 `In(loc)` 再截断。
- `time.LoadLocation` 依赖系统时区数据库，容器里未安装 `tzdata` 会失败。
- `time.Parse` 失败时返回零值时间与 error，**忽略 error 会得到 0001-01-01**，这是「时间莫名变成零值」的根因。
- `time.Now()` 返回的时间值里除了墙上时钟，还带一份**单调时钟读数**（进程启动以来的流逝时间）。

| 写法 | 使用哪个时钟 | 风险 |
| --- | --- | --- |
| `time.Since(start)` | 两个值都来自 `time.Now()`，优先用单调读数 | 不受系统时间调整影响 |
| `t2.Sub(t1)`，任一值来自 `Parse`、数据库、JSON | 只有墙上时钟 | 期间发生 NTP 校正或手工改时间，可能得到负数或巨大值 |

结论：**测量「经过多久」时起点必须来自 `time.Now()`，并且不要把这个时间值序列化后再用来算耗时。** 需要落库的耗时直接存 `elapsed.Milliseconds()`，而不是存两个时间点。

```go
ticker := time.NewTicker(2 * time.Second)
defer ticker.Stop() // 必须：不 Stop 会持续占用运行时资源
for {
	select {
	case <-ticker.C:
		_ = poll() // 实际项目里在此检查并返回错误
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

**Go 1.23 起 `time.Timer` 与 `time.Ticker` 的 channel 是无缓冲的。** 旧实现给 channel 一个容量为 1 的缓冲区，导致「事件已发出但还没被接收」的状态，也让 `Stop` 之后 channel 里是否还有值变得难以推理；改成无缓冲后投递与接收同步，语义变简单。**Go 1.27 起，控制该行为的 `asynctimerchan` 开关被永久移除，`time` 包的 channel 恒为无缓冲**，因此不要再依赖「`Stop()` 之后 channel 里可能还有一个值」的旧行为。`context` 与超时的完整设计见后续讲义 L11；本篇只需记住：不使用 context 的等待最好也带一个退出通道，否则 goroutine 无法回收。

## 8. 综合示例：流式清洗大文件

需求：从受限根目录读取一个可能很大的文本文件，逐行清洗（去空白、跳过空行与注释、转小写、去重统计），把结果写到同根目录下的另一个文件，内存占用与文件大小无关。

```go
import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

// run 把工作限制在 data/ 目录内，返回可被 main 统一处理的错误。
func run() error {
	root, err := os.OpenRoot("data") // Go 1.24+；路径不含用户输入
	if err != nil {
		return fmt.Errorf("open root: %w", err)
	}
	defer root.Close()

	in, err := root.Open("input.txt")
	if err != nil {
		return fmt.Errorf("open input: %w", err)
	}
	defer in.Close()

	out, err := root.Create("clean.txt")
	if err != nil {
		return fmt.Errorf("create output: %w", err)
	}
	defer out.Close()

	sc := bufio.NewScanner(in)
	sc.Buffer(make([]byte, 0, 64*1024), 4*1024*1024) // 支持最长 4MiB 的行
	w := bufio.NewWriterSize(out, 64*1024)

	seen := make(map[string]struct{})
	var total, kept int
	for sc.Scan() {
		total++
		line := strings.TrimSpace(sc.Text())
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}
		line = strings.ToLower(line)
		if _, dup := seen[line]; dup {
			continue
		}
		seen[line] = struct{}{}
		kept++
		if _, err := w.WriteString(line + "\n"); err != nil {
			return fmt.Errorf("write line %d: %w", total, err)
		}
	}
	if err := sc.Err(); err != nil { // 必须：区分读错误与正常读完
		return fmt.Errorf("scan input: %w", err)
	}
	if err := w.Flush(); err != nil { // 必须：检查 Flush 的错误
		return fmt.Errorf("flush output: %w", err)
	}
	fmt.Fprintf(os.Stderr, "total=%d kept=%d\n", total, kept) // 一条汇总，而不是每行一条
	return nil
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "fatal:", err)
		os.Exit(1)
	}
}
```

| 要点 | 体现 |
| --- | --- |
| 受限根目录 | `os.OpenRoot("data")` + `root.Open` / `root.Create` |
| 流式读取 | `bufio.Scanner` 逐行，内存与文件大小无关 |
| 超长行 | `sc.Buffer(buf, 4MiB)` |
| `sc.Err()` | 循环后显式检查并包装 |
| Flush 时机 | 循环后显式 `Flush` 并检查错误 |
| 去重统计 | `map[string]struct{}`（值类型不占额外空间） |
| 错误处理 | 一律 `%w` 包装，`main` 只负责退出码 |

`seen` 的增长与去重后的条目数成正比；若唯一行可能有上亿条，需改用外部排序或布隆过滤器，这属于「内存与数据基数相关」的问题，与文件大小无关。

## 9. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `os.Open` 后忘记 `defer f.Close()` | 文件描述符泄漏，最终 `too many open files` | 资源未释放 | 打开后立刻 defer；循环里打开文件时把处理抽成函数 |
| 循环后不检查 `sc.Err()` | 中途 I/O 错误或超长行被当成「读完」 | `Scan` 返回 false 有「EOF」与「出错」两种原因 | 循环后必须检查 |
| `defer w.Flush()` 后不检查返回 | 磁盘写满时静默丢数据 | Flush 是唯一暴露写失败的机会 | 显式 Flush 并检查 error |
| 把 `strings.Builder` 当值传递 | 复制后写入语义错乱 | `Builder` 内含不可复制的状态 | 传 `*strings.Builder`，不放进切片或 map 值 |
| `t, _ := time.Parse(layout, s)` | 时间变成 `0001-01-01 00:00:00 UTC` | 解析失败返回零值，错误被丢弃 | 检查 error 并 `%w` 包装 |
| `a / b` 不校验 b | `panic: integer divide by zero` | 除零是运行时 panic | 除之前判零并返回 error，或定义明确的零输入语义 |
| `int32` 累加大数 | 结果为负数等异常值 | 溢出静默回绕 | 用能覆盖量级的类型（如 `int64`），关键累加做范围校验 |
| `filepath.Join(base, userInput)` 当安全边界 | 可读到 base 之外的文件 | `Join` 只做字符串清理，不阻止 `..` | 用 `os.Root` 把解析限制在根内 |
| `string(b)` / `[]byte(s)` 放在循环里 | 大量分配，GC 压力高 | 两种转换都复制 | 循环外转换一次；字节处理统一用 `bytes` 包 |
| `ticker` 不 `Stop` | 定时器与 goroutine 泄漏 | 忘了显式停止 | `defer ticker.Stop()` |

## 10. 动手练习

1. **流式 vs 全量**：用 `io.Copy` 与 `io.ReadAll` 分别统计一个 1GiB 文件的字节数，用 `runtime` 包暴露的内存统计（<https://pkg.go.dev/runtime>）观察峰值内存差异。
2. **`os.Root` 防线**：写一个 HTTP handler 接受 `?file=` 参数返回文件内容，分别用 `filepath.Join` 与 `os.Root` 实现，再传 `../../go.mod` 验证两者差别。
3. **嵌入静态资源**：把 `templates/` 用 `//go:embed` 嵌入，用 `fs.WalkDir` 列出全部条目，再把 `embed.FS` 交给 `net/http` 提供的文件服务能力对外提供（见 <https://pkg.go.dev/net/http>）。
4. **拼接 benchmark**：用 `+`、`strings.Builder`、`bytes.Buffer` 三种方式拼接 10000 个字符串，跑 `-benchmem` 并解释 `allocs/op` 的差异。
5. **`Peek` 判格式与超长行**：用 `bytes.Buffer.Peek`（Go 1.26+）实现「先看头部 4 字节决定用 JSON 还是纯文本解析」的分支并断言数据未被消费；再构造含 1MiB 单行的文件，先用默认 `Scanner` 观察 `Err()`，后用 `sc.Buffer` 修复。

## 11. 自检清单

- [ ] 能说出 `Read` 的契约，知道 `n > 0 && err != nil` 时仍要处理这 n 个字节
- [ ] 能用 `errors.Is(err, io.EOF)` 正确判定正常结束
- [ ] 会用 `io.Copy` / `ReadFull` / `TeeReader` / `MultiWriter` / `LimitReader` 组合出流式管道
- [ ] 明确知道 `io.ReadAll` 是全量读入内存，对不可信输入必须先 `LimitReader`
- [ ] 知道 `fs.ValidPath` 的规则，会用 `embed.FS` + `//go:embed` 嵌入资源
- [ ] 知道 `io/fs.ReadLinkFS` 需要 **Go 1.25+**，且 `os.DirFS` 与 `os.Root.FS` 已实现它
- [ ] 能列出 `os.Root` 在 **Go 1.25** 新增的方法，并说清它防的是哪一类问题
- [ ] 知道 `os.Exit` 不执行 defer，且 `main` 只调用返回 error 的 `run()`
- [ ] 会用 `os.LookupEnv` 区分「未设置」与「设为空」
- [ ] 能解释 `strings.Builder` 相对 `+` 的性能优势，并知道不能复制它
- [ ] 会用 `strings.Cut`，并知道 `strings.CutLast` / `bytes.CutLast` 需要 **Go 1.27+**
- [ ] 知道 `bytes.Buffer.Peek` 需要 **Go 1.26+**，且 `Peek` 不消费数据
- [ ] 能说清热路径用 `strconv`、可读性场景用 `fmt` 的理由
- [ ] 会用 `Scanner.Buffer` 处理超长行，并永远检查 `Scanner.Err()`
- [ ] 知道 `bufio.Writer` 必须 Flush 且 Flush 的错误必须检查
- [ ] 能解释单调时钟，并说明耗时不该用序列化后的时间点相减
- [ ] 知道 **Go 1.23** 起 timer/ticker 的 channel 无缓冲，**Go 1.27** 永久移除了旧开关
- [ ] 知道 `Ticker` 必须 `Stop`，等待循环要同时处理 `ctx.Done()`

## 12. 延伸阅读

- Go 1.25 发布说明（`io/fs.ReadLinkFS`、`os.Root` 新方法、Windows 异步句柄）：<https://go.dev/doc/go1.25>
- Go 1.26 发布说明（`io.ReadAll` 优化、`bytes.Buffer.Peek`）：<https://go.dev/doc/go1.26>
- Go 1.27 发布说明（`strings.CutLast`、`bytes.CutLast`、timer channel 行为）：<https://go.dev/doc/go1.27>
- `io` 包：<https://pkg.go.dev/io>；`io/fs` 包：<https://pkg.go.dev/io/fs>；`os` 包：<https://pkg.go.dev/os>
- `strings` 包：<https://pkg.go.dev/strings>；`bytes` 包：<https://pkg.go.dev/bytes>；`strconv` 包：<https://pkg.go.dev/strconv>
- `bufio` 包：<https://pkg.go.dev/bufio>；`time` 包：<https://pkg.go.dev/time>
- 书籍：*Go 语言圣经*（The Go Programming Language 中文版），Alan A. A. Donovan、Brian W. Kernighan 著，机械工业出版社
- 相关讲义：[L01 工具链与 Go Modules](01-toolchain-and-modules.md)、[L02 语言精要与易错点](02-language-essentials.md)、[L03 错误处理与结构化日志](03-errors-and-logging.md)
