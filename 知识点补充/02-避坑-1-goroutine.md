## 一 goroutine

#### 1.1 goroutine 需要合理退出

Goroutine经常用来处理耗时操作，但是如果无限制创建协程，不考虑协程的退出和生命周期，容易造成goroutine的失控： 
```go
	ch := make(chan int)
	for {

		var dummy string
		fmt.Scan(&dummy)

		go func(ch chan int) {				// 模拟耗时操作
			for {
				data := <-ch
				fmt.Println(data)
			}
		}(ch)

		fmt.Println(runtime.NumGoroutine())
	}
```
程序运行后，随着输入的字符串越来越多，goroutine将会无限制地被创建，但并不会结束，为了避免这种情况，可以为协程添加退出条件：
```go
	ch := make(chan int)
	for {

		var dummy string
		fmt.Scan(&dummy)

		// 设置强行退出条件，方式协程内部的退出条件无法触发
		if dummy == "quit" {
			for i := 0; i < runtime.NumGoroutine(); i++ {
				ch <- 0
			}
			continue
		}

		go func(ch chan int) {				// 模拟耗时操作
			for {
				data := <-ch
				if data == 0{				// 添加协程内部退出条件
					break
				}
				fmt.Println(data)
			}
		}(ch)

		fmt.Println(runtime.NumGoroutine())
	}
```

#### 1.2 不能滥用goroutine

为了保证多个 goroutine 并发访问的安全性，通道也需要做一些锁操作，因此通道其实并不比锁高效。对于 TCP 来说， 一般是接收过程创建 goroutine 并发处理 。当套接字结束时，就要正常退出这些 goroutine。  

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
 
 