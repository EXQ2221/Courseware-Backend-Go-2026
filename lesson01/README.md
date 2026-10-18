# 蓝山工作室——Golang第一节课

### 前言

上一节课（第〇节）我们认识了 Go 程序的基本结构，把数据类型、变量、常量、运算符、分支与循环、函数这些最基础的语法过了一遍。

这节课是正式课程的第一节，我们要讲的是 Go 里最常用的几种数据结构，以及函数的进阶用法：

- 数组 array 与切片 slice
- 字典 map
- 函数进阶：函数类型、高阶函数、闭包
- 值类型与引用类型
- go test 单元测试

内容比上节课抽象一些，尤其是**函数进阶**那一部分，第一遍听不懂很正常。先留个印象，后面写代码写多了自然会回来理解它。

***

## 一、数组 array

数组（array）是一段**定长**的、同类型元素的连续存储。同一类型的数组，长度也是类型的一部分。

### 初始化

```go
// var arr [len]type
// arr := [len]type{data1, data2, ...}

a := [3]int{1, 2}            // 未初始化的元素为 0
b := [...]int{1, 2, 3}       // 用初始化值的个数推导长度
c := [5]int{2: 100, 4: 200}  // 用索引初始化，其他位置为 0
fmt.Println(a, b, c)         // [1 2 0] [1 2 3] [0 0 100 0 200]
```

`[...]` 不是省略写法，而是让编译器帮我们数一数右边有几个元素。

### 多维数组

```go
d := [4][2]int{{10, 11}, {20, 21}, {30, 31}, {40, 41}}
e := [...][2]int{{10, 11}, {20, 21}, {30, 31}, {40, 41}} // 第二维不能写 "..."
f := [4][2]int{1: {20, 21}, 3: {40, 41}}                 // 只给部分行赋值
g := [4][2]int{1: {0: 20}, 3: {1: 41}}                   // 精确到某一行的某一列
fmt.Println(d, e, f, g)
```

> 数组的长度必须是常量，而且长度是类型的一部分，所以 `[2]int` 和 `[3]int` 是**两个不同的类型**，不能互相赋值。

### 遍历数组

```go
var a [10]int

// 方法一：普通 for 循环
for i := 0; i < len(a); i++ {
	a[i] = i + 1
	fmt.Printf("a[%d] = %d\n", i, a[i])
}

// 方法二：for range
// range 有两个返回值，第一个是下标，第二个是元素的值
for i, v := range a {
	fmt.Println("a[", i, "]=", v)
}

// 只要值不要下标，用空白标识符 _ 把下标丢掉
for _, v := range a {
	fmt.Println(v)
}
```

> 数组是**值类型**。把数组赋给另一个变量、或者传进函数，都是把整个数组**拷贝一份**，后面讲"值类型与引用类型"时我们再细看这件事。

***

## 二、切片 slice

### 为什么需要切片

数组有两个绕不开的限制：

1. 长度定义后就改不了；
2. 是值类型，每次传递都要复制一整份。

显然这满足不了真实开发的需求（比如读一个不确定长度的文件、动态收集一批数据）。Go 给出的方案就是**切片（slice）**：

- 长度可动态增长；
- 传递时只拷贝一个很小的结构体（指针 + 长度 + 容量），不拷贝底层数据。

### 初始化

```go
// 1. 声明一个 nil 切片（没有被分配底层数组）
var s1 []int

// 2. 空切片，已经分配了底层数组，但长度为 0
s2 := []int{}

// 3. make([]T, length, capacity)，capacity 省略时等于 length
s3 := make([]int, 0)     // len=0 cap=0
s4 := make([]int, 5)     // len=5 cap=5
s5 := make([]int, 2, 5)  // len=2 cap=5

// 4. 字面量，顺便初始化几个值
s6 := []int{1, 2, 3}
```

写法上和数组就差一个中括号里的长度：`[3]int` 是数组，`[]int` 是切片。

> `var s1 []int` 和 `s2 := []int{}` 看起来都是"空"，但前者是 `nil`，后者不是。区别在于：`nil` 切片可以直接 `append`、可以直接 `range`，但它和 `[]int{}` 在某些场景（比如序列化成 JSON）表现不同——`nil` 会变成 `null`，`[]int{}` 会变成 `[]`。

### 长度与容量

- `len(s)`：当前元素个数
- `cap(s)`：底层数组从切片起始位置往后的可用空间

