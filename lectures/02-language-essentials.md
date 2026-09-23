# L02 语言精要与易错点

本篇定位：把 Go 里「看起来会、实际常错」的部分逐条拆开，重点是切片与 map 的别名语义、接口的 nil 陷阱、方法集与可寻址性、泛型的版本边界。
对应学习周：W1–W2。前置要求：完成 [L01 工具链与 Go Modules](01-toolchain-and-modules.md)，能在本机 `go1.25.7` 下 `go run` 单文件程序。
版本纪律：标注「Go 1.26+」「Go 1.27+」的示例，都不能在 1.25 工具链上编译通过。

## 1. 变量、常量与零值

| 类型 | 零值 |
| --- | --- |
| 数值 / `bool` / `string` | `0` / `false` / `""`（字符串零值不是 nil） |
| 指针、切片、map、channel、函数、接口 | `nil` |
| 数组、struct | 逐字段取零值 |

「零值可用」是 Go 的设计目标：`sync.Mutex` 的零值就是可用的锁，`bytes.Buffer` 的零值就是空缓冲区（<https://pkg.go.dev/bytes>）。自定义类型也应尽量让零值有意义。

```go
var s []int
fmt.Println(s == nil, len(s), cap(s)) // true 0 0
type Counter struct{ total int; label string }
fmt.Printf("%+v\n", Counter{})        // {total:0 label:}
```

短变量声明的遮蔽陷阱：`:=` 只要有至少一个新变量就会创建新变量，不同块里的同名变量互相遮蔽。

```go
func f() error {
	var err error
	if true {
		_, err := doSomething() // 新变量，遮蔽了外层 err
		if err != nil {
			return err
		}
	}
	return err // 永远是 nil
}
```

`iota` 每个 `const` 块重置为 0、每行自增 1；显式写第一个常量的类型，后续行继承类型与表达式。

```go
type Status int

const (
	StatusPending Status = iota // 0
	StatusPaid                  // 1
	StatusShipped               // 2
	StatusDone                  // 3
)
```

- 用 `_` 占位可跳过值（例如让 0 表示「未设置」）。
- **`iota` 枚举不保证向后兼容**：中间插入成员会改变后续所有值；值会被持久化或跨进程传输时必须显式写出数值。
- 让枚举实现 `String()` 会改变 `fmt` 的打印结果（打印 `paid` 而不是 `1`）。
- 无类型常量有任意精度，只在被赋予具体类型时才截断（`const big = 1 << 40` 可赋给 `int64` 与 `float64`，赋给 `int32` 则编译错误）；默认类型为整数 `int`、浮点 `float64`、复数 `complex128`、符文 `rune`，布尔与字符串按字面量类型。

## 2. 数组与切片

数组长度是类型的一部分，赋值时整体复制；切片是「指针 + 长度 + 容量」的描述符，赋值只复制描述符，底层数组共享。

```go
a := [3]int{1, 2, 3}
b := a
b[0] = 99
fmt.Println(a[0], b[0]) // 1 99

s := []int{1, 2, 3}
t := s
t[0] = 99
fmt.Println(s[0], t[0]) // 99 99
```

| 表达式 | 含义 |
| --- | --- |
| `len(s)` | 当前可访问的元素个数 |
| `cap(s)` | 从 `s` 起始元素到底层数组末尾的元素个数 |

```go
s := make([]int, 3, 10)
fmt.Println(len(s), cap(s)) // 3 10
s = s[:7]
fmt.Println(len(s), cap(s)) // 7 10
// s = s[:11] // panic: slice bounds out of range
```

`append` 容量够时原地写入，不够时分配新数组并复制。扩容倍数是**实现细节，不要依赖**；但可以用 `cap` 观察「有没有换底层数组」这一事实。已知规模时用 `make([]T, 0, n)` 预分配；一旦扩容，旧切片与新切片不再共享底层数组，此前的别名关系静默失效。

