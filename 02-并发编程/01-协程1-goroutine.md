## 一 Golang对协程的支持

Go语言从语言层面原生提供了协程支持，即goroutine，Go语言内部实现了多个goroutine之间的内存共享，让并发编程变得极为简单。  

执行goroutine只需极少的栈内存(大概是4~5KB)，可同时运行成千上万个并发任务。  

goroutine简单理解：goroutine是通过Go的runtime管理的一个线程管理器，通过关键字`go`实现。    

示例：

```Go
package main

import (
    "fmt"
    "time"
)

func say(s string) {
    for i := 0; i < 3; i++ {
        fmt.Println(s)
    }
}

func main() {
    go say("Go")						// 以协程方式执行
    say("noGo")							// 以普通方式执行
    time.Sleep(5 * time.Second)         // 简单防止该情况： 协程未执行完毕，主程序退出
}
```

输出结果：
```
noGo
noGo
noGo
Go
Go
Go
```

## 二 goroutine使用案例

#### 2.1 同时执行两件事

```go
package main

import (
	"fmt"
	"time"
)

func running() {
	var times int
	for {
		times++
		fmt.Println("tick:", times)
		time.Sleep(time.Second)
	}
}

func main() {

	go running()

	var input string
	fmt.Scanln(&input)

}
```
命令行会不断地输出 tick，同时可以使用 `fmt.Scanln()` 接受用户输入。两个环节可以同时进行，直到按 Enter键时将输入的内容写入 input变量中井返回，
整个程序终止。

#### 2.2 一道基础面试题

示例一：
```go
	for i := 1; i <= 10; i++ {
		go func(){
			fmt.Println(i)		// 全部打印11：因为开启协程也会耗时，协程没有准备好，循环已经走完
		}()

	}
	time.Sleep(time.Second)
```

示例二：
```go
	for i := 1; i <= 10; i++ {
		go func(i int){
			fmt.Println(i)		// 打印无规律数字
		}(i)

	}
	time.Sleep(time.Second)
```

## 三 常用API  

#### 3.1 GOMAXPROCS() 多核利用

goroutine的概念类似于线程，但goroutine由Go程序运行时的调度和管理（线程是由进程的地址空间管理）。 Go程序调度器可以高效地将 CPU 资源分配给每一个任务。   
在Go1.5之后，程序默认运行在多核上，无需设置，1.5之前需要如下设置：
```go
	cpuNum := runtime.NumCPU()				//获取当前系统的CPU核心数
	runtime.GOMAXPROCS(cpuNum)				//Go中可以轻松控制使用核心数
```

Go使用该函数实现了并行执行。

#### 3.2 Gosched() 让出时间片

`runtime.Gosched()`:用于让出CPU时间片，即让出当前协程的执行权限，调度器安排其他等待的任务运行。  

贴士：可以理解为接力赛跑：A跑了一段遇到了Gosched接力给B。  


```Go
func main(){

    for i := 1; i <= 10; i++ {
        go func(i int){
            if i == 5 {
                runtime.Gosched()	// 协程让出，5永远不会第一输出
            }
            fmt.Println(i)		// 打印一组无规律数字
        }(i)

    }
    time.Sleep(time.Second)
}
```

#### 3.3 Goexit() 终止当前协程

`runtime.Goexit()`:用于立即终止当前协程运行，调度器会确保所有已注册defer延迟调用被执行。  

```go
func main(){
    
    for i := 1; i <= 5; i++ {
        defer fmt.Println("defer ", i)
        go func(i int){
            if i == 3 {
                runtime.Goexit()
            }
            fmt.Println(i)
        }(i)

    }

    time.Sleep(time.Second)

}
```

输出的结果类似于：
```
4
2
5
1
defer  5
defer  4
defer  3
defer  2
defer  1
```

## 四 goroutine 与 coroutine

C#、 Lua、 Python语言都支持协程 coroutine。 coroutine与 goroutine都可以将函数或者语句在独立的环境中运行，但是它们之间有两点不同：
- goroutine可能发生并行执行，coroutine始终顺序执行
- goroutine 使用 channel 通信，coroutine 使用 yield 和 resume
  
> coroutine 程序需要主动交出控制权，宿主才能获得控制权并将控制权交给其他 coroutine  

coroutine 的运行机制属于协作式任务处理。在早期的操作系统中，应用程序在不需要使用 CPU 时，会主动交出 CPU 使用权。如果开发者故意让应用程序长时间占用 CPU，操作系统也无能为力。coroutine 始终发生在单线程。

> goroutine可能发生在多线程环境下， goroutine无法控制自己获取高优先度支持  

goroutine 属于抢占式任务处理，和现有的多线程和多进程任务处理非常类似。应用程序对 CPU 的控制最终还需要由操作系统来管理，操作系统如果发现一个应用程序长时间大量地占用 CPU，那么用户有权终止这个任务。

## 五 Go协程总结

Go协程的特点：
- 有独立的栈空间
- 共享程序堆空间
- 调度由用户控制
- 协程是轻量级的线程
注意：如果主线程退出了，则协程及时还没有执行完毕也退出 

贴士：Go程序在启动时，就会为main函数创建一个默认的goroutine。