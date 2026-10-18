# 一个go语言基础oop
go语言中和传统语言的面向对象的实现方式有一定的差异。go语言不支持类和继承，没有private、public关键字，取而代之的使用是组合实现多态，根据首字母大小写实现导出规则。  
> ps:这个文档是我在备课之前写的，可能会有一些重复的地方
## 对象与方法
### 类
#### 结构体（类）的声明
如果你有一点点的go语言基础，你就会知道我们可以通过**type**关键字定义新的类型，比如`struct{}` `interface{}`  
比如
```go
type Animal struct{
	Type string
	Name string
	Age int16
}
```
```go
type Speaker interface{
	Speak()
}
```
在go语言中，没有**类**单独的关键字。事实上，对于go来说，类就是结构体。比如上面我们就声明了一个Animal的结构体（类）。
### 方法
#### 声明
与传统语言oop中，方法在类里面定义不同的是，go语言中方法是在结构体外部。比如说：
```go
type Dog struct{
	Type string
	Name string
	Age int16
}

func (self Dog)Speak(){
	fmt.Println("汪！")
}

```
在go语言中，方法是一种作用于特定类型变量的函数。这种特定类型变量叫做**接收者**。  
就比如上面的例子中，(self Dog)中的self为接收者的变量名，这里为了方便理解命名为"self",实际上你可以为他取任何的合法变量名称。Dog指的是接收者的类型。  
#### 使用 
就以上面的例子来说：
```go
func main(){
	dog:=NewDog()
	dog.Speak()//打印"汪！"
}
```
### 创建类的实例
细心的你可能想问了，既然结构体中不能包含方法，那我们如何构建一个类（结构体）呢  
一般为两种形式：
```go
//第一种，直接字面量创建
dog:=Dog{} //值类型
dog:=&Dog{} //指针类型
```
```go
//第二种，函数工厂
//一般在结构体声明之后会在后面配套一个函数
func NewDog()Dog{
	return Dog{}
}//这里面可以设置初始值，或者添加这个函数的传入参数用来设置初始值
func NewDog()*Dog{
	return &Dog{}
}//同上，但是这是指针类型的对象
dog:=NewDog()//调用函数工厂创建
```
细心的你发现，我特地将值和指针类型分开说明了，这两者在使用上是有一定区别的，我放在后面讲。
## 接口
### 声明与实现
上文中我提到了`interface{}`接口，这是go语言中实现多态的核心机制。一个接口中会写0个或者多个方法，实现全部方法的结构体均可以称为满足了这个接口。  
go语言中，接口是隐式实现的，不需要在对象声明的时候显示写出满足的接口，只要能够满足该接口所需的所有方法。  
我们先创建一个接口：
```go
type Speaker interface{
	Speak()
}
```
我们可以很轻易地发现，前文中结构体Dog的Speak()方法和Speaker接口所需要的Speak()方法有着一样的签名和方法名，所以我们可以称Dog类实现了Speaker接口。
### 用法
```go
type Dog struct{}
func (d Dog)Speak(){
	fmt.Println("汪！")
}

type Cat struct{
	Name string
}
func (_ Cat)Speak(){
	fmt.Println("喵！")
}//这里接收者的变量名为'_'表示后续不会使用，当然和前面一样写成(c Cat)也是可以的
func (c Cat)CallName(){
	fmt.Println(c.Name)
}

type Speaker interface{
	Speak()
}

func Call(speaker Speaker){
	speaker.Speak()
}

func main(){
    dog:=Dog{}
	cat:=Cat{}
	
    Call(dog)//打印"汪！“
    Call(cat)//打印"喵！"
	
	//同理
	var speaker Speaker
	speaker.Speak()//触发panic进程终止，因为未创建实例，speaker为默认空值nil
	
	speaker=Dog{}
    speaker.Speak()//打印"汪！“
	speaker=Cat{}
    speaker.Speak()//打印"喵！"
	
	//但是
	speaker.CallName()//报错，接口类型变量只能执行接口约束内的行为
}
```
### 类型断言
我前面说到，接口可以包含0个方法声明，也就是`interface{}`空接口，它还有个别名是`any`。在泛型出来之前，为了处理多类型相同逻辑的情况，基本都是使用空接口作为参数类型。  
```go
func Name(input any){
	
}
```
这样会隐去类型内部的性质，比如我确定这个函数只能处理整数类型的参数，那我可以通过类型断言的方式：
```go
func Name(input any){
	inputInt:=input.(int)
}
```
但是，如果传入的参数的类型不为int(int64都不行，type MyInt int的MyInt也不行)，程序就会触发panic，所以类型断言和range一样其实可以有两个返回参数，只是第二个参数不用的时候可以不写出来：
```go
func Name(input any){
	inputInt,ok:=input.(int)
	if !ok{
		return
    }
}
```
---
## 进阶
### 继承（组合）
#### 基本用法
```go
type Animal struct{
	Type string
	Name string
	Age int16
}
func (a Animal)ShowName()string{
	return a.Name
}

type Dog struct{
	Animal//直接继承（结构体的匿名嵌入）
}
type Cat struct{
	A Animal//间接继承
}
func main(){
	dog:=Dog{
		Animal:Animal{
			Name:"dog1",
		},//直接继承的默认字段名为该结构体名字，如果是结构体叫animal那字段就是animal
    }
	//dog:=Dog{
	//  Name:"dog1",
	//}//go1.27加入的语法糖，可以直接提升字段
	dog.ShowName()//输出”dog1”
	dog.Animal.ShowName()//输出“dog1”
    cat:=Cat{
        A:Animal{
            Name:"cat1",
		},
	}
	cat.ShowName()//报错
	cat.A.ShowName()//输出“cat1”
}
```
上面这个demo说明在go语言中，匿名嵌入会直接继承嵌入的结构体的所有方法，显式导入就只能通过访问成员变量的方式进行调用。  
这时候你可能会问了，如果匿名嵌入的多个结构体存在同名方法怎么办？很简单，直接退化为成员变量访问即可：
```go
type Cat struct{}
func (_ Cat)Speak(){
	fmt.Println("喵！")
}

type Dog struct{}
func (_ Dog)Speak(){
	fmt.Println("汪！")
}

type Home struct{
	Cat
	Dog
}

func main(){
	home:=Home{}
	home.Speak()//报错，编译器不知道调用哪一个成员方法
	home.Cat.Speak()//输出“喵！”
	home.Dog.Speak()//输出“汪！”
}
```
除此之外，接口也可以实现继承：
```go
type Writer interface{
	Write([]byte)(int,error)
}
type Reader interface{
	Read([]byte)(int,error)
}

type WriterAndReader interface{
	Writer
	Reader
}
//等价于
type WriterAndReader interface{
	Write([]byte)(int,error)
	Read([]byte)(int,error)
}
```
#### 只继承内存结构，不继承方法
我们知道**type**可以定义新的类型，我们只需要`type NewType OldType`，这样NewType就不会继承OldType的所有方法，但是底层的内存结构是相同的，不过由于是不同的类型，转化时需要显式进行
### 接收者的类型
前面提到，接收者为值类型或者指针类型存在区别，其中最简单的区别在于：指针类型可以直接修改对象的成员变量，而值类型本质上是复制副本，即使修改了也仅限这一次方法内部，这和函数同一个道理。  
不过go语言中，官方文档建议一个对象的所有方法的接收者最好均为值类型或者指针类型，当然如果你非要混合这使用也是可以的。
此外，还有一个有意思的地方在于，接收者如果是值类型，那么无论对象实例是值还是指针，都可以满足实现的接口。但是如果接收者是指针类型，那么实例也必须是指针，直接转化为接口类型会报错：
```go
type Animal struct {
	Type string
	Name string
	Age  int16
}
func (a Animal) ShowName() string {
	return a.Name
}

type Inter interface {
	ShowName() string
}


func main() {
	var A Inter = Animal{
		Name: "Dog",
	}
	var B Inter = &Animal{
        Name: "Cat",
    }//能正常运行
}
```
但是我们将接收者类型改为指针：
```go
type Animal struct {
	Type string
	Name string
	Age  int16
}
func (a *Animal) ShowName() string {
	return a.Name
}

type Inter interface {
	ShowName() string
}


func main() {
	var A Inter = Animal{
		Name: "Dog",
	}//这里会报错，提示类型不匹配
	var B Inter = &Animal{
        Name: "Cat",
    }
}
```
### 对象plus
前面提到过，方法一种作用于特定类型变量的函数。这里的特殊类型不限于结构体，只要是type的新类型都可以拥有自己的方法和实现的接口。  
比如：
```go
type httpMessage string
func (h httpMessage)String()string{
	return string(h)
}
func (h httpMessage)Print(){
	fmt.Print(h)
}
```
用法和细节上和前面提到的结构体基本没有差异，这里主要是强调一下go中的对象不局限于结构体  
### 一些八股
> 我这会也不是记得很完全，~~可能~~ ~~多半~~ 大概率会漏掉很多东西，就当是一个小补充吧，这一块可能会有点乱，大致看看吧，了解一下就行了
#### 接口
接口的底层实现是一个包含了类型信息和指针的结构体
```go
type iface struct {
    tab  *itab        // 类型信息 + 方法表
    data unsafe.Pointer // 指向具体值的指针
}
```
可以看到接口只能包含一个指针，那对于满足接口的值和指针就会有不同的待遇：
##### 指针
指针很简单，只需要让iface.data指向传入的数据并补全类型信息即可
##### 值
在go中，变量的值是放在函数的栈空间上的，而GC的负责范围为堆。当你尝试将一个值变量赋值/转换为接口时，会发生以下几步：
1. 在堆上分配一个内存
2. 将这个值拷贝进去
3. 将堆上的地址指针赋值给iface.data
可以看见只要使用了接口，不论你传的是值还是指针，最终都还是逃不过逃逸到堆上的命运(当然go的编译器还是会有一定的优化，尽可能的放在栈上)，甚至值类型还涉及到了一次值拷贝
##### 坑
新手有时候会踩到的一个很经典的坑。  
go语言中结构体是可以做`==`的，原理就是逐个比较结构体内部的字段（前提是这个结构体的每一个字段都是可以判断，像map就不行）。所以接口字段也是一样的，接口判断相等时会比较类型信息和实际数据。  
这样说可能有点抽象，来个例子：
```go
//前面提到过，any是interface{}的别名，本质上就是个接口
var a any=(*int)(nil)
var b any=nil
fmt.Println(a==nil)//输出false
fmt.Println(b==nil)//输出true
```
这个例子中a的类型信息为*int,而不是nil，所以a和nil不相等，所以在函数中将传入参数赋值给接口之前就应该判断非nil而不是赋值之后，不然就可能出现if a==nil 不通过，但是后续调用的时候出现空指针panic