```go
a := []int{1, 2, 3, 4}
b := a[:2]        // len 2 cap 4，与 a 共享底层数组
b = append(b, 99) // 容量够，直接写入 a[2]
fmt.Println(a)    // [1 2 99 4]，a 被意外改写

b = a[:2:2]                      // 修复一：三索引切片限制 cap，append 必然重新分配
b = append([]int(nil), a[:2]...) // 修复二：先显式复制
```

三维切片表达式 `s[a:b:c]`：`0 <= a <= b <= c <= cap(s)`，结果 `len = b - a`、`cap = c - a`。它让调用方无法通过 `append` 写到 `c` 之后的位置，是把「只读视图」交给他人代码的标准手段。

`slices` 自 Go 1.21 起提供泛型切片操作（<https://pkg.go.dev/slices>）：

| 用途 | 说明 |
| --- | --- |
| 克隆 | `slices.Clone`，得到不共享底层数组的副本 |
| 排序 | `slices.Sort`（有序类型）、`slices.SortFunc`（自定义比较函数） |
| 查找与比较 | `slices.Contains`、`slices.Index`、`slices.Equal`、`slices.Compare` |
| 修改 | `slices.Delete`、`slices.Insert`、`slices.Reverse` |
| 迭代器 | Go 1.23 起提供返回迭代器的版本 |

函数名与签名以官方文档为准，本讲义不逐一复制签名。注意 `slices.Delete` 只移动元素并返回新长度，**不缩短底层数组**；元素含指针时尾部残留会造成内存无法回收，必要时把尾部置零。

## 3. map

```go
var m map[string]int
fmt.Println(m["x"], len(m)) // 0 0，读 nil map 合法
// m["x"] = 1                // panic: assignment to entry in nil map

m2 := make(map[string]int, 8) // 正确：先 make
m2["x"] = 1
v, ok := m2["a"]              // 0 false：用 ok 判断键是否存在
v, ok = m2["x"]               // 1 true
delete(m2, "x")               // 删除不存在的键是合法 no-op
_, _, _ = v, ok, m
```

用 `ok` 而不是「值是否为零值」判断键是否存在，否则无法区分「键不存在」与「值恰好是零值」。

`range` 遍历 map 的顺序是随机的，每次运行都可能不同，测试里依赖顺序必然随机失败；需要稳定顺序时先取键再 `slices.Sort`。`maps` 自 Go 1.21 起提供泛型 map 操作（<https://pkg.go.dev/maps>）：取键集合、克隆、相等比较、按键排序遍历、删除满足条件的元素，以及 Go 1.23 起的迭代器版本；函数名与签名以官方文档为准。

并发读写 map 会 panic：

```go
m := map[int]int{}
go func() {
	for i := 0; ; i++ {
		m[i] = i
	}
}()
go func() {
	for {
		_ = m[1]
	}
}()
select {} // 运行结果：fatal error: concurrent map writes
```

- 这是 **fatal error**，不是 panic，`recover` 拦不住，进程直接退出。
- 触发是概率性的：写少、键少、并发度低时可能长时间不出现，压测时才暴露。
- 正确做法：读多写少用 `sync.RWMutex`；只做计数用 `sync/atomic`（Go 1.19 起提供 `atomic.Int64` 等类型）；需要并发读写键集合时用 `sync.Map`（<https://pkg.go.dev/sync#Map>）。用 `go test -race` 可提前发现。

## 4. struct

| 结构 | 能否 `==` 比较 / 做 map 键 |
| --- | --- |
| 全部字段可比较 | 可以 |
| 含 slice、map、func 字段 | 不可以（编译错误） |
| 含接口字段 | 可编译；动态类型不可比较时运行时 panic |

```go
type B struct {
	X    int
	Data []byte
}
// fmt.Println(B{} == B{}) // 编译错误：B 不可比较
var i1, i2 any = []int{1}, []int{1}
// fmt.Println(i1 == i2)   // panic: comparing uncomparable type []int
```

