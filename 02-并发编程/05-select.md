## 一 select的概念

Go语言中的 select 关键字，可以同时响应多个通道的操作，在多个管道中随机选择一个能走通的路！   

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

如果有这样的需求，两个管道中只要有一个管道能够取出数据，那么就使用该数据：
```go
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
	go fn1(ch1)

	ch2 := make(chan string)
	go fn2(ch2)

	select {
	case r1 := <-ch1:
		fmt.Println("r1=", r1)
	case r2 := <-ch2:
		fmt.Println("r2=", r2)
	}
}
```

由于fn1延迟较低，则就会一直得到fn1的数据。

## 二 select的一些注意事项

### 2.1 defautl

select支持default，如果select没有一条语句可以执行，即所有的通道都被阻塞，那么有两种可能的情况：
- 如果给出了default语句，执行default语句，同时程序性的执行会从select语句后的语句中恢复
- 如果没有default语句，那么select语句将被阻塞，直到至少有一个通信可以进行下去
- 所以一般不写default语句 

当然，在一些场景中（for循环使用select获取channel数据），如果channel被写满，也可能会执行default。

注意：select中的case必须是I/O操作。  

#### 2.3 channel超时解决

在并发编程的通信过程中，最需要处理的是超时问题，即向channel写数据时发现channel已满，或者取数据时发现channel为空，如果不正确处理这些情况，会导致goroutine锁死，例如：
```go
// 如果永远没有人写入ch数据，那么上述代码一直无法获得数据，导致goroutine一直阻塞
i := <-ch
```

利用select()可以实现超时处理：
```go
	timeout := make(chan bool, 1)

	go func() {
		time.Sleep(1e9)			// 等待1秒钟
		timeout <- true
	}()

	select {
		case <-ch:      		// 能取到数据
		case <-timeout: 		// 没有从-cha中取到数据，此时能从timeout中取得数据
	}
```

#### 2.4 空select

空的select唯一的功能是阻塞代码：
```go
	select {}
```

## 三 select的一些案例

#### 3.1 案例一 模拟web开发
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

#### 3.2 案例二 使用通道晌应计时器的事件

Go语言中的 time 包提供了计时器的封装。由于 Go 语言中的通道和 goroutine 的设计， 定时任务可以在 goroutine 中以同步方式完成，也可以通过在 goroutine 中异步回调完成。   

实现一段时间之后：
```go
	exitCh := make(chan int)
	fmt.Println("start")

	time.AfterFunc(time.Second, func() {
		fmt.Println("1秒后，结束")
		exitCh <- 0
	})

	// 阻塞以等待结束
	<- exitCh
```

计时器与打点器：
- 计时器( Timer)与倒计时类似，也是给定多少时间后触发， 创建后会返回 time.Timer 对象
- 打点器( Ticker) 表示每到整点就会触发，创建后会返回 time.Ticker 对象

返回的时间对象内包含成员 C ，其类型是只能接收的时间通道 C<-chanTime ，使用这个通道就可以获得时间触发的通知。  

示例：创建一个打点器， 每500毫秒触发一起:创建一个计时器， 2秒后触发，只触发一次。  
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