## 一 Handler

在golang的web开发中，一个handler响应一个http请求：
```go
type Handler interface{
    ServerHTTP(ResponseWriter, *Request)
}
```

Handler可以有多种实现方式：
```go
// 实现一：HandlerFunc。
// HandlerFunc是对是用户定义的处理函数的包装，实现了ServeHTTP方法，在方法内执行HandlerFunc对象
type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}

// 实现二：ServeMux
// ServeMux是管理http请求pattern和handler的，实现了ServeHTTP方法，在其内根据请求匹配HandlerFunc，并执行其ServeHTTP方法
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r) // 调用的是HandlerFunc的ServeHTTP
}


// 实现三：serverHandler
// serverHandler是对Server的封装，实现了ServeHTTP方法，并在其内执行ServeMux的ServeHTTP方法
type serverHandler struct {
    srv *Server
}
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    ...
    handler.ServeHTTP(rw, req) // 调用的是ServerMux的ServeHTTP
}

```

## 二 ServeMux

ServeMux是一个HTTP请求多路复用器。它根据已注册模式（pattern）列表匹配每个传入请求的URL，并调用与URL最匹配的模式的处理程序（handler）。   

按照第一章的方式构建三个服务接口：
```go

    http.HandleFunc("/", mainHandler)
    http.HandleFunc("/hello/", helloHandler)
    http.HandleFunc("/world/", worldHandler)

	server := http.Server{
		Addr: ":8080",
    }
	server.ListenAndServe()
```

此时访问：
- localhost:8080/hello：会响应helloHandler函数
- localhost:8080/hello/：同样会响应helloHandler函数

实际上，访问`localhost:8080/hello`时，其实会以301重定向方式自动补齐为：`/hello/`，然后浏览器自动发起第二次请求。

如果使用ServeMux注册路由：
```go
    mux := http.NewServeMux()
    mux.HandleFunc("/", mainHandler)
    mux.HandleFunc("/hello", helloHandler)
    mux.HandleFunc("/world", worldHandler)

    server := http.Server{
        Addr: ":8080",
        Handler: mux,
    }
	server.ListenAndServe()
```
此时访问：
- localhost:8080/hello：会响应helloHandler函数
- localhost:8080/hello/：会响应mainHandler函数！！！

在使用ServeMux时：
- 如果pattern以"/"开头，表示匹配URL的路径部分
- 如果pattern不以"/"开头，表示从host开始匹配
- 匹配时长匹配优先于短匹配，注册在"/"上的pattern会被所有请求匹配，但其匹配长度最短
- 如果pattern带上了尾随斜线"/"，ServeMux将会对请求不带尾随斜线的URL进行301重定向。例如，在"/images/"模式上注册了一个handler，当请求的URL路径为"/images"时，将自动重定向为"/images/"，除非再单独为"/images"模式注册一个handler。
- 如果为"/images"注册了handler，当请求URL路径为"/images/"时，将无法匹配到

示例：
```
/test/          注册  handler1
/test/thumb/    注册  handler2

如果请求的url是：/test/thumb/，则调用handler2
如果请求的url是：/test/list/，则调用handler1
```

注意：其实当代码中不显式的创建serveMux对象，http包就默认创建一个DefaultServeMux对象用来做路由管理器mutilplexer。  

## 三 中间件

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

	http.HandleFunc("/", before(test))

	server := http.Server{
		Addr: "127.0.0.1:8080",
	}
	server.ListenAndServe()

}

```