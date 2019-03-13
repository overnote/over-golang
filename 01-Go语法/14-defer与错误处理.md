## 一 defer延迟执行

#### 1.1 defer延迟执行修饰符

在函数中，程序员经常需要创建资源(比如:数据库连接、文件句柄、锁等) ，为了在函数执行完 毕后，及时的释放资源，Go 的设计者提供 defer (延时机制)。
```go
func main() {

	//当执行到defer语句时，暂不执行，会将defer后的语句压入到独立的栈中
	//当函数执行完毕后，再从该栈按照先入后出的方式出栈执行
	defer fmt.Println("defer1...")
	defer fmt.Println("defer2...")

	fmt.Println("main...")

}
```
上述代码执行结果：
```
main...
defer2...
defer1...
```
从上述代码看出：Go语言的defer语句会将其后跟随的语句进行延迟处理，在defer归属的函数即将返回时，将延迟处理的语句按defer的逆顺序进行执行，即，先被defer的语句最后被执行。

注意：在 defer 将语句放入到栈时，也会将相关的值拷贝同时入栈
```go
func main() {

	num := 0

	defer fmt.Println("defer中：num=", num)

	num = 3
	fmt.Println("main中：num=",num)

}
```
输出结果：
```
main中：num= 3
defer中：num= 0
```

#### 1.2 defer最佳实践

defer最佳实践：用于关闭资源，比如：`defer connect.close()`。下面列举一个常见的并发使用map的函数：
```go
var (
	mutex sync.Mutex
	testMap = make(map[string]int)
)
func getMapValue(key string) int {

	mutex.Lock()						//对共享资源加锁
	value := testMap[key]
	mutex.Unlock()

	return value
}
```
上述案例是很常见的对并发map执行加锁执行的安全操作，使用defer可以对上述语义进行简化：
```go
var (
	mutex sync.Mutex
	testMap = make(map[string]int)
)
func getMapValue(key string) int {

	mutex.Lock()						//对共享资源加锁
	defer mutex.Unlock()
	return testMap[key]
}
```

defer处理资源案例：
```go

f,err := os.Open(file)

if err != nil {
	return 0
}

info,err := f.Stat()

if err != nil {
	f.Close()
	return 0
}

//后续一系列文件操作后执行关闭
f.Close()
return 0;

```
使用defer优化：
```go
f,err := os.Open(file)

if err != nil {
	return 0
}

defer f.Close()

info,err := f.Stat()

if err != nil {
	// f.Close()			//这句已经不需要了
	return 0
}

//后续一系列文件操作后执行关闭
// f.Close()			//这句已经不需要了
return 0;
```

## 二 错误Error

#### 2.1 Go自带的错误接口

error是go语言声明的接口类型：
```go
type error interface {
	Error() string
}
```
所有符合Error()string格式的方法，都能实现错误接口，Error()方法返回错误的具体描述。

#### 2.2 自定义错误

返回错误前，需要定义会产生哪些可能的错误，在Go中，使用errors包进行错误的定义，格式如下：
```go
var err = errors.New("发生了错误")
```
提示：错误字符串相对固定，一般在包作用于声明，应尽量减少在使用时直接使用errors.New返回。

下面这个例子演示了如何使用`errors.New`:
```go
func Sqrt(f float64) (float64, error) {
	if f < 0 {
		return 0, errors.New("math: square root of negative number")
	}
	// implementation
}
```

在C语言里面是通过返回-1或者NULL之类的信息来表示错误，但是对于使用者来说，不查看相应的API说明文档，根本搞不清楚这个返回值究竟代表什么意思，比如:返回0是成功，还是失败,而Go定义了一个叫做error的类型，来显式表达错误。在使用时，通过把返回的error变量与nil的比较，来判定操作是否成功。例如`os.Open`函数在打开文件失败时将返回一个不为nil的error变量

```Go
func Open(name string) (file *File, err error)
```
下面这个例子通过调用`os.Open`打开一个文件，如果出现错误，那么就会调用`log.Fatal`来输出错误信息：
```Go

f, err := os.Open("filename.ext")
if err != nil {
	log.Fatal(err)
}
```
类似于`os.Open`函数，标准包中所有可能出错的API都会返回一个error变量，以方便错误处理，这个小节将详细地介绍error类型的设计，和讨论开发Web应用中如何更好地处理error。

#### 2.3 自定义错误案例

案例一：简单的错误字符串提示
```go
package main

import (
	"errors"
	"fmt"
)

//定义除数为0的错误
var errByZero = errors.New("除数为0")

func div(num1, num2 int) (int, error) {

	if num2 == 0 {
		return 0, errByZero
	}

	return num1 / num2, nil

}

func main() {
	fmt.Println(div(1, 0))
}
```

