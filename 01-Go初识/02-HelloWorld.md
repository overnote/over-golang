## 一 编写HelloWorld

新建一个helloworld.go文件，内容如下：
```go
package main                        //每个程序都有且仅有一个main包
import "fmt"                       
func main() {                       //主函数main只有一个
    fmt.Println("Hello World!")     //函数调用：包名.函数名
}
```

运行：
```
go run helloworld.go
```

## 二 Go 运行方式

```
# 执行方式一：先编译，再运行
go build hello.go        # 编译。在同级目录下生成文件`hello`，添加参数`-o 名称` 则可指定生成的文件名 
./hello                  # 运行。贴士：win下生成的是.exe文件，直接双击执行即可

# 执行方式二：直接运行
go run hello.go         
```

两种执行流程的区别：  
- 先编译方式：可执行文件可以在任意没有go环境的机器上运行，（因为go依赖被打包进了可执行文件）
- 直接执行方式：源码执行时，依赖于机器上的go环境，没有go环境无法直接运行

## 三 Go语法注意

- Go源文件以 "go" 为扩展名
- 与Java、C语言类似，Go应用程序的执行入口也是main()函数
- Go语言严格区分大小写
- Go不需要分号结尾
- Go编译是一行一行执行，所以不能将类似两个 Print函数写在一行
- Go语言定义的变量或者import的包如果没有使用到，代码不能编译通过
- Go的注释使用 // 或者 /*  */

## 四 Go版本管理工具

Go也有支持多版本管理的工具，例如：gvm(https://github.com/moovweb/gvm)   

使用方式如下：
```
# 安装gvm:
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)

# 安装、切换go版本
gvm install go1.8.3
gvm use go1.8.3         # 使用参数 --default，可以每次启动不需要调用use
```

## 五 开发工具推荐

笔者推荐的go开发工具：
- goland
- vscode

vscode的相关go插件会出现无法下载情况，解决办法：
```
# 1 在%GOPATH%\src\目录下，建立路径golang.org\x
# 2 进入到%GOPATH%\src\golang.org\x，下载需要工具的源码git clone https://github.com/golang/tools.git
# 3 进入到%GOPATH%下，执行
    go install github.com/ramya-rao-a/go-outline
    go install github.com/acroca/go-symbols
    go install golang.org/x/tools/cmd/guru
    go install golang.org/x/tools/cmd/gorename
    go install github.com/rogpeppe/godef
    go install github.com/sqs/goreturns
    go install github.com/cweill/gotests/gotests
# 4 单独处理golint,进入 %GOPATH%\src\github.com\ 后执行：
     git clone https://github.com/golang/lint
    进入到%GOPATH%下，执行：
     go install github.com/lint/golint
```