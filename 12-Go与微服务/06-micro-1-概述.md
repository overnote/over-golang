## 一 micro简介

Micro解决了构建云本地系统的关键需求。它采用了微服务体系结构模式，并将其转换为一组工具，作为可伸缩平 台的构建块。Micro隐藏了分布式系统的复杂性，并为开发人员提供了很好的理解概念。
Micro是一个专注于简化分布式系统开发的微服务生态系统。是一个工具集合, 通过将微服务架构抽象成一组工具。 隐藏了分布式系统的复杂性，为开发人员提供了更简洁的概念。  

## 二 micro安装  

下载：
```
$ go get -u -v github.com/go-log/log
$ go get -u -v github.com/gorilla/handlers
$ go get -u -v github.com/gorilla/mux
$ go get -u -v github.com/gorilla/websocket
$ go get -u -v github.com/mitchellh/hashstructure
$ go get -u -v github.com/nlopes/slack
$ go get -u -v github.com/pborman/uuid
$ go get -u -v github.com/pkg/errors
$ go get -u -v github.com/serenize/snaker
# hashicorp_consul.zip包解压在github.com/hashicorp/consul
$ unzip hashicorp_consul.zip -d github.com/hashicorp/consul # miekg_dns.zip 包解压在github.com/miekg/dns
$ unzip miekg_dns.zip -d github.com/miekg/dns
$ go get github.com/micro/micro
```

编译安装：
```
$ cd $GOPATH/src/github.com/micro/micro
$ go build   -o micro  main.go
$ sudo cp micro /bin/
```

插件安装：
```
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
go get -u github.com/micro/protoc-gen-micro
```

##  二 helloworld

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