```go
s := make([]int, 2, 5)
fmt.Println(len(s), cap(s)) // 2 5
s = append(s, 1, 2, 3)
fmt.Println(len(s), cap(s)) // 5 5
```

长度不能超过容量：`len(s) <= cap(s)` 永远成立。

### 切片表达式

```go
s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}

s[:]      // 等价于 s[0:len(s)]
s[2:5]    // [2 3 4]，len=3，cap=8
s[:5]     // [0 1 2 3 4]
s[5:]     // [5 6 7 8 9]
s[2:5:7]  // [2 3 4]，len=3，cap=5
```

三索引切片 `s[low:high:max]` 中的 `max` 用来**限制容量**：`len = high-low`，`cap = max-low`。它的用处后面讲"切片共享底层数组"时会体现。

> 三个规则要记住：`0 <= low <= high <= max <= cap(s)`；左闭右开，即取到 `high-1`；`low` 省略为 0，`high` 省略为 `len(s)`。

### append 与扩容

`append` 向切片尾部追加元素，**返回一个新的切片**：

```go
var s []int            // nil 切片
s = append(s, 1)       // 追加 1 个
s = append(s, 2, 3)    // 追加 2 个
s = append(s, 4, 5, 6) // 追加 3 个
fmt.Println(s)         // [1 2 3 4 5 6]

s2 := make([]int, 5, 5)
fmt.Println(s2)                 // [0 0 0 0 0]
s2 = append(s2, 6)              // 容量不够，会分配新的底层数组
fmt.Println(s2, len(s2), cap(s2)) // [0 0 0 0 0 6] 6 10

other := []int{7, 8}
s = append(s, other...)         // 追加另一个切片，注意三个点不能少
```

`append` 会智能地增长切片的容量：一旦超过原底层数组的容量，就会重新分配一个更大的底层数组，并把原来的数据复制过去。

关于扩容，可以自己跑一下这段代码，把每次容量变化打出来：

```go
func main() {
	var s []int
	prevCap := cap(s)
	for i := 0; i < 20; i++ {
		s = append(s, i)
		if cap(s) != prevCap {
			fmt.Printf("len=%2d cap=%2d %v\n", len(s), cap(s), s)
			prevCap = cap(s)
		}
	}
}
```

你会看到容量按 1、2、4、8、16、32……这样跳。**"容量不够就重新分配一个更大的底层数组，并把老数据拷过去"**，这就是扩容的全部思想。至于每次扩大多少倍，是运行时的实现细节：早期版本简单粗暴地按 2 倍增长，Go 1.18 之后改成小切片翻倍、大切片按约 1.25 倍增长，中间还有一段平滑过渡。

> 所以面试里被问"切片扩容是几倍"的时候，标准答案不是"2 倍"，而是"看版本、看大小"。那例外到底是什么？自己动手把容量打到 200 以上看看:smirk:

### 切片共享底层数组

这一小节是本节课的**重点**，也是新手最容易踩坑的地方。

切片本身并不持有数据，它只是一个"视图"，指向一个**底层数组**。所以多个切片可能指向同一个底层数组，改一个会影响另一个。

```go
func main() {
	s3 := []int{1, 2, 3, 4, 5, 6, 7, 8} // len=8 cap=8
	s4 := s3[3:6]                        // [4 5 6]，len=3 cap=5，和 s3 共用底层数组
	s5 := s4[0:cap(s4)]                  // [4 5 6 7 8]，len=5 cap=5
	s6 := append(s4, 9)                  // 注意这一行

	fmt.Println(s3) // 猜猜输出什么？
	fmt.Println(s4)
	fmt.Println(s5)
	fmt.Println(s6)
}
```

`s4` 还有 2 个空闲容量，所以 `append` **不会**分配新数组，而是直接往后写——于是 `s3` 的第 7 个元素被悄悄改成了 9，`s5` 也跟着变。

> 把上面每个切片都 `Println` 出来，自己推一遍它们的值，你就彻底理解"slice 总是指向一个底层 array"这句话了。看不出来就在每一步加 print。

想避免这种"误伤"，可以一开始就用三索引切片把容量卡死：

```go
s4 := s3[3:6:6] // len=3 cap=3，此时 append 一定会扩容，不会动到 s3
```

如果确实需要一份独立的拷贝，用 `copy`：

```go
src := []int{1, 2, 3}
dst := make([]int, len(src))
n := copy(dst, src) // 返回实际拷贝的元素个数
fmt.Println(dst, n) // [1 2 3] 3
```

