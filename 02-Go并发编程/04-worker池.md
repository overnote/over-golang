## 一 worker池
如果有一个任务就启用一个goroutine，那么会造成协程数量越来越多，利用生产者消费者模型，创建一个workder池，可以控制goroutine数量，防止goroutine泄露：

```go

```