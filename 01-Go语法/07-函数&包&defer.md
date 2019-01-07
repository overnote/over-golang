## 一 函数
#### 1.1 函数声明
函数声明格式
```
func 函数名字 (参数列表) (返回值列表）{
	// do something
	return 返回值列表
}
```
注意：
- 函数名首字母小写为私有，大写为公有；
- 参数列表可以有0-多个，多参数使用逗号分隔，不支持默认参数；
- 返回值列表返回值类型可以不用写变量名
- 如果只有一个返回值且不声明类型，可以省略返回值列表与括号
- 有返回值，函数内必须有return
Go中函数常见写法：
```
//无返回值
func fn(){}  //此时默认返回0，所以也可以写为 func fn() int {}

//Go推荐给函数返回值起一个变量名
func fn1()(result int){
	return 1
}

//第二种返回值写法
func fn2()(result int){
	result = 1
	return 
}
```
多返回值情况下的Go函数：
```
func fn3 () (int, int, int) {
   return 1,2,3
}

//Go返回值推荐写法：
func fn4 () (a int, b int, c int) {
   a , b, c = 1, 2, 3
   return 
}

```
注意：多个参数类型如果相同，可以简写为： a,b int
#### 1.2 变量作用域
局部变量：函数内部声明，作用域仅限于函数内部
全局变量：函数外部声明，作用域为整个包，如果首字母大写，则整个程序都可使用。
注意：Go中，大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；小写字母开头的就是不可导出的，是私有变量。
#### 1.3 不定参数
不定参数必须是最后一个参数，只是一个语法糖，在内部机制上，不定参数是一个切片，即：[]type，这是因为参数args可以用for循环来获取每个传入的参数。 
```
//返回int类型的不定参数
func fn1(args ...int){
   fmt.Println(len(args))
}
//注意：所有不定参数的类型必须相同，且必须是最后一个参数，不定参数在函数内其实就相当于切片，对切片的操作同样适合不定参数。
```
#### 1.4 匿名函数
```
func main()  {
   a := 3;
   f1 := func() {    // f1 即为匿名函数
      fmt.Println(a) //匿名函数访问外部变量
   }
   f1()

   func() {         //匿名函数自调
      fmt.Println(a);
   }()
}
//匿名函数实战：取最大值,最小值
x, y := func(i,j int) (max,min int) {
   if i > j {
      max = i
      min = j
   } else {
      max = j
      min = i
   }
   return
}(10,20)
fmt.Println(x + ' ' + y)

```
#### 1.5 Go函数特性总结
支持有名称的返回值；
不支持默认值参数；
不支持重载；
不支持命名函数嵌套，匿名函数可以嵌套；
Go函数从实参到形参的传递永远是值拷贝，有时函数调用后实参指向的值发生了变化，是因为参数传递的是指针的拷贝，实参是一个指针变量，传递给形参的是这个指针变量的副本，实质上仍然是值拷贝；
Go函数支持不定参数；
#### 1.6 函数类型
函数类型：函数去掉函数名、参数名和{}后的结果，使用%T打印该结果。
两个函数类型相同的前提是：拥有相同的形参列表和返回值列表，且列表元素的次序、类型都相同，形参名可以不同。
定义了函数类型，就可以使用该类型进行传参。
```
func mathSum(a, b int) int {
	return a + b
}

func mathSub(a, b int) int {
	return a - b
}

//定义一个函数类型
type MyMath func(int, int) int

//定义的函数类型作为参数使用
func Test(f MyMath, a , b int) int{
	return f(a,b)
}
```
通常我们可以把函数类型当做一种引用类型，实际函数类型变量和函数名都可以当做指针变量，只想函数代码开始的位置，没有初始化的函数默认值是nil。
#### 1.7 匿名函数
匿名函数可以看做函数字面量，所有直接使用函数类型变量的地方都可以由匿名函数代替。匿名函数可以直接赋值给函数变量，可以当做实参，也可以作为返回值使用，还可以直接被调用。
```
package main

import "fmt"

//匿名函数1
var sumFunc = func(a, b int) int {
	return a + b
}

//匿名函数2
var subFunc = func(a, b int) int {
	return a - b
}

func Test (f func(int, int) int, a , b int) int {
	return f(a, b)
}

func wrap (op string) func(int, int) int {
	switch op {
	case "sum":
		return func(a, b int) int { return a + b }
	case "sub":
		return func(a, b int) int { return a + b }
	default:
		return nil
	}
}

func main() {

	//方式一：匿名函数被直接调用
	defer func(){
		if err := recover(); err != nil{
			fmt.Println(err)
		}
	}()

	//方式二：使用匿名函数变量名调用
	sumFunc(1, 2)

	//方式三：匿名函数作为实参
	Test(func(x, y int) int {
		return x + y
	},1 ,2)

	//方式四：
	myFunc := wrap("sum")
	result := myFunc(2,3)
	fmt.Println(result)

}
```
#### 1.8 init函数
Go语言中，除了可以在全局声明中初始化实体，也可以在init函数中初始化。init函数是一个特殊的函数，它会在包完成初始化后自动执行，执行优先级高于main函数，并且不能手动调用init函数，每一个文件有且仅有一个init函数，初始化过程会根据包的以来关系顺序单线程执行。
```go
package main
import (
	"fmt"
)
func init() {
	//在这里可以书写一些初始化操作
	fmt.Println("init...")
}
func main() {
	fmt.Println("main...")
}
```
## 二 包
在实际的开发中，我们往往需要在不同的文件中，去调用其它文件的定义的函数，比如 main.go 中，去使用 utils.go 文件中的函数，此时需要包的引入。  
比如需要使用"fmt"包中的Println()函数：
```go
import "fmt"
fmt.Println()
```
在Go中，Go的每一个文件都是属于一个包的，也就是说Go是以包的形式来管理文件和项目目录结构。所以如果要导入某些第三方包，直接输入包所在地址即可。
注意：
- 在给一个文件打包时，该包对应一个文件夹，比如这里的 utils 文件夹对应的包名就是 utils, 文件的包名通常和文件所在的文件夹名一致，一般为小写字母
- 当一个文件要使用其它包函数或变量时，需要先引入对应的包
```
引入方式 1:import "包名"
引入方式 2:
import (
	"包名"
	"包名" 
)
```
- package 指令在 文件第一行，然后是 import 指令
- 在 import 包时，路径从 $GOPATH 的 src 下开始，不用带 src , 编译器会自动从 src 下开始引入
- 为了让其它包的文件，可以访问到本包的函数，则该函数名的首字母需要大写，类似其它语言 的 public ,这样才能跨包访问
- 在访问其它包函数，变量时，其语法是 包名.函数名

