## 一 常见相关API
#### 1.1 runtime.Gosched()     
让出CPU时间片，让出当前goroutine执行权限，调度器安排其他等待的任务运行，并在小瑕疵某个时候从该位置恢复执行。
类似于接力赛拍，A跑了一段时间碰到了Gosched()把接力棒交给B继续跑：
```Go
func fn (s string) {
	for i := 0; i < 2; i++ {
		fmt.Println(s)
	}
}

func main(){

	go fn("hello")

	for i := 0; i < 2; i++ {
		//runtime.Gosched()
		fmt.Println("world")
	}

}
```
#### 1.2 runtime.GOMAXPROCS()
上述用来设置可以并行计算CPU核心数的最大值，并返回之前的值。
```Go
    //依次增大1 2 3 切换频次也会越来越大
	n := runtime.GOMAXPROCS(1)
	fmt.Println("n===",n)
	for {
		go fmt.Println(1)
		fmt.Println(0)
    }
```
## worker池
如果有一个任务就启用一个goroutine，那么会造成协程数量越来越多，利用生产者消费者模型，创建一个workder池，可以控制goroutine数量，防止goroutine泄露：

```go

```