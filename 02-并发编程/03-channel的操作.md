## 三 channel的相关操作

### 3.1 通道数据的遍历

channel只支持 for--range 的方式进行遍历：
```go
	ch := make(chan int)

	go func() {
		for i := 0; i <=3; i++ {
			ch <- i
			time.Sleep(time.Second)
		}
	}()

	for data := range ch {
		fmt.Println("data==", data)
		if data == 3 {
			break
		}
	}
```

### 3.2 通道关闭

通道是一个引用对象，支持GC回收，但是通道也可以主动被关闭：
```go
	ch := make(chan int)
	close(ch)				// 关闭通道
	ch <- 1					// 报错：send on closed channel
```

从通道中接收数据时，可以利用多返回值判断通道是否已经关闭：
```go
func main() {

	ch := make(chan int, 10)

	go func(ch chan int) {
		for i := 0; i < 10; i++ {
			ch <- i
		}
		close(ch)
	}(ch)

	for {
		if num, ok := <-ch; ok == true {
			fmt.Println("读到数据：", num)
		} else {
			break
		}
	}
} 
```

如果channel已经关闭，此时需要注意：
- 不能再向其写入数据，否则会引起错误：`panic:send on closed channel`
- 可以从已经关闭的channel读取数据，如果通道中没有数据，会读取通道存储的默认值

### 3.3 通道读写

默认情况下，管道的读写是双向的，但是为了对 channel 进行使用的限制，可以将 channel 声明为只读或只写。  
```go
	var chan1 chan<- int		// 声明 只写channel
	var chan2 <-chan int 		// 声明 只读channel
```

单向chanel不能转换为双向channel，但是双向channel可以隐式转换为任意类型的单向channel：
```go
// 只写端
func write(ch chan<- int) {
	ch <- 100
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 200
	fmt.Printf("ch addr：%v\n", ch) // 输出内存地址
	ch <- 300                      // 该处数据未读取，后续操作直接阻塞
	fmt.Printf("ch addr：%v\n", ch) // 没有输出
}

// 只读端
func read(ch <-chan int) {
	// 只读取两个数据
	fmt.Printf("取出的数据data1：%v\n", <-ch) // 100
	fmt.Printf("取出的数据data2：%v\n", <-ch) // 200
}

func main() {

	var ch chan int         // 声明一个双向
	ch = make(chan int, 10) // 初始化

	// 向协程中写入数据
	go write(ch)

	// 向协程中读取数据
	go read(ch)

	// 防止主go程提前退出，导致其他协程未完成任务
	time.Sleep(time.Second * 3)
}
```

双向channel进行显式转换：
```Go
	ch := make(chan int)		// 声明普通channel
	ch1 := <-chan int(ch)		// 转换为 只读channel
	ch2 := chan<- int(ch)		// 转换为 只写channel
```