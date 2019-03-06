## 一 Hello World
```go
package main


import(
	"fmt"
	"net/http"
)

func test1(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "test1")
}

func test2(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "test2")
}

func main() {

	server := http.Server{
		Addr: "127.0.0.1:8080",
	}

	http.HandleFunc("/test1", test1)

	http.HandleFunc("/test2", test2)

	server.ListenAndServe()

}
```

## 二 常见功能

#### 2.1 路由串联

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