## 三 闭包

#### 3.1 闭包案例

```go
func fn1(a int) func(i int) int {
	return func(i int) int {
		print(&a, a)
		return a
	}
}

func main() {

	f := fn1(1)			//输出地址
	g := fn1(2)			//输出地址
	
	fmt.Println(f(1))		//输出1
	fmt.Println(f(1))		//输出1

	fmt.Println(g(2))		//输出2
	fmt.Println(g(2))		//输出2

}
```

## 四 函数参数传递
#### 4.1 值传递和引用传递
不管是值传递还是引用传递，传递给函数的都是变量的副本，不同的是，值传递的是值的
拷贝，引用传递的是地址的拷贝，一般来说，地址拷贝效率高，因为数据量小，而值拷贝决定拷贝的 数据大小，数据越大，效率越低。
如果希望函数内的变量能修改函数外的变量，可以传入变量的地址&，函数内以指针的方式操作变量。
## 五 常用函数
#### 5.1 常用字符串函数
```go
//字符串的字节长度：汉字占用3个字节，字符占据1个字节
len("hello")

//字符串遍历：可以同时处理中文问题
r := []rune("hello北京")
fmt.Println(r[2])	//	查看第3个

//字符串转整数
n, err := strconv.Atoi("hello")

//整数转字符串
str = strconv.Itoa(12345)

//字符串转[]byte
var bytes = []byte("hello")

// []byte 转 字符串
 str = string([]byte{97, 98, 99})

//10进制转2 8 16进制
str = strconv.FormatInt(123, 2) // 2-> 8 , 16

//查找子字符串是否在指定的字符串中
 strings.Contains("seafood", "foo") //true

//统计一个字符串有几个指定的子串
strings.Count("ceheese", "e") //4

//不区分大小写的字符串比较(==是区分字母大小写的)
fmt.Println(strings.EqualFold("abc", "Abc")) // true

//返回子串在字符串第一次出现的 index 值，如果没有返回-1
strings.Index("NLT_abc", "abc") // 4

// 返回子串在字符串最后一次出现的 index，如没有返回-1
strings.LastIndex("go golang", "go")

// 将指定的子串替换成 另外一个子串
strings.Replace("go go hello", "go", "go 语言", n)  //n 可以指 定你希望替换几个，如果 n=-1 表示全部替换

// 按照指定的某个字符，为分割标识，将一个字符串拆分成字符串数组
strings.Split("hello,wrold,ok", ",")

// 将字符串的字母进行大小写的转换
strings.ToLower("Go") // go strings.ToUpper("Go") // GO

// 将字符串左右两边的空格去掉
strings.TrimSpace(" tn a lone gopher ntrn ")

// 将字符串左右两边指定的字符去掉 同样有 TrimLeft和 TrimRight
strings.Trim("! hello! ", " !")

// 判断字符串是否以指定的字符串开头
strings.HasPrefix("ftp://192.168.10.1", "ftp") // true

// 判断字符串是否以指定的字符串结束
strings.HasSuffix("NLT_abc.jpg", "abc") //false
```
#### 5.2 时间函数
```go
now := time.Now()

fmt.Printf(now.Format("2018-10-10 15:04:05"))
fmt.Printf(now.Format("2018-10-10"))
fmt.Printf(now.Format("15:04:05"))

//时间戳
now.Unix()		//10位 从1970年J 1 到现在的秒数
now.Unixnano()	//20位 同上，单位是纳秒
```