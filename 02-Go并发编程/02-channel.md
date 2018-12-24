## 一 channel
#### 1.1 channel的创建
```Go
var ch chan int					//创建一个传递类型为int的channel
var m map[string] chan bool		//声明一个map，元素是bool类型的channel
ch := make(chan int)			//声明并初始化了一个int型的名为ch的channel

//将数据写入channel：	写入数据会阻塞程序，直到有goroutine从这个channel中取出。
ch <- value

//从channel中取数据：如果channel之前写入数据，取数据也会阻塞程序，直到写入
value := <-ch

```
#### 1.2 超时机制
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
#### 1.3 单向channel
单向channel只能用来发数据或者接受数据，事实上由于不能让goroutine锁死，channel必须是双向的，这里的单向channel只是对其使用的限制。
```Go
var ch1 chan int 		//ch1是一个正常channel
var ch2 chan<- float64	//ch2只能用于写入float64数据
var ch3 <-chan int		//ch3只能用于读取int数据

//单向channel的转换与使用：
ch4 := make(chan int)
ch5 := <-chan int(ch4)
ch6 := chan<- int(ch4)
func Parse(ch <-chan int) {
    for value := range ch {
        fmt.Println(“Parsing value,” value)
    }
}
```
#### 1.4 channel关闭
关闭语法：close(ch)

判断channel是否关闭：
x, ok := <-ch		//看ok这个布尔值即可。

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

#### 1.6 无缓冲的channel
无缓冲的通道指在接收前前没有能力保存任何值得通道，这种类型的通道要求发送goroutine和接收goroutine同时准备好，否则会导致先执行发送或接收操作的goroutine阻塞。
这种对通道的操作行为是同步的，其中任意一个操作都无法离开另一个操作单独存在。
## 二 select
#### 2.1 select的作用
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