> 思考：上节课说过函数是一等公民，那切片里可以放函数吗？可以，`[]func()` 是合法的类型。那 `map[string]func()` 呢？也可以，这就是后面 Web 框架里"路由表"的原型。

***

## 三、字典 map

前面讲的数组、切片都是"单一元素的容器"，装的是一串同类型的值。map 不一样，它装的是**键值对**。

### 基础用法

```go
func main() {
	// 方法一：make 函数
	ages := make(map[string]int)

	// 方法二：字面量
	grades := map[string]string{
		"Alice": "A",
		"Bob":   "B",
	}

	// 增加 / 修改
	ages["Alice"] = 25
	ages["Bob"] = 30
	ages["Charlie"] = 22
	ages["Alice"] = 26 // 键已存在，就是修改

	// 查询：comma ok 写法
	aliceAge, exists := ages["Alice"]
	if exists {
		fmt.Printf("Alice's age: %d\n", aliceAge)
	} else {
		fmt.Println("Alice not found.")
	}

	// 遍历
	for name, age := range ages {
		fmt.Printf("%s is %d years old.\n", name, age)
	}

	// 删除
	delete(ages, "Bob")

	// 长度
	fmt.Println(len(ages))
}
```

### 键和值能放什么类型？

这是常考的一个点：**Go 字典的键类型不能是函数类型、字典类型和切片类型。**

原因是 Go 语言规范规定，键类型的值之间必须能使用 `==` 和 `!=` 比较。而函数、map、切片都不支持判等（切片只能和 `nil` 比），所以不能当键。

值的类型没有这个限制，随便放。

```go
m1 := map[string][]int{"a": {1, 2}}       // 值可以是切片
m2 := map[int]func() string{}             // 值可以是函数
// m3 := map[[]int]string{}               // 编译错误：切片不能作为键
```

### 几个容易踩的坑

**1. nil map 不能写入**

```go
var m map[string]int
m["a"] = 1 // panic: assignment to entry in nil map
```

要用得先 `make`。读是没问题的，读一个 `nil` map 只会拿到零值。

**2. 取不存在的键不会报错**

返回的是值类型的零值，所以要靠 comma ok 判断"到底有没有这个键"：

```go
v, ok := ages["NotExist"] // v = 0, ok = false
```

**3. 遍历顺序是随机的**

```go
for k := range ages {
	fmt.Println(k) // 每次运行顺序都可能不同
}
```

> 注意一个迷惑现象：`fmt.Println(map)` 打印出来的结果是**按 key 排好序**的，这是 `fmt` 包帮你排的，不代表 map 本身有序。写业务逻辑时千万不要依赖 map 的顺序。

**4. map 里的值本身不可取地址**

```go
type User struct {
	Name string
	Age  int
}

users := map[string]User{"u1": {"Tom", 18}}
// u := users["u1"]      // 拿到的是值的副本
// users["u1"].Age = 19  // 编译错误：不能寻址

u := users["u1"]
u.Age = 19
users["u1"] = u // 要整体写回去
```

> 有能力的同学可以了解一下 map 的底层实现——和哈希有关（哈希函数、哈希和、哈希碰撞）。另外提醒一句：map 不是并发安全的，多个 goroutine 同时读写会直接 panic，这块我们并发课上再讲。

***

## 四、函数进阶

### 函数是一等公民

在 Go 里，函数不只能封装代码，它本身还能**像普通值一样**传递、赋值、做类型判断和转换，就像切片和 map 一样。

```go
type Printer func(contents string) (n int, err error)

func printToStd(contents string) (int, error) {
	return fmt.Println(contents)
}

func main() {
	var p Printer
	p = printToStd
	p("something")
}
```

这里的 `Printer` 是一个**函数类型**。注意写法：类型声明名称右边是 `func` 关键字，再右边是参数列表和结果列表——和写函数声明几乎一样，只是把函数名换成了 `func`。

**函数签名**就是参数列表和结果列表的统称。参数名、结果名都不算签名，函数名也不算。

> 只要两个函数的**参数列表和结果列表的元素顺序及其类型**一致，我们就说它们是同一个函数类型的实现。所以上面的 `printToStd` 可以直接赋给 `Printer` 类型的变量，即使它的名字和结果名都不同。

顺便记两个细节：

- 结果列表只有一个且没命名时，可以省掉圆括号，比如 `func add(a, b int) int`；
- 函数类型的零值是 `nil`，可以直接和 `nil` 比较。

### 高阶函数

高阶函数满足下面任意一条：

1. **接受其他函数作为参数**；
2. **把其他函数作为结果返回**。

