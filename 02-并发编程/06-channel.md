## 一 并发模型与channel 

单纯的将函数并发执行是没有意义的，函数与函数之间必须能够交换数据才能体现并发执行函数的意义。为了实现数据的通信，有两种常见并发模型：
- 共享数据：一般使用共享内存方式来共享数据，Go中的实现方式为互斥锁（sync包）。
- 消息：消息机制认为每个并发单元都是独立个体，拥有自己的变量。不同的并发单元之间不共享各自的变量，只通过消息来进行数据输入输出，Go中的实现方式为channle。

在Go中对上述两种方式都进行了实现，但是Go不推荐共享数据方式，推荐channel的方式进行协程通信。因为多个 goroutine 为了争抢数据，容易发生竞态问题，会造成执行的低效率，使用队列的方式是最高效的， channel 就是一种队列一样的结构。  

如图所示：
![](../images/go/02-04.svg)
 
channel特性：
- channel的本质是一个数据结构-队列，先进先出
- channel是线程安全的，多goroutine访问时，不需要加锁，因为在任何时候，同时只能有一个goroutine访问通道。
- channel拥有类型，一个string的channle只能存放string类型数据

## 二 channel的使用

#### 2.1 channel的创建

```go
package main

import "fmt"

func main(){

    var ch chan int             					// 声明一个channel，但未初始化，值为nil
    ch = make(chan int)         					// 初始化，此时ch有了地址

    // 协程中向通道填充数据
    go func() {
        ch <- 100
        fmt.Printf("入队后数据：%v\n",ch)			// 输出内存地址
        ch <- 200
        fmt.Printf("入队后数据：%v\n",ch)			// 输出内存地址
        ch <- 300
        fmt.Printf("入队后数据：%v\n",ch)			// 没有输出
    }()

    // 主协程中取出数据
    data1 := <-ch
    data2 := <-ch
    fmt.Printf("取出的数据data1：%v\n", data1)		// 100
    fmt.Printf("取出的数据data2：%v\n", data2)		// 200

}

```

注意：channel内可以存储多种数据类型，如下所示：
```go
//初始化channel
ci := make(chan int)
cs := make(chan string)
cf := make(chan interface{})				
```

#### 2.2 channel阻塞式接收数据

channel直接接收数据方式是阻塞式的，如下所示：
```go

data, ok := <-ch

// ok 可以省略
data := <-ch

// 数据也可以省略，表示直接出队一个数据，不使用该数据
<- ch
```

#### 2.3 通道数据的遍历

channel只支持 for--range 的方式进行遍历：
```go
	ch := make(chan int)

	go func() {
		for i := 0; i <=3; i++ {
			ch <- i
			time.Sleep(time.Second)
		}
	}()

	for data := range ch {
		fmt.Println("data==", data)
		if data == 3 {
			break
		}
	}
```

## 三 通道缓冲

#### 3.1 无缓冲通道

上述案例中声明的通道都是无缓冲的。  

无缓冲的通道： 通道的收发操作必须在不同的两个`goroutine`间进行，因为：通道的数据在没有接收方处理时，数据发送方会持续阻塞，所以通道的接收必定在另外一个 goroutine 中进行。  

这也是 2.2 章节中的channel特性，利用该阻塞接收特性可以实现channel队列的并发同步效果：
```go
	ch := make(chan int)		// 没有声明长度却可以装入数据，下面章节介绍

	go func() {
		fmt.Println("start goroutine..")
		ch <- 0
		fmt.Println("exit goroutine..")
	}()

	fmt.Println("wait goroutine..")

	<-ch

	fmt.Println("all done..")
```

输出顺序：exit 一直在 all 之前
```
wait goroutine..
start goroutine..
exit goroutine..
all done..
```

注意：非阻塞的通道接收方法可能造成高的 CPU 占用，因此使用非常少。如果需要实现接收超时检测， 可以配合 select 和计时器 channel 进行。

#### 3.2 有缓冲通道

在无缓冲通道的基础上，为通道增加一个有限大小的存储空间形成带缓冲通道：
```go
	ch := make(chan int, 3)					// 3就是缓冲长度
	fmt.Println("缓冲长度：", len(ch))		// 0
	ch <- 1
	ch <- 2
	fmt.Println("缓冲长度：", len(ch))		// 2
```

贴士：在上述案例中，因为使用了缓冲通道。即便没有 goroutine 接收，发送者也不会发生阻塞。