案例二：实现错误接口
```go
package main

import (
	"fmt"
)

//声明一种解析错误
type ParseError struct {
	Filename string
	Line int
}

//实现error接口，返回错误描述
func (e *ParseError) Error() string {
	return fmt.Sprintf("%s:%d", e.Filename, e.Line)
}

//创建一些解析错误
func newParseError(filename string, line int) error {
	return &ParseError{filename, line}
}

func main() {

	var e error

	e = newParseError("main.go", 1)

	fmt.Println(e.Error())

	switch detail := e.(type) {
	case *ParseError:
		fmt.Printf("Filename: %s Line:%d \n", detail.Filename, detail.Line)
	default: 
		fmt.Println("other error")
	}

}
```

#### 2.4 errors包分析

Go中的erros包对New的定义非常简单:
```go

//创建错误对象
func New(text string) error {
	return &errorString{text}
}

//错误字符串
type errorString struct {
	s string
}

//返回发生何种错误
func (e *errorString) Error() string {
	return e.s
}
```
错误对象都要事先error接口的Error()方法，这样，所有的错误都可以获得字符串的描述。

## 三 panic 宕机

#### 2.1 手动触发宕机

Go语言可以在程序中手动触发宕机，让程序崩溃，这样开发者可以及时发现错误。  

Go语言程序在宕机时，会将堆栈和goroutine信息输出到控制台，所以宕机有额可以方便知晓发生错误的位置。如果在编译时加入的调试信息甚至连崩溃现场的变量值、运行状态都可以获取，那么如何触发宕机？  

```go
package main

func main() {

	panic("crash")

}
```

运行结果是：
```
panic: crash

goroutine 1 [running]:
main.main()
	/Users/username/Desktop/TestGo/src/main.go:5 +0x39
exit status 2
```

使用`panic`函数可以制造崩溃，panic声明如下；
```go
func panic(v interface{})
```
panic()参数可以是任意类型。

注意：手动触发宕机并不是一种偷懒的方式，反而能迅速报错，终止程序继续运行，防止更大的错误产生，但是如果任何错误都使用宕机处理，也不是一个良好的设计。

#### 3.2 defer与panic

当panic()发生宕机，panic()后面的代码将不会被执行，但是在panic前面已经运行过的defer语句依然会在宕机时发生作用：
```go
package main

import "fmt"

func main() {

	defer fmt.Println("before")
	panic("crash")

}
```

## 四 recover 宕机恢复

#### 4.1 让程序在崩溃时继续执行

无论是代码运行错误由Runtime层抛出的panic崩溃，还是主动触发的panic崩溃，都可以配合defer和recover实现错误捕捉和处理，让代码在发生崩溃后允许继续执行。  

在其他语言里，宕机往往以异常的形式存在，底层抛出异常，上层逻辑通过try/catch机制捕获异常，没有被捕获的严重异常会导致宕机，捕获的异常可以被忽略，让代码继续执行。Go没有异常系统，使用panic触发宕机类似于其他语言的抛出异常，recover的宕机恢复机制就对应try/catch机制。  

#### 4.2 使用defer与recover处理错误

```go
func test(num1 int, num2 int){
	defer func(){
		err := recover()	//recover内置函数，可以捕获异常
		if err != nil {
			fmt.Println("err=", err);
		}
	}()
	fmt.Println(num1/num2)
}

func main() {

	test(2,0)

}
```

#### 4.3 panic recover综合示例
```go

package main

import (
	"fmt"
	"runtime"
)

//崩溃时需要传递的上下文信息
type panicContext struct {
	function string
}

//保护方式允许一个函数
func ProtectRun(entry func()) {

	defer func() {
		err := recover()	//发生宕机时，获取panic传递的上下文并打印
		switch err.(type) {
		case runtime.Error:
			fmt.Println("runtime error:", err)
		default:
			fmt.Println("error:", err)
		}
	}()
	
	entry()

}

func main() {

	fmt.Println("运行前")

	ProtectRun(func(){

		fmt.Println("手动宕机前")

		panic(&panicContext{"手动触发panic",})

		fmt.Println("手动宕机后")

	})

	ProtectRun(func(){

		fmt.Println("赋值宕机前")

		var a *int
		*a = 1

		fmt.Println("赋值宕机后")

	})

	fmt.Println("运行后")

}
```

运行结果：
```
运行前
手动宕机前
error: &{手动触发panic}
赋值宕机前
runtime error: runtime error: invalid memory address or nil pointer dereference
运行后
```

#### 4.4 panic和recover关系

panic和defer的组合：
- 有panic没有recover，程序宕机
- 有panic也有recover，程序不会宕机，执行完对应的defer后，从宕机点退出当前函数后继续执行