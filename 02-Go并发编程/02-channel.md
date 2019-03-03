## 一 全局互斥锁 
不同的goroutine间通讯方式之间有两种通讯方式：
- 全局变量的互斥锁
- 使用channel解决
全局互斥锁案例：计算1-20的各个数的阶乘，并存储在map中。
```go
package main
import (
	"fmt"
	"time"
)

var factorialMap = make(map[int]int, 10)
func factorial(num int) {
	res := 1
	for i := 1; i <= num; i++ {
		res *= i
	}
	factorialMap[num] = res
}

func main() {

	for i := 1; i <= 20; i++ {
		go factorial(i)
	}

	//休眠10秒钟后，看结果
	time.Sleep(time.Second * 10)
	for i, v := range factorialMap {
		fmt.Printf("map[%d]=%d\n", i, v) 
	}

}
```
由于上述代码没有对全局变量factorialMap加锁，因此会出现资源争夺问题，代码会出现错误，提示
```
fatal error: concurrent map writes
```
调整后的代码：
```go
package main
import (
	"fmt"
	"time"
	"sync"
)

var (
	factorialMap = make(map[int]int, 10)
	lock sync.Mutex				//全局互斥锁lock，sync：同步，Mutex：互斥
)
func factorial(num int) {
	res := 1
	for i := 1; i <= num; i++ {
		res *= i
	}
	lock.Lock()					//加锁
	factorialMap[num] = res
	lock.Unlock()				//解锁
}

func main() {

	for i := 1; i <= 20; i++ {
		go factorial(i)
	}

	//休眠10秒钟后，看结果
	lock.Lock()
	time.Sleep(time.Second * 10)
	for i, v := range factorialMap {
		fmt.Printf("map[%d]=%d\n", i, v) 
	}
	lock.Unlock()

}
```
## 二 channel
#### 2.1 channel的使用
在上述计算阶乘的案例中，使用全局变量加锁同步来解决 goroutine 的通讯，但不完美，主线程在等待所有 goroutine 全部完成的时间很难确定，我们这里设置 10 秒，仅仅是估算。如果主线程休眠时间长了，会加长等待时间，如果等待时间短了，可能还有 goroutine 处于工作
状态，这时也会随主线程的退出而销毁。通过全局变量加锁同步来实现通讯，也并不利用多个协程对全局变量的读写操作。  
为了解决上述问题，Go推出了channel：
- channel的本质是一个数据结构-队列，先进先出
- channel是线程安全的，多goroutine访问时，不需要加锁
- channel拥有类型，一个string的channle只能存放string类型数据
如图所示：
![](/images/Golang/并发-02.png)

channel的定义：channel的定义：var 变量名 chan 数据类型，但是没有初始化不能使用。
```go
	var c1 chan int							    //声明channel c1
	fmt.Printf("初始化前，c1=%v\n", c1)			//c1=<nil>
	c1 = make(chan int)
	fmt.Printf("初始化后，c1=%v\n", c1)			//c1=0xc000042060
```

channel内可以存储多种数据类型：
```go
//初始化channel
ci := make(chan int)
cs := make(chan string)
cf := make(chan interface{})
```

channel的操作：channel通过操作符`<-`来接收和发送数据
```go
	ch := make(chan int, 10)//指定长度为10，不指定空间长队，则不能存储数据
	ch <- 100				//向c中存储一个数据100
	fmt.Printf("入队后数据：%v\n",ch)	//0xc000074000
	ch <- 200
	ch <- 300

	<- ch				//直接出队一个数据，且不使用	

	data1 := <-ch
	data2 := <-ch
	fmt.Printf("取出的数据data1：%v\n", data1)	//200
	fmt.Printf("取出的数据data2：%v\n", data2)	//300
```

channel队列让协程同步：
```go
package main

import (
	"fmt"
	"time"
)

func test(c chan bool) {
	time.Sleep(5 * time.Second)
	fmt.Println("异步执行test...")
	c <- true
}


func main() {

	var exitChan chan bool
	exitChan = make(chan bool)
	go test(exitChan)
	fmt.Println("主线程执行...")
	<-exitChan		//输出主线程执行后，去管道取数据，但是管道没有数据，会一直等待数据，即等待go协程函数填入队列

}	
```
#### 2.2 channel的遍历和关闭
channel只支持 for--range 的方式进行遍历，请注意两个细节
- 在遍历时，如果 channel 没有关闭，则回出现 deadlock 的错误
- 在遍历时，如果 channel 已经关闭，则会正常遍历数据，遍历完后，就会退出遍历。
使用内置函数 close 可以关闭 channel, 当 channel 关闭后，就不能再向 channel 写数据了，但是仍然 可以从该 channel 读取数据。
```go
	ch := make(chan int, 10)
	for i := 0; i < 10; i++ {
		ch <- i
	}

	//关闭
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
#### 2.3 单向channel只读与只写

默认情况下管道是双向的，但是也可以声明为为只读，或者只写性质。  

单向channel只能用来发数据或者接受数据，事实上由于不能让goroutine锁死，channel必须是双向的，这里的单向channel只是对其使用的限制。

```go
	var chan1 chan<- int		//只写
	var chan2 <-chan int 		//只读
```

单向channel的转换与使用：
```Go

ch4 := make(chan int)
ch5 := <-chan int(ch4)
ch6 := chan<- int(ch4)
func Parse(ch <-chan int) {
    for value := range ch {
        fmt.Println(“Parsing value,” value)
    }
}
```
#### 2.4 全局唯一操作

某些函数只允许调用一次，但是有goroutine的情况下无法控制，Go引入了Once：

```Go
var a string
var onec sync.Once
func setup() {
    a = “hi”
}
func doprint() {
    once.Do(setup)
    print(a)
}
```
如果主协程退出，那么其他子协程也会退出

#### 2.5 无缓冲的channel

无缓冲的通道指在接收前前没有能力保存任何值得通道，这种类型的通道要求发送goroutine和接收goroutine同时准备好，否则会导致先执行发送或接收操作的goroutine阻塞。
这种对通道的操作行为是同步的，其中任意一个操作都无法离开另一个操作单独存在。

## 三 select

#### 3.1 select的作用

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
#### 3.2 channel超时解决
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