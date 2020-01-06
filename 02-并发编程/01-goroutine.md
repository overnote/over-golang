## 一 Golang对协程的支持

Go语言从语言层面原生提供了协程支持，即 `goroutine`，执行goroutine只需极少的栈内存(大概是4~5KB)，所以Go可以轻松的运行多个并发任务。  

Go中以关键字 `go` 开启协程：
```go
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

上述的程序的打印结果并不是按照顺序的，因为go关键字开启了协程，两个say函数并不是在一个控制流中！  

## 二 goroutine使用案例

### 2.1 同时执行两件事

```go
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

### 2.2 一道基础面试题

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

### 3.1 GOMAXPROCS() 多核利用

goroutine相关API位于 `runtime` 包。  

如果Go程序要运行在多核上，则可以如下操作，此时可以实现程序的并行执行：
```go
	cpuNum := runtime.NumCPU()				//获取当前系统的CPU核心数
	runtime.GOMAXPROCS(cpuNum)				//Go中可以轻松控制使用核心数
```
贴士：在Go1.5之后，程序已经默认运行在多核上，无需上述设置

### 3.2 Gosched() 让出时间片

`runtime.Gosched()`：用于让出CPU时间片，即让出当前协程的执行权限，调度器可以去安排其他等待的任务运行。示例如下： 
```Go
func main(){
    for i := 1; i <= 10; i++ {
        go func(i int){
            if i == 5 {
                runtime.Gosched()	// 协程让出，但并不代表不执行，而是 5永远不会第一输出
            }
            fmt.Println(i)		// 打印一组无规律数字
        }(i)
    }
    time.Sleep(time.Second)
}
```

贴士：可以理解为接力赛跑：A跑了一段遇到了Gosched接力给B。

### 3.3 Goexit() 终止当前协程

`runtime.Goexit()`：用于立即终止当前协程运行，调度器会确保所有已注册defer延迟调用被执行。  

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