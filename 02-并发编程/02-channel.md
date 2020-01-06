## 一 channel简介

如果在程序中开启了多个goroutine，那么这些goroutine之间该如何通信呢？  

go提供了一个channel（管道）数据类型，可以解决协程之间的通信问题！channel的本质是一个队列，遵循先进先出规则（FIFO），内部实现了同步，确保了并发安全！  

## 二 channel的创建

### 2.0 channel的语法

channel在创建时，可以设置一个可选参数：缓冲区容量
- 创建有缓冲channel：`make(chan int, 10)`，创建一个缓冲长度为10的channel
- 创建无缓冲channel：`make(chan int)`，其实就是第二个参数为0

channel内可以存储多种数据类型，如下所示：
```go
	ci := make(chan int)
	cs := make(chan string)
	cf := make(chan interface{})				
```

从管道中读取，或者向管道写入数据，使用运算符：`<-`，他在channel的左边则是读取，右边则代表写入：
```go
	ch := make(chan int)
	ch <- 10				// 写入数据10
	num := <- ch			// 读取数据
```

### 2.1 无缓冲channel

无缓冲的channel是阻塞读写的，必须写端与读端同时存在，写入一个数据，则能读出一个数据：
```go
package main

import (
	"fmt"
	"time"
)

// 写端
func write(ch chan int) {
	ch <- 100
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 200
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 300                      // 该处数据未读取，后续操作直接阻塞
	fmt.Printf("ch addr：%v\n", ch) // 没有输出
}

// 读端
func read(ch chan int) {
	// 只读取两个数据
	fmt.Printf("取出的数据data1：%v\n", <-ch) // 100
	fmt.Printf("取出的数据data2：%v\n", <-ch) // 200
}

func main() {

	var ch chan int     // 声明一个无缓冲的channel
	ch = make(chan int) // 初始化

	// 向协程中写入数据
	go write(ch)

	// 向协程中读取数据
	go read(ch)

	// 防止主go程提前退出，导致其他协程未完成任务
	time.Sleep(time.Second * 3)
}
```

注意：无缓冲通道的收发操作必须在不同的两个`goroutine`间进行，因为通道的数据在没有接收方处理时，数据发送方会持续阻塞，所以通道的接收必定在另外一个 goroutine 中进行。  

如果不按照该规则使用，则会引起经典的Golang错误`fatal error: all goroutines are asleep - deadlock!`：
```go
func main() {
	ch := make(chan int)
	ch <- 10
	<-ch
}
```

### 2.2 有缓存channel

有缓存的channel是非阻塞的，但是写满缓冲长度后，也会阻塞写入。
```go
package main

import (
	"fmt"
	"time"
)

// 写端
func write(ch chan int) {
	ch <- 100
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 200
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 300                      // 写入第三个，造成阻塞
	fmt.Printf("ch addr：%v\n", ch) // 没有输出
}
func main() {

	var ch chan int        // 声明一个有缓冲的channel
	ch = make(chan int, 2) // 可以写入2个数据

	// 向协程中写入数据
	go write(ch)

	// 防止主go程提前退出，导致其他协程未完成任务
	time.Sleep(time.Second * 3)
}
```

同样的，当数据全部读取完毕后，再次读取也会造成阻塞，如下所示：
```go
func main() {
	ch := make(chan int, 1)
	ch <- 10
	<-ch
	// <-ch
}
```  
此时程序可以顺序运行，不会报错，这是与无缓冲通道的区别，但是当继续打开 注释 部分代码时，通道阻塞，所有协程挂起，此时也会产生错误：`fatal error: all goroutines are asleep - deadlock!`。

### 2.3 总结 无缓冲通道与有缓冲通道

无缓冲channel：
- 通道的容量为0，即 `cap(ch) = 0`
- 通道的个数为0，即 `len(ch) = 0`
- 可以让读、写两端具备并发同步的能力

有缓冲channel：
- 在make创建的时候设置非0的容量值
- 通道的个数为当前实际存储的数据个数
- 缓冲区具备数据存储的能力，到达存储上限后才会阻塞，相当于具备了异步的能力
- 有缓冲channel的阻塞产生条件：
  - 带缓冲通道被填满时，尝试再次发送数据会发生阻塞
  - 带缓冲通道为空时，尝试接收数据会发生阻塞

问题：为什么 Go语言对通道要限制长度而不提供无限长度的通道?  
> channel是在两个 goroutine 间通信的桥梁。使用 goroutine 的代码必然有一方提供数据，一方消费数据 。通道如果不限制长度，在生产速度大于消费速度时，内存将不断膨胀直到应用崩溃。