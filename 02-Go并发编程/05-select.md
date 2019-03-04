## 一 select

#### 1.0 select示例

在有多个channel时，肯定不能让其串行执行：

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

	case r1 := <- ch1 :
		fmt.Println("r1:", r1)
	}

}	
```

select同时监听多个channel，如果某个channel可读（有数据）则执行。如果多个channel同时可读（即上述fn1和fn2的延时时间都是3秒）那么随机读取一个channel的数据。

#### 1.1 select的作用

关键字select可以监听channel上的数据流动，用法与switch类似，由select开始下一个新的需选择块，每个选择条件由case语句来描述，但是在使用方面有一条限制，最大的限制是：
每个case语句必须是一个IO操作，大致的结构如下：

```Go
select {
    case <-chan1:
        //如果chan1成功读取到数据，执行该case
    case chan2 <- 1:
        //如果成功向chan2写入数据，则执行该case处理语句
    default:
        //如果上面没有成功，则进入default处理流程
}
```

在一个select语句中，Go会按顺序从头到尾评估每一个发送和接收的语句。如果其中任意一个语句可以继续执行，即没有被阻塞，那么久从那些可以被之心个的语句中任意选择一条来使用。

如果没有一条语句可以执行，即所有的通道都被阻塞，那么有两种可能的情况：
- 如果给出了default语句，执行default语句，同时程序性的执行会从select语句后的语句中恢复
- 如果没有default语句，那么select语句将被阻塞，直到至少有一个通信可以进行下去。
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
select 案例：
```go

```
#### 1.2 channel超时解决
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