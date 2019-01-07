## 一 Go异常处理接口
#### 1.1 defer延迟执行修饰符
在函数中，程序员经常需要创建资源(比如:数据库连接、文件句柄、锁等) ，为了在函数执行完 毕后，及时的释放资源，Go 的设计者提供 defer (延时机制)。
```go
func main() {

	//当执行到defer语句时，暂不执行，会将defer后的语句压入到独立的栈中
	//当函数执行完毕后，再从该栈按照先入后出的方式出栈执行
	defer fmt.Println("defer1...")
	defer fmt.Println("defer2...")

	fmt.Println("main...")

}
```
上述代码执行结果：
```
main...
defer2...
defer1...
```
注意：在 defer 将语句放入到栈时，也会将相关的值拷贝同时入栈
```go
func main() {

	num := 0

	defer fmt.Println("defer中：num=", num)

	num = 3
	fmt.Println("main中：num=",num)

}
```
输出结果：
```
main中：num= 3
defer中：num= 0
```
defer最佳实践：用于关闭资源
```go
connect = openDatabase()
defer connect.close()
```
#### 1.2 error接口
Go引入了一个关于错误处理的标准模式，即error接口：
```
type error interface {
	Error() string
}

//相关方法
type errorString struct {
	texxt string
}
func New(text string) error {
	return &errorString(text)
}
func (e *errorString) Error() string{
	return e.text
}

//fmt的Errorf函数也可以生成error类型
```