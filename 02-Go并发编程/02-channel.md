## 一 channel

#### 1.0  channel的引出

单纯的将函数并发执行是没有意义的，函数与函数之间必须能够交换数据才能体现并发执行函数的意义。虽然可以使用共享内存进行数据交换，但是共享内存在不同的goroutine中容易发生竞态问题，为了保证数据的正确性，必须使用互斥量对内存进行加锁，这种做法又肯定造成了性能问题。  

Go提倡使用通信的方式代替共享内存，即使用通道channel。  

![](/images/Golang/channel01.png)

多个 goroutine 为了争抢数据，势必造成执行的低效率，使用队列的方式是最高效的， channel 就是一种队列一样的结构 。

#### 1.1 channel的使用

channel特性：
- channel的本质是一个数据结构-队列，先进先出
- channel是线程安全的，多goroutine访问时，不需要加锁，因为在任何时候，同时只能有一个goroutine访问通道。
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
	//指定长度为10，不指定空间长度，则不能存储数据
	ch := make(chan int, 10)		

	ch <- 100									//向c中存储一个数据100
	fmt.Printf("入队后数据：%v\n",ch)			//0xc000074000
	ch <- 200
	ch <- 300

	<- ch									//直接出队一个数据，且不使用	

	data1 := <-ch
	data2 := <-ch
	fmt.Printf("取出的数据data1：%v\n", data1)	//200
	fmt.Printf("取出的数据data2：%v\n", data2)	//300

```

#### 1.2 channel的遍历和关闭

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
#### 1.3 单向channel只读与只写

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


#### 1.4 无缓冲的channel

```go
ch := make(chan string, 2)		//2 即是缓存的长度，即ch是带缓存的channel
```

无缓冲的通道指在接收前没有能力保存任何值的通道，这种类型的通道要求发送goroutine和接收goroutine同时准备好，否则会导致先执行发送或接收操作的goroutine阻塞。

这种对通道的操作行为是同步的，其中任意一个操作都无法离开另一个操作单独存在。  

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

#### 1.5 全局唯一操作

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

#### 1.6 channel的长度与容量

容量是channel的最大长度：cap(ch)  
长度是channel当前包含元素数：len(ch)

####  1.7 通道阻塞

发送数据阻塞：  

把数据往通道中发送时，如果接收方一直都没有接收，那么发送操作将被持续阻塞，Go程序运行时能智能的发现一些永远无法成功发送的语句并提示错误
```go
	ch := make(chan int)			//添加一个int类型的第二个参数不会报错
	ch <- 0							//报错：`all goroutines are asleep`。
```

接收数据阻塞：
```go

	ch := make(chan int, 3)
	ch<- 1
	ch<- 2
	ch<- 3	

	data1 := <-ch			//阻塞接收数据
	fmt.Println(data1)

	data2,ok := <-ch		//非阻塞接收数据
	fmt.Println(data2)
	fmt.Println(ok)			//true
```

非阻塞通道接收会造成很高的cpu占用，因此使用较少，如果需要实现接收超时检测，推荐select。  

#### 1.8 通道关闭

给被关闭的通道发送数据会造成panic：
```go

	ch := make(chan int, 3)
	ch<- 1	

	close(ch)

	ch<- 4
```

不过已经关闭的通道在接收数据时不会发生阻塞。

