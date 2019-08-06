## 只执行一次

sync包除了提供了互斥锁、读写锁、条件变量外，还提供了一个结构体：`sync.Once`，负责只执行一次，也即全局唯一操作。  

使用方式如下：
```go
var once sync.Once
once.Do(func(){})           // Do方法的有效调用次数永远是1
```

`sync.Once`的典型应用场景是只执行一次的任务，如果这样的任务不适合在init函数中执行，该结构体类就会派上用场。  

sync.Once内部使用了“卫述语句、双重检查锁定、共享标记的原子操作”来实现`Once`功能。  

once示例：
```go
package main

import (
	"fmt"
	"sync"
)

type Person struct {
	Name string
	Age int
}

func (p *Person)Grown() {
	p.Age += 1
}

func main() {

	var once sync.Once
	var wg sync.WaitGroup

	p := &Person{
		"比尔",
		0,
	}

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			once.Do(func(){
				p.Grown()
			})
			wg.Done()
		}()
	}

	wg.Wait()

	fmt.Println("年龄是：", p.Age)			// 只会输出 1

}
```