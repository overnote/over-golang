## 一 Golang对协程的支持

Go语言从语言层面原生提供了协程支持，即goroutine，Go语言内部实现了多个goroutine之间的内存共享，让并发编程变得极为简单。执行goroutine只需极少的栈内存(大概是4~5KB)，可同时运行成千上万个并发任务。  

goroutine由Go的runtime管理，通过关键字`go`实现，示例：

```go
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
    go say("Go")						// 以协程方式执行say函数
    say("noGo")							// 以普通方式执行say函数
    time.Sleep(5 * time.Second)         // 睡眠5秒：防止协程未执行完毕，主程序退出
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

## 四 深入理解线程与协程

#### 4.1 理解线程

线程是OS中执行的最小处理单元，存在于某个进程中，即单个进程可能包含多个线程。  

比如Web服务器需要同时处理多个请求，这些请求彼此独立。当请求到达时，web 服务器会创建一个线程，或者从线程池中获取一个线程，然后将请求来委派给线程来实现并发。不过请记住 RobPike 的名言——『并发不是并行』。  

#### 4.2 线程不一定比进程轻量

线程是否比进程更加轻量取决于你从哪个角度看。理论上，线程之间共享内存，创建新线程的时候不需要创建真正的虚拟内存空间，也不需要 MMU（内存管理单元）上下文切换。此外，线程间通信比进程之间通信更加简单，主要是因为线程之间有共享内存，而进程通信往往需要利用各种模式的 IPC（进程间通信），如信号量，消息队列，管道等。  

但是在多处理器操作系统中，线程并不一定比进程更高效：例如 Linux 就是不区分线程和进程的，两者在 Linux 都被称作任务（task）。每个任务在 cloned 的时候都有一个介于最小到最大之间的共享级别。  
- 调用 fork() 创建任务时，创建的是一个没有共享文件描述符，PID 和内存空间的新任务。而调用 pthread_create() 创建任务时，创建的任务将包含上述所有共享资源。  
- 线程之间保持共享内存 与多核的L1 缓存 中的数据同步，与在隔离内存中运行不同的进程相比，需要付出更加大的代价。

任务切换付出的代价一直在Linux中被抹平，但是创建新任务还是要比创建新线程需要很多开销。  

#### 4.3 线程的改进方向

线程变慢的主要三个原因：
- 线程自身有一个很大的堆（≥ 1MB）占用了大量内存，如果一下创建 1000 个线程意味着需要 1GB 的内存！！！！！
- 线程需要重复存储许多寄存器，其中一些包括 AVX（高级向量扩展），SSE（流式 SIMD 外设），浮点寄存器，程序计数器（PC），堆栈指针（SP），这会降低应用程序性能。
- 线程创建和消除需要调用操作系统以获取资源（例如内存），而这一操作相对是比较慢的。

#### 4.4 goroutine

Goroutines 是在 Golang 中执行并发任务的方式，不过要切记：  
> Goroutines仅存在于 Go 运行时而不存在于 OS 中，因此需要 Go调度器（GoRuntimeScheduler） 来管理它们的生命周期。

Go运行时为此维护了三个C结构（https://golang.org/src/runtime/runtime2.go）：
- G 结构：表示单个 Goroutine，包含跟踪其堆栈和当前状态所需的对象。还包含自己负责的代码的引用。
- M 结构：表示 OS 线程。包含一些对象指针，例如全局可执行的 Goroutines 队列，当前运行的 Goroutine，它自己的缓存以及对 Go 调度器的引用。
- P 结构：也做Sched结构，它是一个单一的全局对象，用于跟踪 Goroutine 和 M 的不同队列以及调度程序运行时需要的其他一些信息，例如单一全局互斥锁（Global Sched Lock）。  

G 结构主要存在于两种队列之中，一个是 M （线程）可以找到任务的可执行队列，另外一个是一个空闲的 Goroutine 列表。调度程序维护的 M（执行线程）只能每次关联其中一个队列。为了维护这两种队列并进行切换，就必须维持单一全局互斥锁（Global Sched Lock）。  

因此，在启动时，go 运行空间会为 GC，调度程序和用户代码启动许多 Goroutine。并创建 OS 线程来处理这些 Goroutine。不过创建的线程数量最多可以等于 GOMAXPROCS（默认为 1，但为了获得最佳性能，通常设置为计算机上的处理器数量）。  

#### 4.5 协程对比线程的改进

为了使运行时的堆栈更小，go 在运行期间使用了大小可调整的有限堆栈，并且初始大小只有 2KB/goroutine。新的 Goroutine 通常会分配几 kb 的空间，这几乎总是足够的。如果不够的话，运行空间还能自动增长（或者缩小）内存来实现堆栈的管理，从而让大部分 Goroutine 存在于适量的内存中。每个函数调用的平均 CPU 开销大概是三个简单指令。因此在同一地址空间中创建数十万个 Goroutine 是切实可行的。但是如果 Goroutine 是线程的话，系统资源将很快被消耗完。 

#### 4.6 协程阻塞

当 Goroutine 进行阻塞调用时，例如通过调用阻塞的系统调用，这时调用的线程必须阻塞，go 的运行空间会操作自动将同一操作系统线程上的其他 Goroutine，将它们移动到从调度程序（Sched Struct）的线程队列中取出的另一个可运行的线程上，所以这些 Goroutine 不会被阻塞。因此，运行空间应至少创建一个线程，以继续执行不在阻塞调用中的其他 Goroutine。 而且关键的是程序员是看不到这一点的。结论是，我们称之为 Goroutines 的事物，可以是很低廉的：它们在堆栈的内存之外几乎没有开销，而内存中也只有几千字节。

Go 协程也可以很好地扩展。  

但是，如果你使用只存在于 Go 的虚拟空间的 channels 进行通信（产生阻塞时），操作系统将不会阻塞该线程。 只是让该 Goroutine 进入等待状态，并安排另一个可运行的 Goroutine（来自 M 结构关联的可执行队列）它的位置。  

#### 4.7 Go Runtime Scheduler

Go Runtime Scheduler 跟踪记录每个 Goroutine，并安排它们依次地在进程的线程池中运行。  

Go Runtime Scheduler 执行协作调度，这意味着只有在当前 Goroutine 阻塞或完成时才会调度另一个 Goroutine，这通过代码可以轻松完成。这里有些例子：
- 调用系统调用如文件或网络操作阻塞时
- 因为垃圾收集被停止后

这样比定时阻塞并调度新线程的抢占式调度要好得多，因为当线程数量增加，或者当高优先级任务将被调度运行时，有低优先级的任务已经在运行了（此时低优先级队列将被阻塞），定时抢占调度可能导致某些任务完成花费的时间大大超过实际所需时间。  

另一个优点是，因为 Goroutine 在代码中隐式调用的，例如在睡眠或 channel 等待期间，编译只需要安全地恢复在这些时刻处存活的寄存器。在 Go 中，这意味着在上下文切换期间仅更新 3 个寄存器，即 PC，SP 和 DX（数据寄存器） 而不是所有寄存器（例如 AVX，浮点，MMX）。

## 五 goroutine 与 coroutine

C#、 Lua、 Python语言都支持协程 coroutine（Java也有一些第三方库支持）。  

coroutine与 goroutine都可以将函数或者语句在独立的环境中运行，但是它们之间有两点不同：
- goroutine可能发生并行执行，coroutine始终顺序执行
- goroutine 使用 channel 通信，coroutine 使用 yield 和 resume
  
> coroutine 程序需要主动交出控制权，宿主才能获得控制权并将控制权交给其他 coroutine  

coroutine 的运行机制属于协作式任务处理。在早期的操作系统中，应用程序在不需要使用 CPU 时，会主动交出 CPU 使用权。如果开发者故意让应用程序长时间占用 CPU，操作系统也无能为力。coroutine 始终发生在单线程。

> goroutine可能发生在多线程环境下， goroutine无法控制自己获取高优先度支持  

goroutine 属于抢占式任务处理，和现有的多线程和多进程任务处理非常类似。应用程序对 CPU 的控制最终还需要由操作系统来管理，操作系统如果发现一个应用程序长时间大量地占用 CPU，那么用户有权终止这个任务。

## 六 Go协程总结

Go协程的特点：
- 有独立的栈空间
- 共享程序堆空间
- 调度由用户控制

注意：
- Go程序在启动时，就会为main函数创建一个默认的goroutine，也就是入口函数main本身也是一个协程
- 如果主线程退出了，则协程即使还没有执行完毕也退出 