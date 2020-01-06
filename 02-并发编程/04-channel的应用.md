## 四 channel的一些示例

### 4.1 示例一 限制并发

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
		time.Sleep(time.Second * 5)
	}
}
```

### 4.2 生产者消费者模型
```go
package main

import (
	"fmt"
	"time"
)

// 生产者
func producer(ch chan<- int) {
	i := 1
	for {
		ch <- i
		fmt.Println("Send:", i)
		i++
		time.Sleep(time.Second * 1) // 避免数据流动过快
	}
}

// 消费者
func consumer(ch <-chan int) {
	for {
		v := <-ch
		fmt.Println("Receive:", v)
		time.Sleep(time.Second * 2) // 避免数据流动过快
	}
}

func main() {

	// 生产消费模型中的缓冲区
	ch := make(chan int, 5)
	// 启动生产者
	go producer(ch)
	// 启动消费者
	go consumer(ch)
	// 阻塞主程序退出
	for {

	}
}
```

当然，该模型也可以使用无缓冲模型，区别如下：
- 无缓冲生产消费模型：同步通信
- 有缓冲生产消费模型：异步通信

### 4.3 定时器

定时器`time.Timer`的底层其实也是一个管道。
```go
tyoe Timer struct {
    C <-chan Time
    r runtimeTimer
}
```
在时间到达之前，没有数据写入，则timer.C会一直阻塞，直到时间到达，系统会自动向timer.C中写入当前时间，阻塞就会被解除：
```go
    // 创建定时器 定义延迟时间为2秒
	layTimer := time.NewTimer(time.Second * 2)
	// 从管道取数据，但是一直都是空的，阻塞中，直到2庙后有数据才能取出
	fmt.Println(<-layTimer.C)
```

定时器的其他操作：
- Stop()：停止定时器，此时如果从管道中取数据，则会阻塞
- Reset()：重置定时器，此时需要传入一个新的定时时间间隔
- timer.Tiker：可以创建一个周期定时器