## 一 升级为grpc版本

重新生成Proto文件，srv的main.go
```go
package main
import (
    "github.com/micro/go-log"
    "github.com/micro/go-micro"
    "micro/grpc/srv/handler"
    "micro/grpc/srv/subscriber"
    example "micro/grpc/srv/proto/example"
    "github.com/micro/go-grpc"
)

func main() {

    // 创建新服务
    service := grpc.NewService( 
        //当前微服务的注册名 
        micro.Name("go.micro.srv.srv"), 
        //当前微服务的版本号 
        micro.Version("latest"),
    )

    // 初始化服务 
    service.Init()

    // Register Handler
    //通过protobuf的协议注册我们的handler
    //参数1是我们创建好的服务返回的句柄
    //参数2 使我们new的handler包中的类 
    example.RegisterExampleHandler(service.Server(), new(handler.Example))
    
    // Register Struct as Subscriber //注册结构体来自于Subscriber
    micro.RegisterSubscriber("go.micro.srv.srv", service.Server(),new(subscriber.Example))

    // Register Function as Subscriber
    // 注册函数自于Subscriber
    micro.RegisterSubscriber("go.micro.srv.srv", service.Server(),subscriber.Handler)


     // Run service
    if err := service.Run(); err != nil {
        log.Fatal(err)
    }


}
```

srv的example.go:
```go
package handler
import (
"context"
    "github.com/micro/go-log"
//更换了相关proto文件
    example "micro/grpc/srv/proto/example"
)

type Example struct{}

func (e *Example) Call(ctx context.Context, req *example.Request, rsp
*example.Response) error {
    log.Log("Received Example.Call request")
    rsp.Msg = "Hello " + req.Name
    return nil
}

//流数据的检测操作
func (e *Example) Stream(ctx context.Context, req *example.StreamingRequest, stream example.Example_StreamStream) error {
    log.Logf("Received Example.Stream request with count: %d", req.Count)
    for i := 0; i < int(req.Count); i++ {
        log.Logf("Responding: %d", i)
        if err := stream.Send(&example.StreamingResponse{
            Count: int64(i),
        }); err != nil {
            return err 
        }
    }

    return nil
}

//心跳检测机制
func (e *Example) PingPong(ctx context.Context, stream example.Example_PingPongStream) error {
    for {
        req, err := stream.Recv()
        if err != nil {
        } 
    }
}

```

修改web的main.go
```go
package main
import (
        "github.com/micro/go-log"
    "net/http"
        "github.com/micro/go-web"
        "micro/grpc/web/handler"
)

func main() {
    // create new web service
    service := web.NewService(
        web.Name("go.micro.web.web"),
        web.Version("latest"),
        web.Address(":8080"),
    )
    // initialise service
    if err := service.Init(); err != nil {
        log.Fatal(err)
    }
    // register html handler
    service.Handle("/", http.FileServer(http.Dir("html")))
    // register call handler
    service.HandleFunc("/example/call", handler.ExampleCall)
    // run service
    if err := service.Run(); err != nil {
        log.Fatal(err)
    } 

}

```

修改web的handler.go
```go
package handler
import (
    "context"
    "encoding/json"
    "net/http"
    "time"
    example "micro/grpc/srv/proto/example"
    "github.com/micro/go-grpc"
)

func ExampleCall(w http.ResponseWriter, r *http.Request) {
    server :=grpc.NewService()
    server.Init()
    // decode the incoming request as json
    var request map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    // call the backend service
    //exampleClient := example.NewExampleService("go.micro.srv.srv",
    client.DefaultClient)
    //通过grpc的方法创建服务连接返回1个句柄
    exampleClient := example.NewExampleService("go.micro.srv.srv", server.Client()) rsp, err := exampleClient.Call(context.TODO(), &example.Request{
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