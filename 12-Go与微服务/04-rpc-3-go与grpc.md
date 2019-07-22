## 一 go搭建 grpc helloworld

#### 1.0 设置proto文件

创建文件：server/protoe/hello.proto    server是go mod的模块名
```
syntax = "proto3";
package protoes;

message Name {
    string name = 1;
}
message Msg1 {
    string message = 1;
}
message Msg2 {
    Msg1 message = 1;
}

//定义提供的服务
service ServeRoute{
    rpc Serve1(Name) returns (Msg1) {}
    rpc Serve2(Name) returns (Msg2) {}
}

```

在protoes文件所在的文件夹输入下面命令，生产pb.go文件:
```
protoc --go_out=plugins=grpc:. *.proto
```

#### 1.1 服务端

```go
package main

import (
    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "log"
    "net"
    pb "server/protoes"
)

//通过一个结构体，实现proto中定义的所有服务
type Hello struct{}

func (h Hello) Serve1(ctx context.Context, in *pb.Name) (*pb.Msg1, error) {
    log.Println("serve 1 works: get name: ", in.Name)
    resp := &pb.Msg1{Message:"this is serve 1"}
    return resp, nil
}

func (h Hello) Serve2(ctx context.Context, in *pb.Name) (*pb.Msg2, error) {
    log.Println("serve 2 works, get name: ", in.Name)
    resp := &pb.Msg2{
        Message:&pb.Msg1{Message:"this is serve 2"},
    }
    return resp, nil
}

func main() {

    // Address gRPC服务地址
    listen, err := net.Listen("tcp", "127.0.0.1:3001")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    s := grpc.NewServer()
    // 与http的注册路由类似，此处将所有服务注册到grpc服务器上，
    pb.RegisterServeRouteServer(s, Hello{})

    log.Println("grpc serve running")
    if err := s.Serve(listen); err != nil{
        log.Fatal(err)
    }
}

```

#### 1.2 客户端

注意：客户端目录中也存在着 protoes文件夹与对应文件

```go
package main
import (
	pb "client/protoes"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"log"
)

func main() {

	conn, err := grpc.Dial("127.0.0.1:3001", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()

	c := pb.NewServeRouteClient(conn)

	reqBody1 := &pb.Name{Name:"wang"}
	res1, err := c.Serve1(context.Background(), reqBody1)  //就像调用本地函数一样，通过serve1得到返回值
	if err != nil {
		log.Fatalln(err)
	}
	log.Println("message from serve: ", res1.Message)

	reqBody2 := &pb.Name{Name:"li"}
	res2, err := c.Serve2(context.Background(), reqBody2)  //就像调用本地函数一样，通过serve2得到返回值
	if err != nil {
		log.Fatalln(err)
	}
	log.Println("message from serve: ", res2.Message.Message)
}

```