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
#### 1.2 使用defer与recover处理错误
```go
func test(num1 int, num2 int){
	defer func(){
		err := recover()	//recover内置函数，可以捕获异常
		if err != nil {
			fmt.Println("err=", err);
		}
	}()
	fmt.Println(num1/num2)
}

func main() {

	test(2,0)

}
```
#### 1.3 自定义错误
Go 程序中，也支持自定义错误， 使用 errors.New 和 panic 内置函数。
- errors.New("错误说明") , 会返回一个 error 类型的值，表示一个错误
- panic 内置函数 ,接收一个 interface{}类型的值(也就是任何值了)作为参数。可以接收 error 类
   型的变量，输出错误信息，并退出程序
```go
//假定一个读取配置的函数，如果文件名不正确返回一个自定义错误
func readConfig(name string) (err error){
	if name == "conf" {
		//后续操作
		return nil
	} else {
		return errors.New("filename not correct ")
	}
}

func main() {

	err := readConfig("config")
	if err != nil {
		panic(err)
	}
	fmt.Println("继续执行...")

}
```