## 一 micro简介

Go Micro是基于Golang的微服务开发框架，该框架解决了构建云本地系统的关键需求，提供了分布式系统开发需要的核心库，包含了RPC与事件驱动的通信机制。  

Go Micro隐藏了分布式系统的复杂性，将微服务体系内的技术转换为了一组工具集合，且符合可插拔的设计哲学，开发人员可以利用它快速构建系统组件，并能依据需求剥离默认实现并实现定制。  

Go Micro核心特性：
- 服务发现（Service Discovery）：自动服务注册与名称解析，默认的服务发现系统是Consul
- 负载均衡（Load Balancing）：在服务发现之上构建了负载均衡机制，对服务请求分发的均匀分布，并且在发生问题时进行重试
- 消息编码（Message Encoding：支持基于内容类型（content-type）动态编码消息，content-type默认包含proto-rpc和json-rpc。
- Request/Response：RPC通信基于支持双向流的请求/响应方式，提供有抽象的同步通信机制，默认的传输协议是http/1.1，而tls下使用http2协议。
- 异步消息（Async Messaging）：发布订阅（PubSub）等功能内置在异步通信与事件驱动架构中
- 可插拔接口（Pluggable Interfaces） - Go Micro为每个分布式系统抽象出接口。因此，Go Micro的接口都是可插拔的

## 二 micro安装  

安装依赖：
```
# consul下载 https://www.consul.io/downloads.html
consul agent -dev   # 开启consul监听即可

# protobuf下载
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
go get -u github.com/micro/protoc-gen-micro
```

安装 go micro：
```
go get -u github.com/micro/micro
cd $GOPATH/src/github.com/micro/micro
go build   -o micro  main.go
cp micro /usr/local/bin/
```

测试：
```
micro
```

##  二 helloworld

```
micro new ~/helloworld
cd ~/hellloworld
protoc --proto_path=. --micro_out=. --go_out=. proto/example/example.proto
go run main.go
```

## 三 详细命令

创建微服务命令：
```
new         # 创建 通过指定相对于$GOPATH的目录路径，创建一个新的微服务。

USAGE:
#用法
micro new [command options][arguments...]
--namespace     "go.micro" Namespace for the service e.g com.example #服务的命名空间
--type "srv"    Type of service e.g api, fnc, srv, web #服务类型
--fqdn          FQDN of service e.g com.example.srv.service (defaults tonamespace.type.alias)
--alias         Alias is the short name used as part of combined name if specified
                #   别名是在指定时作为组合名的一部分使用的短名称 run Run the micro runtime
run             Run the micro runtime # 运行 运行这个微服务时间
```

创建2个服务：
```
$micro new --type "srv" micro/rpc/srv
#"srv" 是表示当前创建的微服务类型 
#sss是相对于go/src下的文件夹名称 可以根据项目进行设置 
#srv是当前创建的微服务的文件名

Creating service go.micro.srv.srv in /home/itcast/go/src/micro/rpc/srv

.
#主函数
├── main.go
#插件
├── plugin.go #被调用函数
├── handler
│ └── example.go #订阅服务
├── subscriber
│ └── example.go #proto协议
├── proto/example
│ └── example.proto #docker生成文件
├── Dockerfile
├── Makefile
└── README.md


download protobuf for micro:

brew install protobuf
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
go get -u github.com/micro/protoc-gen-micro

compile the proto file example.proto:

cd /home/itcast/go/src/micro/rpc/srv
protoc --proto_path=. --go_out=. --micro_out=. proto/example/example.proto

#使用创建srv时给的protobuf命令保留用来将proto文件进行编译
micro new --type "web" micro/rpc/web
Creating service go.micro.web.web in /home/itcast/go/src/micro/rpc/web

.
#主函数
├── main.go
#插件文件
├── plugin.go
#被调用处理函数
├── handler
│ └── handler.go
#前端页面
├── html
│ └── index.html
#docker生成文件
├── Dockerfile
├── Makefile
└── README.md

#编译后将web端呼叫srv端的客户端连接内容修改为srv的内容 
#需要进行调通

```

启动consul进行监管:
```
consul agent -dev
```

对srv服务进行的操作:
```
#根据提示将proto文件生成为.go文件
cd /home/itcast/go/src/micro/rpc/srv
protoc --proto_path=. --go_out=. --micro_out=. proto/example/example.proto #如果报错就按照提示将包进行下载
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
go get -u github.com/micro/protoc-gen-micro
#如果还不行就把以前的包删掉从新下载
```



