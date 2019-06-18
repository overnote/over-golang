## 一 等待组 sync.WaitGroup

 `sync.WaitGroup`类型的值也是并发安全的，该类型结构体中内内部拥有一个计数器，计数器的值可以通过方法调用实现计数器的增加和减少 。  

当我们添加了 N 个并发任务进行工作时，就将等待组的计数器值增加 N。每个任务完成时，这个值减1。 同时，在另外一个 goroutine 中等待这个等待组的计数器值为 0 时， 表示所有任务己经完成。  

等待组常用方法：
- (wg *WaitGroup) Add(delta int)	等待组计数器+1，该方法也可以传入负值让等待计数减少，切记不能减少为负数，会引发崩溃
- (wg *WaitGroup) Done()			等待组计数器-1，等同于Add传入负值
- (wg *WaitGroup) Wait()			等待组计数器!=0时阻塞，直到为0

应用场景：WaitGroup一般用于协调多个goroutine运行。  

简单示例：
```go
package main

import (
	"fmt"
	"net/http"
	"sync"
)

func main() {

	var wg sync.WaitGroup					// 声明一个等待组

	var urls = []string{					
		"https://www.baidu.com/",
		"https://www.163.com/",
		"https://www.weibo.com/",
	}

	for _, url := range urls {
		wg.Add(1)						// 每个任务开始，等待组+1
		go func(url string) {
			defer wg.Done()
			_, err := http.Get(url)		// 执行访问
			fmt.Println(url, err)
		}(url)
	}

	wg.Wait()						// 等待所有任务完成
	fmt.Println("over")

}
```  

上述案例可以使用channel方式，每个go协程执行时，channel传递完成信号，但是使用通道的方式明显过重。  