嵌入与提升：嵌入类型的字段与方法被提升到外层；外层同名成员遮蔽被提升的成员；冲突的同级提升导致「ambiguous selector」编译错误，不会随机选一个；嵌入接口时提升的是接口方法；嵌入 `*Base` 时提升的是 `*Base` 的方法集。

```go
type Base struct{ ID int }

func (b Base) Describe() string { return "base" }

type User struct {
	Base // 嵌入：字段名是 Base
	Name string
}

u := User{Base: Base{ID: 1}, Name: "n"}
fmt.Println(u.ID, u.Describe()) // 等价于 u.Base.ID、u.Base.Describe()
```

内存对齐：用 `unsafe` 包（<https://pkg.go.dev/unsafe>）的 `Sizeof` 可以直接测量。在 `GOARCH=amd64` 下 `int64` 需要 8 字节对齐，结构体整体对齐取最大字段的对齐值。

```go
type BadOrder struct {
	A bool  // 1 字节 + 7 填充
	B int64 // 8
	C bool  // 1 字节 + 7 填充
}

type GoodOrder struct {
	B int64 // 8
	A bool  // 1
	C bool  // 1 + 6 填充
}

fmt.Println(unsafe.Sizeof(BadOrder{}), unsafe.Sizeof(GoodOrder{})) // 24 16
```

**只在结构体会被大量分配时才调整字段顺序**；对每请求只建几个的结构体抠 8 字节收益为零。

结构体标签是编译期可见、运行期通过反射读取的字符串，格式约定为 `key:"value"`，空格分隔多个键，例如 `json:"id"`、`json:"name,omitempty"`、`json:"-"`（不参与序列化）。写错 key 不会编译失败，只在运行时被忽略，必须靠测试覆盖。

## 5. 接口

接口是隐式实现的，不需要 `implements` 声明；`var _ Speaker = Dog{}` 是编译期断言，类型不再满足接口时这一行先失败。

```go
type Speaker interface{ Speak() string }

type Dog struct{}

func (Dog) Speak() string { return "woof" }

var s Speaker = Dog{}
var _ Speaker = Dog{}
_ = s
```

```go
var v any = "hello"

s, ok := v.(string) // 安全形式：失败时 ok=false，s 为零值
// n := v.(int)     // 不安全形式：失败时 panic

switch x := v.(type) {
case nil: // 只在接口值完全为 nil（类型与数据都为空）时命中
	fmt.Println("nil")
case string:
	fmt.Println("string:", x)
case interface{ Len() int }: // case 可以是接口
	fmt.Println("has Len")
default:
	fmt.Printf("other: %T\n", x)
}
```

上例中的 `switch x := v.(type)` 就是 `type switch`：它按动态类型分支，而不是按值比较。接口值由「类型」与「数据」两个字组成，**接口为 nil 要求两个字都为空**；把类型化 nil 指针放进接口后，接口本身不为 nil。

```go
type MyError struct{ Msg string }

func (e *MyError) Error() string { return e.Msg }

func bad() error {
	var p *MyError
	return p // 返回 (*MyError)(nil)，但 error 接口非 nil
}

err := bad()
fmt.Println(err == nil) // false
fmt.Println(err)        // <nil>：Error() 在 nil 接收者上被调用
if err != nil {
	fmt.Println("进了错误分支，但打印出来是 <nil>")
}
```

判据：**函数返回接口类型时，永远不要返回「具体类型的 nil 变量」**；要么 `return nil`，要么返回真实值。接口值的内存布局是实现细节，但「接口 = 类型 + 数据」这一模型足以推出三个现象：接口赋值通常需要一次堆分配（编译器能证明不逃逸时可优化）；接口比较先比类型再比数据；数据字为空、类型字非空时接口不为 nil。

## 6. 方法集与接收者选择

| 接收者写法 | `T` 的方法集 | `*T` 的方法集 |
| --- | --- | --- |
| `func (t T) M()` | 含 `M` | 含 `M` |
| `func (t *T) M()` | 不含 | 含 `M` |

推论：指针接收者实现接口时**只有 `*T` 满足该接口**（`var _ Sayer = B{}` 会编译错误，`&B{}` 才行）；值接收者实现时 `T` 与 `*T` 都满足。

