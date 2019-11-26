## 一 go micro简介

Go Micro是基于Golang的微服务开发框架，该框架解决了构建云本地系统的关键需求，提供了分布式系统开发需要的核心库，包含了RPC与事件驱动的通信机制。  

Go Micro隐藏了分布式系统的复杂性，将微服务体系内的技术转换为了一组工具集合，且符合可插拔的设计哲学，开发人员可以利用它快速构建系统组件，并能依据需求剥离默认实现并实现定制。  

Go Micro核心特性：
- 服务发现（Service Discovery）：自动服务注册与名称解析，默认的服务发现系统是Consul
- 负载均衡（Load Balancing）：在服务发现之上构建了负载均衡机制，对服务请求分发的均匀分布，并且在发生问题时进行重试
- 消息编码（Message Encoding：支持基于内容类型（content-type）动态编码消息，content-type默认包含proto-rpc和json-rpc。
- Request/Response：RPC通信基于支持双向流的请求/响应方式，提供有抽象的同步通信机制，默认的传输协议是http/1.1，而tls下使用http2协议。
- 异步消息（Async Messaging）：发布订阅（PubSub）等功能内置在异步通信与事件驱动架构中
- 可插拔接口（Pluggable Interfaces） - Go Micro为每个分布式系统抽象出接口。因此，Go Micro的接口都是可插拔的

## 二 go micro安装  

micro安装步骤：
```
# 安装依赖插件 protobuf
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
go get -u github.com/micro/protoc-gen-micro

# 安装 micro库，该库用于生成micro命令，micro命令可以用来快速生成基于go-micro的项目：
go get -u -v github.com/micro/micro
cd $GOPATH
go install github.com/micro/micro                    

# 测试
micro
```

## 三 micro 的 new 和 run 命令

```
micro new   # 相对于$GOPATH创建一个新的微服务
            # 参数 --namespace "test"   服务的命名空间
            # 参数 --type "srv"         服务类型，常用的有 srv api web fnc
            # 参数 --fqdn               服务正式的全定义
            # 参数 --alias              别名是在指定时作为组合名的一部分使用的短名称

micro run   # 运行这个微服务
```

注意：new默认创建的项目是以rpc为通信协议、mdns为服务发现的，基本不具备生产价值。笔者在下一章使用 go micro 手动创建项目，集成了grpc、etcd。  

## 三 hello world

hello.proto：
```
syntax = "proto3";
 
service Greeter {
	rpc Hello(HelloRequest) returns (HelloResponse) {}
}
message HelloRequest {
	string name = 1;
}
 
message HelloResponse {
	string greeting = 2;
}
```

生成proto的go文件：
```
protoc --proto_path=$GOPATH/src:. --micro_out=. --go_out=. hello.proto
```

服务端：
```go
package main
 
import (
	"context"
	"fmt"
 
	micro "github.com/micro/go-micro"
	proto "mygoproject/gomirco" //这里写你的proto文件放置路劲
)
 
type Greeter struct{}
 
func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
	rsp.Greeting = "Hello " + req.Name
	return nil
}
 
func main() {
	// Create a new service. Optionally include some options here.
	service := micro.NewService(
		micro.Name("greeter"),
	)
 
	// Init will parse the command line flags.
	service.Init()
 
	// Register handler
	proto.RegisterGreeterHandler(service.Server(), new(Greeter))
 
	// Run the server
	if err := service.Run(); err != nil {
		fmt.Println(err)
	}
}
```

客户端：
```go
package main
 
import (
	"context"
	"fmt"
 
	micro "github.com/micro/go-micro"
	proto "mygoproject/gomirco" //这里写你的proto文件放置路劲
)
 
 
func main() {
	// Create a new service. Optionally include some options here.
	service := micro.NewService(micro.Name("greeter.client"))
	service.Init()
 
	// Create new greeter client
	greeter := proto.NewGreeterService("greeter", service.Client())
 
	// Call the greeter
	rsp, err := greeter.Hello(context.TODO(), &proto.HelloRequest{Name: "John"})
	if err != nil {
		fmt.Println(err)
	}
 
	// Print response
	fmt.Println(rsp.Greeting)
}
```