注意区分通道的容量与长度：
- 容量：cap(ch)，即通道声明时设置的容量大小。
- 长度：len(ch)，即通道中实际存储的数据多少。

带缓冲通道发生阻塞条件：
- 带缓冲通道被填满时，尝试再次发送数据会发生阻塞
- 带缓冲通道为空时，尝试接收数据会发生阻塞

问题：为什么 Go语言对通道要限制长度而不提供无限长度的通道?  
> channel是在两个 goroutine 间通信的桥梁。使用 goroutine 的代码必然有一方提供数据，一方消费数据 。通道如果不限制长度，在生产速度大于消费速度时，内存将不断膨胀直到应用崩溃。

## 四 通道读写

默认情况下管道的读写是双向的，因为不能让 goroutine 锁死，channel 必须是双向的。  

但是为了对 channel 进行使用的限制，可以将 channel 声明为只读或只写。  

```go
	var chan1 chan<- int		// 声明 只写channel
	var chan2 <-chan int 		// 声明 只读channel
```

单向channel的转换：
```Go
ch3 := make(chan int)			// 声明普通channel
ch4 := <-chan int(ch4)			// 转换为 只读channel
ch5 := chan<- int(ch4)			// 转换为 只写channel
```

## 五 通道关闭

通道是一个引用对象，支持GC回收，但是通道也可以主动被关闭：

```go
	ch := make(chan int)
	close(ch)				// 关闭通道
	ch <- 1					// 报错：send on closed channel
```

给被关闭的通道发送数据会造成panic，不过已经关闭的通道在接收数据时不会发生阻塞，此时将会接收到通道类型的零值，然后停止阻塞并返回。

通道在遍历时候关闭，需要注意两个细节：
- 在遍历时，如果 channel 没有关闭，则会出现 deadlock 的错误
- 在遍历时，如果 channel 已经关闭，则会正常遍历数据，遍历完后，就会退出遍历。
```go
	ch := make(chan int, 10)
	for i := 0; i < 10; i++ {
		ch <- i
	}

	//关闭:关闭后不能再向 channel 写数据，但可以从该 channel 读取数据。
	close(ch)			

	for v := range ch {
		fmt.Println("v=", v)
	}
```


如何判断管道被关闭：
```go
data, ok := <-ch
if ok == false {

}

```

## 六 channel的一些示例

#### 6.1 示例一 限制并发

耗时操作为timeMore，现在有100个并发，限制为5个：
```go
package main

import (
	"time"
	"fmt"
)

func timeMore(ch chan string) {
	
	// 执行前先注册，写不进去就会阻塞
	ch <- "任务"			

	fmt.Println("模拟耗时操作")
	time.Sleep(time.Second)	// 模拟耗时操作

	// 任务执行完毕，则管道中销毁一个任务
	<-ch

}

func main() {

	ch := make(chan string, 5)

	// 开启100个协程
	for i := 0; i < 100; i++ {
		go timeMore(ch)
	}

	for {
		time.Sleep(time.Second)
	}

}
```

#### 2.2 生产者消费者模型

方式一：无缓冲区
```go
package main
 
import (
	"fmt"
	"time"
)
 
func producer(ch chan<- int) {
	for i := 0; i < 5; i++ {
		ch <- i
		fmt.Println("Send:", i)
	}
}
 
func consumer(ch <-chan int) {
	for i := 0; i < 5; i++ {
		v := <-ch
		fmt.Println("Receive:", v)
	}
}

func main() {
	ch := make(chan int)
	go producer(ch)
	go consumer(ch)
	time.Sleep(3 * time.Second)
}
```

因为channel没有缓冲区，所以当生产者给channel赋值后，生产者线程会阻塞，直到消费者线程将数据从channel中取出。消费者第一次将数据取出后，进行下一次循环时，消费者的线程也会阻塞，因为生产者还没有将数据存入，这时程序会去执行生产者的线程。程序就这样在消费者和生产者两个线程间不断切换，直到循环结束。  

输出结果：
```
Receive: 0
Send: 0
Send: 1
Receive: 1
Receive: 2
Send: 2
Send: 3
Receive: 3
Receive: 4
Send: 4
```

方式二：带缓冲区，只需要修改channel创建代码为：`ch := make(chan int, 10)`即可。输出结果为：
```
Send: 0
Send: 1
Send: 2
Send: 3
Send: 4
Receive: 0
Receive: 1
Receive: 2
Receive: 3
Receive: 4
```