可寻址性：编译器允许 `x.M()` 在 `M` 是指针接收者方法时写成 `(&x).M()`，前提是 `x` **可寻址**。

```go
type Counter struct{ n int }

func (c *Counter) Inc()    { c.n++ }
func (c Counter) Get() int { return c.n }

var c Counter
c.Inc() // 等价 (&c).Inc()
// Counter{}.Inc()                 // 编译错误：不可寻址
// map[string]Counter{}["k"].Inc() // 编译错误：map 元素不可寻址
```

不可寻址的典型来源：结构体字面量、map 元素、函数返回值、常量。这也是 `map[string]struct{}` 装可变对象行不通的原因——要改只能取出、修改、写回。

| 判据 | 选择 |
| --- | --- |
| 方法需要修改接收者，或类型含 `sync.Mutex` 等不可复制字段 | 指针接收者 |
| 类型较大（经验值超过几十字节） | 指针接收者，避免每次调用复制 |
| 类型很小且不可变（如 `type ID int64`） | 值接收者 |
| 同一类型的所有方法 | **保持一致**，混用会让方法集语义变复杂 |

## 7. 泛型

`~T` 是**近似类型**，表示「底层类型是 `T` 的所有类型」，因此 `type Celsius float64` 也满足 `~float64`；`comparable` 是预声明约束，表示支持 `==` 与 `!=`，是 map 键的泛型前提。

```go
type Number interface {
	~int | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
	var total T
	for _, x := range xs {
		total += x
	}
	return total
}
```

类型推断：调用时通常不需要显式写类型参数（`Sum([]int{1, 2, 3})` 推断出 `T = int`）。**Go 1.27 起**，泛型函数被赋值给（或转换为）匹配的函数类型时也参与推断：`var f func([]int) int = Sum`。

**Go 1.26 起**，泛型类型可以在自身的类型参数列表中引用自身：`type Adder[A Adder[A]] interface { Add(A) A }`，让「返回自身类型」的约束不必借助额外的辅助接口。

**Go 1.27 起**，方法声明可以自带类型参数：

```go
// 需要 Go 1.27 及以上，1.25 / 1.26 无法编译
type Slice[E any] []E

func (s Slice[E]) Map[R any](f func(E) R) []R {
	out := make([]R, 0, len(s))
	for _, v := range s {
		out = append(out, f(v))
	}
	return out
}
```

两条硬限制：**接口的方法不能声明类型参数**（`Map[R any](func(E) R) []R` 这样的接口方法非法）；**接口方法也不能由泛型方法实现**，即使签名看起来一致。需要「多态映射」时把类型参数提到接口上，或把映射函数作为普通参数传入。`math/rand/v2` 的 `Rand` 在 Go 1.27 新增泛型方法 `N`，就是这一能力的实际用例（<https://pkg.go.dev/math/rand/v2>）。

## 8. 循环变量与 `for range`

**Go 1.22 起循环变量每轮独立**：

```go
var fns []func()
for _, v := range []int{1, 2, 3} {
	fns = append(fns, func() { fmt.Print(v) })
}
for _, f := range fns {
	f()
}
// Go 1.21 及以前输出 333（整个循环只有一个变量）
// Go 1.22 及以后输出 123（每轮迭代创建新变量）
```

因此 `v := v` 这种旧写法不再必要。两个边界：改变只针对 `for` 语句中**由 `:=` / `range` 声明的变量**（循环外 `var v int` 再在循环内赋值仍是同一个变量）；语义由 `go.mod` 的 `go` 行决定，模块写 `go 1.21` 时即使工具链是 1.25 也按旧语义编译。

**Go 1.22 起 `for range` 支持整数**：`for i := range 3` 等价于从 0 到 2，n 为 0 或负数时不迭代。

**Go 1.23 起支持对函数使用 range**，并引入 `iter` 包（<https://pkg.go.dev/iter>）：