```go
type operate func(x, y int) int

func calculate(x, y int, op operate) (int, error) {
	if op == nil {
		return 0, errors.New("invalid operation")
	}
	return op(x, y), nil
}
```

`calculate` 就是高阶函数——它把函数当成参数收进来。实践中的典型例子是排序：`sort.Slice` 要你传一个"谁该排前面"的比较函数。

```go
nums := []int{5, 2, 4}
sort.Slice(nums, func(i, j int) bool {
	return nums[i] < nums[j]
})
```

另一种写法是**把函数作为返回值**：

```go
type calculateFunc func(x, y int) (int, error)

func genCalculator(op operate) calculateFunc {
	return func(x, y int) (int, error) {
		if op == nil {
			return 0, errors.New("invalid operation")
		}
		return op(x, y), nil
	}
}
```

`genCalculator` 不直接算，而是"生成一个能算的函数"返回给你，用起来就是这样：

```go
add := genCalculator(func(x, y int) int { return x + y })
fmt.Println(add(1, 2)) // 3
```

### 闭包

上面 `genCalculator` 返回的那个匿名函数，就是一个**闭包函数**。它内部用到的 `op` 既不是它的参数、也不是它自己声明的变量，而是外层 `genCalculator` 的参数，这种变量叫**自由变量**。

`op` 到底是什么，在写这个匿名函数的时候是不知道的，只有 `genCalculator` 被调用、`op` 被赋值的那一刻才确定下来——这时函数的状态从"不确定"变成"确定"，也就真正**闭合**成了一个闭包。

最小的例子是一个计数器：

```go
func counter() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}

func main() {
	next := counter()
	fmt.Println(next()) // 1
	fmt.Println(next()) // 2

	other := counter() // 新的闭包，有自己的 i
	fmt.Println(other()) // 1
}
```

> `i` 是"逃逸"到堆上的局部变量。每次调用 `counter()` 都会产生一个**新的、独立的** `i`，所以 `next` 和 `other` 互不影响。

**闭包到底有什么用？** 说"延迟执行"太虚了，实际价值是：它让我们能在运行期间**动态地生成逻辑**，并且**把状态藏起来**。几个真实场景：

**1. 函数工厂**

```go
func CalculationFactory(operation string) func(int, int) int {
	switch operation {
	case "add":
		return func(a, b int) int { return a + b }
	case "subtract":
		return func(a, b int) int { return a - b }
	case "multiply":
		return func(a, b int) int { return a * b }
	case "divide":
		return func(a, b int) int {
			if b != 0 {
				return a / b
			}
			return 0
		}
	default:
		return nil
	}
}
```

传 `"add"` 拿加法，传 `"divide"` 拿除法——同一个函数，根据不同配置产出不同的能力。这就是本节课作业里计算器的核心思路。

**2. 状态共享（迭代器）**

```go
func intsIterator(nums []int) func() (int, bool) {
	i := 0
	return func() (int, bool) {
		if i >= len(nums) {
			return 0, false
		}
		v := nums[i]
		i++
		return v, true
	}
}
```

**3. 回调函数**

```go
func GetData(data int, callback func(int)) {
	go func() {
		result := data + 2
		callback(result)
	}()
}
```

**4. 中间件**（现在看不懂没关系，Web 课会再见）

```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		m := validPath.FindStringSubmatch(r.URL.Path)
		if m == nil {
			http.NotFound(w, r)
			return
		}
		fn(w, r, m[2]) // 校验通过才继续执行真正的处理函数
	}
}
```

> 闭包很抽象，现在能理解"函数可以记住外层的变量"就够了，剩下的等写 Web 中间件的时候回来再看。

### 值类型与引用类型

最后回到函数本身。看一个例子：

```go
package main

import "fmt"

func main() {
	array1 := [3]string{"a", "b", "c"}
	fmt.Printf("The array: %v\n", array1)
	array2 := modifyArray(array1)
	fmt.Printf("The modified array: %v\n", array2)
	fmt.Printf("The original array: %v\n", array1)
}

func modifyArray(a [3]string) [3]string {
	a[1] = "x"
	return a
}
```

> 大家猜猜，`array1` 会变吗？

答案是**不会**。所有传给函数的参数值都会被复制，函数内部操作的只是副本。数组是值类型，复制时会连同所有元素一起拷贝，所以函数里改的只是副本。

但换个类型就不一样了：

