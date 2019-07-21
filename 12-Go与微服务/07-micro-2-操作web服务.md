## 一 操作web

#### 1.1 main文件

```go
package main
import (
    "github.com/micro/go-log"
    "net/http"
    "github.com/micro/go-web"
    "micro/rpc/web/handler"
)

func main() {
    // 创建1个web服务
    service := web.NewService( //注册服务名
        web.Name("go.micro.web.web"), //服务的版本号
        web.Version("latest"), //!添加端口 web.Address(":8080"),
    )
    //服务进行初始化
    if err := service.Init(); err != nil {
                    log.Fatal(err)
    }
    //处理请求 / 的路由 //当前这个web微服务的 html文件进行映射 service.Handle("/", http.FileServer(http.Dir("html")))

    //处理请求 /example/call 的路由 这个相应函数 在当前项目下的handler service.HandleFunc("/example/call", handler.ExampleCall)
    //运行服务
    if err := service.Run(); err != nil {
    }

}
```

#### 1.2 handler文件

将准备好的html文件替换掉原有的文件

```go
package handler
import (
    "context"
    "encoding/json"
    "net/http"
    "time"
    "github.com/micro/go-micro/client"
//将srv中的proto的文件导入进来进行通信的使用
    example "micro/rpc/srv/proto/example"
)

//相应请求的业务函数
func ExampleCall(w http.ResponseWriter, r *http.Request) {


    // 将传入的请求解码为json
    var request map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }

    // 调用服务
    //替换掉原有的服务名
    //通过服务名和
    exampleClient := example.NewExampleService("go.micro.srv.srv",
    client.DefaultClient)

    rsp, err := exampleClient.Call(context.TODO(), &example.Request{
            Name: request["name"].(string),
    })

    if err != nil {
            http.Error(w, err.Error(), 500)
            return
    }

     // we want to augment the response
    response := map[string]interface{}{
        "msg": rsp.Msg,
         "ref": time.Now().UnixNano(),
    }

     // encode and write the response as json
    if err := json.NewEncoder(w).Encode(response); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
}

```