```go
func Count(n int) func(yield func(int) bool) {
	return func(yield func(int) bool) {
		for i := range n {
			if !yield(i) {
				return // 调用方 break 时必须立刻停止
			}
		}
	}
}

for v := range Count(3) {
	fmt.Println(v)
}
```

- 被遍历方**必须**在 `yield` 返回 `false` 后立即返回，否则会泄漏资源（例如忘了关文件）；调用方 `break`、`return`、`goto` 跳出循环都会让 `yield` 返回 `false`。
- `iter` 包还提供两返回值版本（常用于 map 迭代）；类型名与签名以官方文档为准。

| 可 range 的对象 | 起止版本 |
| --- | --- |
| 数组、数组指针、切片、字符串、map、channel | 1.0 |
| 整数 | Go 1.22 |
| 符合约定签名的函数 | Go 1.23 |

## 9. 内建函数与版本差异

| 内建 | 引入版本 | 说明 |
| --- | --- | --- |
| `min` / `max` | 1.21 | 支持有序类型，可变参数 |
| `clear` | 1.21 | 清空 map（删除全部键）或把切片元素置零 |
| `new(expr)` | **1.26+** | `new` 的运算对象可以是表达式，直接取得带初值的指针 |

```go
fmt.Println(min(3, 5), max(3, 5)) // 3 5
m := map[string]int{"a": 1}
clear(m)                          // m 变为空 map
s := []int{1, 2, 3}
clear(s)                          // s 变为 [0 0 0]，长度不变
type Config struct{ TimeoutSec *int } // 可选字段，nil 表示使用默认值
cfg := Config{TimeoutSec: new(60)}    // 需要 Go 1.26 及以上：得到指向 60 的 *int
```

`new(expr)` 解决「可选字段需要指针以区分『未设置』与『零值』」这一痛点：1.25 及以前必须先声明变量再取地址，或写一个返回指针的辅助泛型函数。

## 10. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| `b := a[:2]; b = append(b, x)` | `a` 的元素被意外改写 | `b` 的 cap 覆盖到 `a` 后半段 | 用 `a[:2:2]` 限制 cap，或先 `slices.Clone` |
| 多个 goroutine 读写同一个 map | `fatal error: concurrent map writes` | map 非并发安全，且该错误无法 `recover` | `sync.RWMutex`、`sync.Map` 或 `sync/atomic` |
| `var p *T; return p`（返回接口类型） | `err != nil` 为真但打印是 `<nil>` | 接口含「类型 + 数据」，类型非空则接口非 nil | 直接 `return nil`，或显式判空后再返回 |
| `defer fmt.Println(x)` 之后又改 `x` | 打印的是旧值 | `defer` 实参在 `defer` 语句执行时立即求值 | 用闭包：`defer func() { fmt.Println(x) }()` |
| 热路径反复 `[]byte(s)` / `string(b)` | CPU 与 GC 压力偏高 | 两种转换都会复制 | 只在边界转换一次；需要零拷贝时用 `unsafe.String` 等并自行保证生命周期 |
| `fmt.Printf("%v", ptr)` 想打印字段 | 输出 `&{1 n}` 或地址 | `%v` 对指针解引用一层 | 要字段名用 `%+v`，要地址用 `%p` |
| `for _, v := range items { ptrs = append(ptrs, &v) }` | 1.21 及以前所有指针指向同一变量 | 循环变量语义随 Go 1.22 改变，且受 `go` 行影响 | 写 `&items[i]`，语义与版本无关 |
| `goto` 跳过变量声明 | 编译错误 `jumps over declaration` | `goto` 不能跨越变量声明进入其作用域 | 把目标放到声明之前，或改用 `break` / 函数拆分 |
| `if cond { x, err := f() }` 后在块外查 `err` | 外层 `err` 永远为 nil | `:=` 在块内创建了新变量 | 块外用 `=` 赋值，或把错误处理收进块内 |
| 用「是否零值」判断 map 键是否存在 | 存了零值的键被当成不存在 | 无法区分「不存在」与「值为零值」 | 用 `v, ok := m[k]` |
| 比较含 slice/map 字段的 struct | 编译错误；经接口比较则运行时 panic | 含不可比较类型的 struct 不可比较 | 手写字段比较或用 `slices.Equal` |
| 用 `iota` 枚举做持久化 | 中间插入成员后历史数据错位 | `iota` 值随声明顺序变化 | 显式写数值，或只追加不改序 |
| 结构体字段随意排序 | 大批量对象内存占用偏高 | 对齐填充 | 只在高频大批量场景调整，并用 `unsafe.Sizeof` 验证 |