```go
func modifySlice(s []int) {
	s[0] = 100 // 会影响到调用方
}

func main() {
	s := []int{1, 2, 3}
	modifySlice(s)
	fmt.Println(s) // [100 2 3]
}
```

切片、map、channel 这类"引用语义"的类型，复制时只拷贝它们本身（对切片来说就是那个小结构体），**底层数据是共享的**——这叫浅表复制，不是深拷贝。

| | 值类型 | 引用语义 |
| -- | -- | -- |
| 典型代表 | 数组、结构体 | 切片、map、channel |
| 赋值/传参 | 整体拷贝一份 | 拷贝头部结构，共享底层数据 |
| 函数内修改 | 不影响调用方 | 会影响调用方 |

> 想深拷贝切片，就得自己 `make` + `copy`，或者用 `maps.Clone` / `slices.Clone`（Go 1.21+ 标准库提供）。

***

## 五、go test

单元测试，又称"程序员测试"——顾名思义，这是程序员自己该做的检查工作。

Go 从诞生起就非常重视测试，标准库直接提供了 `testing` 包和 `go test` 命令，不需要装任何第三方框架。

Go 支持三类测试：

- **功能测试（test）**：验证函数行为符不符合预期
- **基准测试（benchmark）**：也就是性能测试，测函数跑得多快
- **示例测试（example）**：既是文档，也能当功能测试跑

### 命名与书写规则

- 测试源码文件和被测文件**放在同一个包里**；
- 文件名以被测文件名为前导，以 `_test.go` 结尾。比如 `calc.go` 对应的测试文件叫 `calc_test.go`；
- 每个测试文件至少包含一个测试函数；
- 三类测试函数的前缀和签名是**硬性规定**：

| 类型 | 名称前缀 | 参数 |
| -- | -- | -- |
| 功能测试 | `Test` | `*testing.T`（只能有一个） |
| 基准测试 | `Benchmark` | `*testing.B` |
| 示例测试 | `Example` | 无强制要求 |

### 第一个测试

被测代码 `calc.go`：

```go
package calc

func Add(a, b int) int {
	return a + b
}
```

测试代码 `calc_test.go`：

```go
package calc

import "testing"

func TestAdd(t *testing.T) {
	got := Add(1, 2)
	if got != 3 {
		// 注意：Go 标准库没有 assert，判断失败要自己用 t.Errorf / t.Fatalf
		t.Errorf("Add(1, 2) = %d, want 3", got)
	}
}
```

命令行：

```bash
go test              # 跑当前包的测试
go test -v           # 显示每个用例的执行过程
go test ./...        # 跑当前模块下所有包的测试
go test -run TestAdd # 只跑名字匹配的用例（支持正则）
go test -cover       # 看测试覆盖率
go test -bench=.     # 跑基准测试
```

### 表驱动测试

Go 里最推荐的写法是**表驱动**：把输入和期望输出列成一张表，循环跑一遍。

```go
func TestAdd(t *testing.T) {
	cases := []struct {
		name string
		a, b int
		want int
	}{
		{"正数", 1, 2, 3},
		{"含负数", -1, 1, 0},
		{"零", 0, 0, 0},
	}

	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			if got := Add(c.a, c.b); got != c.want {
				t.Errorf("Add(%d, %d) = %d, want %d", c.a, c.b, got, c.want)
			}
		})
	}
}
```

`t.Run` 会给每个子用例起名字，跑失败时一眼就能看出是哪个 case 挂了。

### 基准测试

```go
func BenchmarkAdd(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Add(1, 2)
	}
}
```

`b.N` 由 `go test` 自动调整：它会不断加大循环次数，直到测出比较稳定的耗时，最后报告每次操作耗时（ns/op）。

### 示例测试

```go
func ExampleAdd() {
	fmt.Println(Add(1, 2))
	// Output: 3
}
```

`// Output:` 注释里写期望输出，`go test` 会真的执行并比对。它同时也是能直接展示在文档里的例子。

> 思考：`testing.T` 和 `testing.B` 之外，`testing` 包里还有别的类型和方法吗？它们分别做什么用？（提示：`t.Helper`、`t.Cleanup`、`b.ResetTimer`、`t.Run` 的返回值……）

***

## 六、Git 基本使用

上节课我们已经知道 git 是用来做版本控制和团队协作的。这节课要用 git 交作业，这里就补一下最常用的命令，跟着走一遍就会了。

### 首次使用：配置身份

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

这两行只需要执行一次，之后所有仓库都会用这个身份记录"这次提交是谁写的"。

### 三个区域

