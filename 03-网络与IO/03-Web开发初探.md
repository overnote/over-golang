## 一 Hello World
```Go
package main

import (
	"fmt"
	"net/http"
)

func hello(res http.ResponseWriter, req *http.Request) {
	fmt.Fprintln(res, "hello world")
}

func main() {

	http.HandleFunc("/", hello)

	http.ListenAndServe(":3000", nil)

}
```