## 11. 动手练习

1. **切片别名**：写 `AppendOne(s []int, v int) []int`，验证 `cap` 充足与不足两种情况下对入参的影响，再用三索引切片写出「绝不污染入参」的版本。
2. **接口 nil**：写两个「看起来返回 nil」的函数，一个返回 `error(nil)`，一个返回 `(*MyError)(nil)`，打印 `err == nil` 并用 `%T` 与 `%v` 解释差异。
3. **可寻址性**：分别对结构体字面量、map 元素、函数返回值调用指针接收者方法，记录三条编译错误信息。
4. **对齐测量**：为同一业务结构体写两种字段排列，用 `unsafe.Sizeof` 打印体积并计算填充字节数；再复现一次 `fatal error: concurrent map writes` 并用 `sync.RWMutex` 修复。
5. **版本边界**：用 `~` 定义接受 `type Celsius float64` 的 `Sum`；再写一个 Go 1.27 才合法的泛型方法示例并在注释中标注所需版本。

## 12. 自检清单

- [ ] 能说出所有类型的零值，并知道 nil map 可读不可写
- [ ] 能解释 `len` 与 `cap` 的区别，并说明 `append` 何时换底层数组
- [ ] 能写出 `s[a:b:c]` 三个下标的约束与用途
- [ ] 知道 `slices` 与 `maps` 自 Go 1.21 起提供，并能查到具体函数签名
- [ ] 能复现并解释 `fatal error: concurrent map writes` 为什么不能 `recover`
- [ ] 能用 `unsafe.Sizeof` 说明字段顺序对结构体体积的影响，并用「类型 + 数据」模型解释接口的 nil 陷阱
- [ ] 能背出值接收者与指针接收者的方法集差异，并判断表达式是否可寻址
- [ ] 能说出 `~T` 与 `comparable` 的语义，并明确知道泛型方法需要 **Go 1.27+**、接口方法不能带类型参数
- [ ] 明确知道泛型类型自引用与 `new(expr)` 需要 **Go 1.26+**
- [ ] 能解释 Go 1.22 前后循环变量语义差异，并说明为什么不再需要 `v := v`
- [ ] 知道 `for range` 支持整数（1.22）与函数（1.23，`iter` 包），并会用 `min` / `max` / `clear`

## 13. 延伸阅读

- Go 1.22 发布说明（循环变量、整数 range）：<https://go.dev/doc/go1.22>
- Go 1.23 发布说明（`iter`、range-over-func）：<https://go.dev/doc/go1.23>
- Go 1.26 发布说明（`new(expr)`、泛型类型自引用）：<https://go.dev/doc/go1.26>；Go 1.27 发布说明（泛型方法）：<https://go.dev/doc/go1.27>
- 泛型教程：<https://go.dev/doc/tutorial/generics>；切片内部结构：<https://go.dev/blog/slices-intro>
- `slices` 包：<https://pkg.go.dev/slices>；`maps` 包：<https://pkg.go.dev/maps>；`iter` 包：<https://pkg.go.dev/iter>；`unsafe` 包：<https://pkg.go.dev/unsafe>
- 书籍：*Go 语言程序设计*（The Go Programming Language），Alan A. A. Donovan、Brian W. Kernighan 著，机械工业出版社
- 相关讲义：[L01 工具链与 Go Modules](01-toolchain-and-modules.md)、[L03 错误处理与结构化日志](03-errors-and-logging.md)
