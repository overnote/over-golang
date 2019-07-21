## 一 go搭建 grpc helloworld

客户端：
```go
import (
    "log"
    "os"
    "golang.org/x/net/context"
    "google.golang.org/grpc"
    pb "google.golang.org/grpc/examples/helloworld/helloworld" //这是引用编译好的protobuf
)
const (
    address     = "localhost:50051"
    defaultName = "world"
)
func main() {
    // 建立到服务器的连接。
    conn, err := grpc.Dial(address, grpc.WithInsecure()) if err != nil {
            log.Fatalf("did not connect: %v", err)
        }
    //延迟关闭连接
    defer conn.Close() //调用protobuf的函数创建客户端连接句柄 c := pb.NewGreeterClient(conn)
    // 联系服务器并打印它的响应。 name := defaultName
    if len(os.Args) > 1 {
            name = os.Args[1]
    }
    //调用protobuf的sayhello函数
    r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: name}) if err != nil {
            log.Fatalf("could not greet: %v", err)
        }
    //打印结果
    log.Printf("Greeting: %s", r.Message)
}
```


服务端：
```go
package main
import (
    "log"
    "net"
    "golang.org/x/net/context"
    "google.golang.org/grpc"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
    "google.golang.org/grpc/reflection"
)
const (
    port = ":50051"
)
// 服务器用于实现helloworld.GreeterServer。 type server struct{}
// SayHello实现helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
    //监听
    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    //new服务对象
    s := grpc.NewServer()
    //注册服务
    pb.RegisterGreeterServer(s, &server{}) // 在gRPC服务器上注册反射服务。 reflection.Register(s)
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    } 
}
```

## 二 Go实现grpc远程调用

#### 2.1 protobuf协议定义

```
#到工作目录
$ CD $GOPATH/src/ #创建目录
$ mkdir grpc/myproto #进入目录
$ cd grpc/myproto #创建proto文件
$ vim helloServer.proto
```

文件内容：
```
syntax = "proto3";
package my_grpc_proto;
service HelloServer{ // 创建第一个接口
rpc SayHello(HelloRequest)returns(HelloReplay){} // 创建第二个接口
    rpc GetHelloMsg(HelloRequest)returns(HelloMessage){}
}
message HelloRequest{
    string name = 1 ;
}
message HelloReplay{
    string message = 1;
}
message HelloMessage{
    string msg = 1;
}
```
在当前文件下，编译 helloServer.proto文件：
```
$ protoc --go_out=./ *.proto #不加grpc插件
$ protoc --go_out=plugins=grpc:./ *.proto #添加grpc插件 #对比发现内容增加
#得到 helloServer.pb.go文件
```

#### 2.2 服务端

```go
package main
import (
    "net"
    "fmt"
    "google.golang.org/grpc"
    pt "demo/grpc/proto"
    "context"
)
const (
    post  = "127.0.0.1:18881"
)
//对象要和proto内定义的服务一样 type server struct{}
//实现RPC SayHello 接口
func(this *server)SayHello(ctx context.Context,in *pt.HelloRequest)(*pt.HelloReplay , error){
    return  &pt.HelloReplay{Message:"hello"+in.Name},nil
}
//实现RPC GetHelloMsg 接口
func (this *server) GetHelloMsg(ctx context.Context, in *pt.HelloRequest) (*pt.HelloMessage, error) {
    return &pt.HelloMessage{Msg: "this is from server HAHA!"}, nil
}

func main() {
    //监听网络
    ln ,err :=net.Listen("tcp",post)
    if err!=nil {
    fmt.Println("网络异常",err) }
    // 创建一个grpc的句柄
    srv:= grpc.NewServer()
    //将server结构体注册到 grpc服务中 pt.RegisterHelloServerServer(srv,&server{})
    //监听grpc服务
    err= srv.Serve(ln)
    if err!=nil { 
        fmt.Println("网络启动异常",err)
    }
}
```


#### 2.3 客户端

```go
package main
import (
    "google.golang.org/grpc"
    pt "demo/grpc/proto"
    "fmt"
    "context"
)
const (
    post  = "127.0.0.1:18881"
)

func main(){

    // 客户端连接服务器 conn,err:=grpc.Dial(post,grpc.WithInsecure()) if err!=nil {
    fmt.Println("连接服务器失败",err) }
    defer conn.Close() //获得grpc句柄
    c:=pt.NewHelloServerClient(conn) // 远程调用 SayHello接口
    //远程调用 SayHello接口
    r1, err := c.SayHello(context.Background(), &pt.HelloRequest{Name: "panda"}) 
    if err != nil {
        fmt.Println("cloud not get Hello server ..", err)
        return
    }
    fmt.Println("HelloServer resp: ", r1.Message)
    //远程调用 GetHelloMsg接口
    r2, err := c.GetHelloMsg(context.Background(), &pt.HelloRequest{Name: "panda"}) 
    if err != nil {
        fmt.Println("cloud not get hello msg ..", err)
        return
    }

     fmt.Println("HelloServer resp: ", r2.Msg)
}
```

#### 2.4 运行

```
#先运行 server，后运行 client
#得到以下输出结果
HelloServer resp: hellopanda
HelloServer resp: this is from server HAHA!
#如果反之则会报错
```