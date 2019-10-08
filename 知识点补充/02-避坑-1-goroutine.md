## 一 goroutine

#### 1.1 goroutine 需要合理退出

在Go语言中，开发者吸光将并发内容与goroutine一一对应创建，但是很少会考虑如何退出和控制goroutine生命周期，这会造成goroutine失控。  

列举一个常见的例子：
```go
package main

import (
	"fmt"
	"runtime"
)

// 一个耗时操作
func consumer(ch chan int) {
	for {
		data := <-ch
		fmt.Println(data)
	}
}

func main() {

	ch := make(chan int)
	for {
		var dummy string
		fmt.Scan(&dummy)
		go consumer(ch)
		fmt.Println(runtime.NumGoroutine())
	}
}
```

这个程序实际在模拟一个进程根据需要创建 goroutine 的情况。运行后， 问题已经被 暴露出来: 随着输入的字符串越来越多， goroutine将会无限制地被创建， 但并不会结束。 这种情况如果发生在生产环境中 ， 将会造成内存大量分配，最终使进程崩溃。 现实的情况 也许比这段代码更加隐蔽 :也许你设置了一个退出的条件，但是条件永远不会被满足或者 触发。  

为了避免这个情况，需要为consumer()函数添加合理退出条件：
```go
package main

import (
	"fmt"
	"runtime"
)

// 一个耗时操作
func consumer(ch chan int) {
	for {
		data := <-ch
		if data == 0 {
			break
		}
		fmt.Println(data)
	}
}

func main() {

	ch := make(chan int)
	for {
		var dummy string
		fmt.Scan(&dummy)
		if dummy == "exit" {
			for i := 0; i < runtime.NumGoroutine() - 1; i++ {
				ch <- 0
			}
			continue
		}
		go consumer(ch)
		fmt.Println(runtime.NumGoroutine())
	}
}
```

#### 1.2 不能滥用goroutine

通道( channel)和 map、切片 一样 ，也是由 Go 源码编写而成。为了保证两 个 goroutine 并发访问的安全性，通道也需要做 一些锁操作 ， 因此通道其实并不 比锁高效。  

下面的例子展 示套接字 的接收和 井发管理。对于 TCP 来 说， 一般是接 收过程创建 goroutine 并发处理 。当套接字结束时·，就要正常退出这些 goroutine。  

```go
package main

import (
	"fmt"
	"net"
	"time"
)

func socketRecv(conn net.Conn, exitChan chan string) {
	buf := make([]byte, 1024)
	for {								// 创建循环不停接收数据
		_, err := conn.Read(buf)		// 从套接字中读取数据
		if err != nil {
			break
		}
	}
	exitChan <- "exit"
}

func main() {
	conn, err := net.Dial("tcp", "www.163.com:80")
	if err != nil {
		fmt.Println(err)
		return
	}

	exit := make(chan string)
	go socketRecv(conn, exit)
	time.Sleep(time.Second)
	conn.Close()				// 主动关闭套接字，触发err
	fmt.Println(<-exit)

}
```

在这个例子中， goroutine 退出使用通道来通知，这种做法可以解决 问题，但是实际上
通道中的数据并没有完全使用。  

通道的内部实现代码在 Go 语言开发包的 src/runtime/chan.go 中 ，经过分析后大概了解 到通道也是用常见的互斥量等进行同步 。 因此通道虽然是一个语言级特性，但也不是被神化 的特性，通道的运行和使用都要 比传统互斥量、 等待组( sync.WaitGroup)有一定的消耗。  

修改后的例子如下：
```go
package main

import (
	"fmt"
	"net"
	"sync"
	"time"
)

func socketRecv(conn net.Conn, wg *sync.WaitGroup) {
	buf := make([]byte, 1024)
	for {								// 创建循环不停接收数据
		_, err := conn.Read(buf)		// 从套接字中读取数据
		if err != nil {
			break
		}
	}
	wg.Done()
}

func main() {
	conn, err := net.Dial("tcp", "www.163.com:80")
	if err != nil {
		fmt.Println(err)
		return
	}

	var wg sync.WaitGroup
	wg.Add(1)
	go socketRecv(conn, &wg)

	time.Sleep(time.Second)
	conn.Close()				// 主动关闭套接字，触发err

}
```
 
 