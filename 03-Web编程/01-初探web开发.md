## 一 Hello World

```go
package main

import(
	"fmt"
	"net/http"
)

func helloworld(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "helloworld")
}

func main() {

	server := http.Server{
		Addr: ":8080",
	}

	http.HandleFunc("/", helloworld)

	server.ListenAndServe()

}
```

web开发涉及的流程（下一章将会介绍各个流程中的技术细节）：
![](../images/Golang/web-01.png)

## 二 常见功能

#### 2.1 参数获取

在helloword案例中，对句柄函数做出如下修改，并访问：`localhost:8080/?id=3&name=zs`
```go
func helloworld(w http.ResponseWriter, r *http.Request) {

	// 默认不解析参数，所以这里必须设置
	parseErr := r.ParseForm()					
	if parseErr != nil {
		fmt.Println("参数解析错误：", parseErr)
		return
	}

	fmt.Println(r.Form)							// map[id:[3] name:[zs]]
	fmt.Println("path", r.URL.Path)				// /
	fmt.Println("scheme", r.URL.Scheme)			// scheme
	fmt.Println(r.Form["id"])					// [3]

	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("val:", strings.Join(v, ""))
	}

	_, fpError := fmt.Fprintf(w, "hellow world") 	//写入到客户端的信息
	if fpError != nil {
		fmt.Println("打印错误：", fpError)
	}
}
```

#### 2.2 路由与静态文件管理

虽然使用Go开发的web应用大多已经处于前后端分离的模式，这里还是需要提及一下静态文件的管理：
```go
	// 创建一个Go默认的路由
	mux := http.NewServeMux()
	// 静态文件管理：注意生成二进制文件时，public的路径，推荐使用绝对路径
	files := http.FileServer(http.Dir("./public"))
	mux.Handle("/static/", http.StripPrefix("/static/", files))
	mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request){
		fmt.Fprintf(w, "hello world")
	})
	server := &http.Server{
		Addr: "localhost:8080",
		Handler: mux,
	}
	server.ListenAndServe()
```

在本案例中，使用`http.FileServer(http.Dir("./public"))`创建了一个文件服务器，当访问地址以`/static/`开头时，路由器mux将移除该字符串，并在public目录中查找被请求的文件，如下所示：
```
# 访问地址
http://localhost:8080/static/html/hello.html
# 查找的文件为 public目录中
/html/hello.html
```

#### 2.3 路由串联

很多场景中，路由的处理函数在执行前，要先进行一些校验，比如安全检查，错误处理等等，这些行为需要在路由处理函数执行前有限执行。 

```go
package main


import(
	"fmt"
	"net/http"
)

func before(handle http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r * http.Request) {
		fmt.Println("执行前置处理")
		handle(w, r)
	}
}

func test(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "test1")
}


func main() {

	server := http.Server{
		Addr: "127.0.0.1:8080",
	}

	http.HandleFunc("/test", before(test))

	server.ListenAndServe()

}

```

