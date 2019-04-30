## 一 select

#### 1.2 select的引入

在使用通道时，想同时接收多个通道的数据是一件困难的事情。通道在接收数据时-， 如果没有数据可以接收将会发生阻塞。虽然可以使用如下模式进行遍历，但运行性能会非常差：
```go
for {
	//尝试接收 chl 通道
	data , ok := <-chl //尝试接收 ch2 通道 
	data, ok := <-ch2 //接收后续通道
	...
}
```

Go语言中提供了 select关键字，可以同时响应多个通道的操作。 select的每个 case都 会对应一个通道的收发过程。当收发完成时 ，就会触发 case 中响应的语句。多个操作在每次 select 中挑选一个进行响应:
```go
select {
	case 操作1:
		响应操作1
	case 操作2:
		响应操作2
	...
	default:
		没有操作的情况
}
```

#### 1.1 select示例

select用于监听channel上的数据流动，在有多个channel时，肯定不能让其串行执行：

```go
package main

import (
	"fmt"
	"time"
)

func fn1(ch chan string) {
	time.Sleep(time.Second * 3)
	ch <- "fn1111"
}

func fn2(ch chan string) {
	time.Sleep(time.Second * 6)
	ch <- "fn2222"
}


func main() {

	ch1 := make(chan string)
	ch2 := make(chan string)

	go fn1(ch1)
	go fn2(ch2)

	r1 := <- ch1
	fmt.Println("r1=", r1)
	r2 := <- ch2
	fmt.Println("r2=", r2)
}	

```

如上所示，获取r1和r2的操作在串行执行，main函数整体耗时时间是fn2执行的时间：6秒。那么如何才能让获取ch1直接3秒钟内完成，获取ch2在6秒内完成？  

```go
package main

import (
	"fmt"
	"time"
)

func fn1(ch chan string) {
	time.Sleep(time.Second * 6)
	ch <- "fn1111"
}

func fn2(ch chan string) {
	time.Sleep(time.Second * 6)
	ch <- "fn2222"
}


func main() {

	ch1 := make(chan string)
	ch2 := make(chan string)

	go fn1(ch1)
	go fn2(ch2)


	select {

	case r1 := <- ch1 :
		fmt.Println("r1:", r1)

	case r2 := <- ch2 :
		fmt.Println("r2:", r2)
	}

}	
```

select同时监听多个channel，如果某个channel可读（有数据）则执行。如果多个channel同时可读（即上述fn1和fn2的延时时间都是3秒）那么随机读取一个channel的数据。  

select支持default，如果所有的case分支内都没有可读数据，那么执行default。  

当然，在一些场景中（for循环使用select获取channel数据），如果channel被写满，也可能会执行default。

注意：select中的case必须是IO操作。

#### 1.2 select的作用

在一个select语句中，Go会按顺序从头到尾评估每一个发送和接收的语句。如果其中任意一个语句可以继续执行，即没有被阻塞，那么久从那些可以被之心个的语句中任意选择一条来使用。

如果没有一条语句可以执行，即所有的通道都被阻塞，那么有两种可能的情况：
- 如果给出了default语句，执行default语句，同时程序性的执行会从select语句后的语句中恢复
- 如果没有default语句，那么select语句将被阻塞，直到至少有一个通信可以进行下去
- 所以一般不写default语句，
```Go
func fibonacci(ch chan<- int, quit<-chan bool) {
	x, y := 1, 1
	for {			//监听channel数据的流动
		select {
			case ch <- x:
				x, y = y, x+y
			case flag := <-quit:
				fmt.Println("flag=", flag)
			return
		}
	}
}

func main(){

	ch := make(chan  int)		//数字通信
	quit := make(chan bool)		//程序是否结束

	//消费者，从channel读取内容
	go func() {
		for i := 0; i < 8; i++ {
			num := <-ch
			fmt.Println(num)
		}
		quit <- true
	}()

	//生产者，生产数字，写入channel
	fibonacci(ch, quit)

}
```

#### 1.3 channel超时解决

在并发编程的通信过程中，最需要处理的是超时问题，即向channel写数据时发现channel已满，或者取数据时发现channel为空，如果不正确处理这些情况，会导致goroutine锁死，例如：
i := <-cha
如果永远没有人写入ch数据，那么上述代码一直无法获得数据，导致goroutine一直阻塞。利用select()可以实现超时处理：
```Go
timeout := make(chan bool, 1)

go func() {
    time.Sleep(1e9)	//等待1秒钟
    timeout <- true
}()

select {
    case <-ch:      //能取到数据
    case <-timeout: //没有从-cha中取到数据，此时能从timeout中取得数据
}
```

#### 1.4 空select

```go
select {}
```
空的select唯一的功能是阻塞代码。

## 二 select的一些案例

#### 2.1 案例一 模拟web开发
```go
package main

import (
	"fmt"
)

/**
	模拟远程调用RPC：使用通道代替 Socket 实现 RPC 的过程。
	客户端与服务器运行在同 一个进程， 服务器和客户端在两个 goroutine 中运行。
 */

// 模拟客户端
func RPCClient(ch chan string, req string) (string, error) {
	ch <- req							// 向服务器发送请求模拟：请求数据放入通道
	select {							// 等待服务器返回模拟：使用select
	case data := <-ch:
		return data, nil
	}
}

// 模拟服务端
func RPCServer(ch chan string) {
	// 通过无限循环继续处理下一个客户端请求。
	for {
		data := <-ch
		fmt.Println("server received: ", data)
		ch <- "roger"						// 向客户端反馈
	}
}

func main() {

	// 模拟 启动服务器
	ch := make(chan  string)
	go RPCServer(ch)

	// 模拟 发送数据
	receive, err := RPCClient(ch, "hi")
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Println("client receive: ", receive)
	}
}
```

#### 2.2 案例二 使用通道晌应计时器的事件

Go语言中的 time包提供了计时器的封装。由于 Go语言中的通道和 goroutine的设计， 定时任务可以在 goroutine 中通过同步的方式完成，也可以通过在 goroutine 中异步回调完 成。这里将分两种用法进行例子 展示。  

实现一段时间之后：
```go
	exitCh := make(chan int)
	fmt.Println("start")

	time.AfterFunc(time.Second, func() {
		fmt.Println("1秒后，结束")
		exitCh <- 0
	})

	// 等待结束
	<- exitCh
```

计时器( Timer)的原理和倒计时闹钟类似，都是给定多少时间后触发。打点器( Ticker) 的原理和钟表类似， 钟表每到整点就会触发。这两种方法创建后会返回 time.Ticker对象和 time.Timer对象，里面通过一个C成员， 类型是只能接收的时间通道 C<-chanTime)，使 用这个通道就可 以获得时间触发的通知。下面代码创建一个打点器， 每 500毫秒触发一起:创建一个计时器， 2秒后触发，只 触发一次。
```go
	// 创建一个打点器，每500毫秒触发一次
	ticker := time.NewTicker(time.Millisecond * 500)

	// 创建一个计时器，2秒后触发
	stopper := time.NewTimer(time.Second * 2)

	// 声明计数变量
	var i int

	for {			// 不断检查通道情况
		select {
		case <-stopper.C:		// 计时器到了
			fmt.Println("stop")
			goto StopHere		// 跳出循环
		case <-ticker.C:		// 打点触发了
			i++					// 记录触发多少次
			fmt.Println("tick", i)
		}
	}

	StopHere:
		fmt.Println("done")
```