git 把文件的变化分成三步走：**工作区 → 暂存区 → 版本库**。

- 工作区：你正在编辑的文件夹
- 暂存区：决定"这次到底要提交哪些改动"
- 版本库：真正存下来的历史记录

`git add` 负责把工作区的改动放进暂存区，`git commit` 再把暂存区的内容记进版本库——两步分开，就是为了让你能挑着提交。

### 常用命令

| 命令 | 作用 |
| -- | -- |
| `git init` | 把当前目录初始化成一个 git 仓库 |
| `git clone <url>` | 把远程仓库克隆到本地 |
| `git status` | 看当前有哪些改动（最常用，随时敲） |
| `git add .` | 把当前目录下的所有改动加入暂存区 |
| `git add <文件>` | 只暂存指定文件 |
| `git commit -m "说明"` | 把暂存区的内容提交成一个版本 |
| `git log --oneline` | 查看提交历史 |
| `git remote add origin <url>` | 给本地仓库关联一个远程仓库 |
| `git push` | 把本地提交推送到远程 |
| `git pull` | 把远程的最新提交拉取到本地 |
| `git checkout -b <分支名>` | 新建并切换到一个分支 |

> `git add .` 会把**当前目录下所有**改动都加进来，方便但也容易误伤：`.idea/` 这类 IDE 配置、编译产物、密钥文件都可能被顺手提交。所以敲之前先 `git status` 看一眼，或者在仓库根目录放一个 `.gitignore`，把不想提交的路径写进去。

### 一个完整的流程

假设你已经按作业要求在 GitHub 上建好了一个空仓库，比如 `Lanshan-Go-2026-Homework`：

```bash
# 1. 克隆到本地（本地已经有代码的话，用 git init 初始化，再 git remote add origin 关联远程）
git clone https://github.com/你的用户名/Lanshan-Go-2026-Homework.git
cd Lanshan-Go-2026-Homework

# 2. 写完作业，先看看改了哪些文件
git status

# 3. 把改动放进暂存区
git add .

# 4. 提交一个版本，-m 后面写这次改动做了什么
git commit -m "feat: 完成第一节课作业"

# 5. 推送到远程
git push
```

推送成功后刷新 GitHub 页面，代码就出现在仓库里了。

> 第一次推送一个新分支时可能要带上 `-u`：`git push -u origin main`，之后直接 `git push` 就行。
> 多人协作时，push 之前先 `git pull` 拉一下别人的最新改动，能省掉不少冲突。

***

# 课后作业

### LV0

把本节课讲过的代码都敲一遍，特别是**切片共享底层数组**那段：在实际运行前，先自己想出每个切片的 `len`、`cap` 和内容，再跑起来对答案。

### LV1

编写一个函数，接收一个整数切片，返回一个 `map`，键是切片中的元素，值是该元素出现的次数。

### LV2

实现一个简易计算器，完成两个数的四则运算，要求**必须用到高阶函数或闭包**

### LV3

给你这节课写的函数补一个 `_test.go` 测试文件，至少包含：

- 一个功能测试；
- 一个表驱动测试（覆盖正常值和边界值，比如除数为 0）。

### LVX

进阶（选做）：

1. 用 `go test -bench=.` 给 `Add` 写一个基准测试，看看它的 ns/op 是多少；
2. 实现一个"通用去重"函数，用闭包维护已出现过的元素集合；
3. 用 `slices.Clone` 或 `make` + `copy` 实现对切片的深拷贝，并写测试验证修改副本不会影响原切片。

### 思考题

- 如果有多个切片指向同一个底层数组，你觉得应该注意什么？
- 怎样沿用"扩容"的思想对切片进行"缩容"？
- 了解 map 底层的哈希逻辑（哈希函数、哈希和、哈希碰撞）。
- 函数拿到的参数值只是副本，那么函数返回给调用方的结果值也会被复制吗？
- 了解一下 `container` 包里的其他容器，比如 `List` 和 `Element`。

给大家布置这么多思考题，是怕学习进度快的同学没事做，给大家一个进阶学习的方向。**0 基础的同学下来能把这节课上讲的内容消化好就很不错了！！**

***

## 作业提交

在 GitHub 上新建一个仓库，命名为 `Lanshan-Go-2026-Homework`，把作业代码上传到该仓库中，并把仓库地址发送到 `wanghaihang@lanshan.email`。

建议用 git 把代码提交到仓库（命令行用法见本节课的「六、Git 基本使用」）。

有问题在飞书群里提问或者飞书私聊均可。