## 一 Once 只执行一次

sync包提供了互斥锁、读写锁、条件变量等常见并发场景需要的API。sync还有一些其他API，如结构体：`sync.Once`，负责只执行一次，也即全局唯一操作。  

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

## 二 对象池 sync.Pool

`sync.Pool`可以作为临时值的容器，该容器具备自动伸缩、高效特性，同时也是并发安全的，其方法有：
- Get：从池中取出一个值，类型为`interface{}`
- Put：存储一个值到池中，存储的值类型为`interface{}`

使用示例：
```go
	p := &sync.Pool{
		New: func() interface{} {
			return 0
		},
	}

	a := p.Get().(int)
	p.Put(1)
	b := p.Get().(int)
	fmt.Println(a, b) // 0 1
```

注意：
- 如果池子从未Put过，其New字段也没有被赋值一个非nil值，那么Get方法返回结果一定是nil。  
- Get获取的值不一定存在于池中，如果Get到的值存在于池中，则该值Get后会被删除

对象池原理：
- 对象池可以把内部的值产生的压力分摊，即专门为每个操作它的协程关联的P建立本地池。Get方法被调用时，先从本地P对象的本地私有池和本地共享池获取值，如果获取失败，则从其他P的本地私有池偷取一个值返回，如果依然没有获取到，会依赖于当前对象池的值生成函数。注意：生产函数产生的值不会被放到对象池中，只是起到返回值作用
- 对象池的Put方法会把参数值存放到本地P的本地池中，每个P的本地共享池中的值，都是共享的，即随时可能会被偷走
- 对象池对垃圾回收友好，执行垃圾回收时，会将对象池中的对象值全部溢出

应用场景：sync.Pool的定位不是做类似连接池的东西，仅仅是增加对象重用的几率，减少gc的负担，而开销方面也不是很便宜的。   

案例：由于fmt包总是使用一些`[]byte`对象，可以为其建立了一个临时对象池，存放这些对象，需要的时候，从pool中取，拿不到则分配一份，这样就能避免一直生成`[]byte`，垃圾回收的效率也高了很多。   

示例：
```go
// 声明[]byte的对象池，每个对象为一个[]byte
var BytePool = sync.Pool{
	New: func() interface{} {
		b := make([]byte, 1024)
		return &b
	},
}

func main() {

	time1 := time.Now().Unix()

	// 不使用对象池创建 10000000
	obj1 := make([]byte, 1024)
	for i := 0; i < 100000000; i++ {
		obj1 = make([]byte, 1024)
		_ = obj1
	}

	time2 := time.Now().Unix()

	// 使用对象池创建 10010000
	obj2 := BytePool.Get().(*[]byte)
	for i := 0; i < 100000000; i++ {
		obj2 = BytePool.Get().(*[]byte)
		BytePool.Put(obj2)
		_ = obj2
	}

	time3 := time.Now().Unix()

	fmt.Println("without pool: ", time2-time1, "s") // 16s
	fmt.Println("with    pool: ", time3-time2, "s") // 1s
}
```