## 一 准备环境

使用Docker方式搭建一个有3个 Server 节点和1个 Client 节点的 Consul 集群。
```
# 这是第一个 Consul 容器，其启动后的 IP 为172.17.0.5
docker run -d --name=c1 -p 8500:8500 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 -ui

docker run -d --name=c2 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.5

docker run -d --name=c3 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.5

#下面是启动 Client 节点
docker run -d --name=c4 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=false --client=0.0.0.0 --join 172.17.0.5

# 查看集群
consul members
```

## 二 代码实践 

#### 2.1 proto文件

假设现在有一个用 Node.js 写的服务 node-server 需要通过 gRPC 的方式调用一个用 Go 写的服务 go-server。  

下面是用 Protobuf 定义的服务和数据类型文件 hello.proto。  

```
syntax = "proto3";

package hello;
option go_package = "hello";

// The greeter service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (stream HelloRequest) returns (stream HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

用命令通过 Protobuf 的定义生成 Go 语言的代码：protoc --go_out=plugins=grpc:./hello ./*.proto 会在 hello 目录下得到 hello.pb.go 文件，然后在 hello.go 文件中实现我们定义的 RPC 服务。


#### 2.2 GoServer

在hello.go文件中实现RPC服务：
```go
package hello

import "context"

type GreeterServerImpl struct {}

func (g *GreeterServerImpl) SayHello(c context.Context, h *HelloRequest) (*HelloReply, error)  {
    result := &HelloReply{
        Message: "hello" + h.GetName(),
    }
    return result, nil
}
```

main.go：主要是将我们定义的服务注册到 gRPC 中，并建了一个 /ping 接口用于之后 Consul 的健康检查。
```go
package main

import (
    "go-server/hello"
    "google.golang.org/grpc"
    "net"
    "net/http"
)

func main() {
    lis1, _ := net.Listen("tcp", ":8888")
    lis2, _ := net.Listen("tcp", ":8889")
    grpcServer := grpc.NewServer()
    hello.RegisterGreeterServer(grpcServer, &hello.GreeterServerImpl{})
    go grpcServer.Serve(lis1)
    go grpcServer.Serve(lis2)
    
    http.HandleFunc("/ping", func(res http.ResponseWriter, req *http.Request){
        res.Write([]byte("pong"))
    })
    http.ListenAndServe(":8080", nil)
}
```

至此 go-server 端的代码就全部编写完了，可以看出代码里面没有任何涉及到 Consul 的地方，用 Consul 做服务注册是可以做到对项目代码没有任何侵入性的。

#### 2.3 将 go-server 注册到 Consul 中

将服务注册到 Consul 可以通过直接调用 Consul 提供的 REST API 进行注册，还有一种对项目没有侵入的配置文件进行注册。Consul 服务配置文件的详细内容可以在此查看。下面是我们通过配置文件进行服务注册的配置文件 services.json：
```js
{
  "services": [
    {
      "id": "hello1",
      "name": "hello",
      "tags": [
        "primary"
      ],
      "address": "172.17.0.9",
      "port": 8888,
      "checks": [
        {
          "http": "http://172.17.0.9:8080/ping",
          "tls_skip_verify": false,
          "method": "GET",
          "interval": "10s",
          "timeout": "1s"
        }
      ]
    },{
      "id": "hello2",
      "name": "hello",
      "tags": [
        "second"
      ],
      "address": "172.17.0.9",
      "port": 8889,
      "checks": [
        {
          "http": "http://172.17.0.9:8080/ping",
          "tls_skip_verify": false,
          "method": "GET",
          "interval": "10s",
          "timeout": "1s"
        }
      ]
    }
  ]
}
```

配置文件中的 172.17.0.9 代表的是 go-server 所在服务器的 IP 地址，port 就是服务监听的不同端口，check 部分定义的就是健康检查，Consul 会每隔 10秒钟请求一下 /ping 接口以此来判断服务是否健康。将这个配置文件复制到 c4 容器的 /consul/config 目录，然后执行consul reload 命令后配置文件中的 hello 服务就注册到 Consul 中去了。通过在宿主机执行curl http://localhost:8500/v1/catalog/services\?pretty就能看到我们注册的 hello 服务。

#### 2.3 NodeServer

```js
const grpc = require('grpc');
const axios = require('axios');
const protoLoader = require('@grpc/proto-loader');
const packageDefinition = protoLoader.loadSync(
  './hello.proto',
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
const hello_proto = grpc.loadPackageDefinition(packageDefinition).hello;

function getRandNum (min, max) {
  min = Math.ceil(min);
  max = Math.floor(max);
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

const urls = []
async function getUrl() {
  if (urls.length) return urls[getRandNum(0, urls.length-1)];
  const { data } = await axios.get('http://172.17.0.5:8500/v1/health/service/hello');
  for (const item of data) {
    for (const check of item.Checks) {
      if (check.ServiceName === 'hello' && check.Status === 'passing') {
        urls.push(`${item.Service.Address}:${item.Service.Port}`)
      }
    }
  }
  return urls[getRandNum(0, urls.length - 1)];
}

async function main() {
  const url = await getUrl();
  const client = new hello_proto.Greeter(url, grpc.credentials.createInsecure());
   
  client.sayHello({name: 'jack'}, function (err, response) {
    console.log('Greeting:', response.message);
  }); 
}

main()
```

代码中 172.17.0.5 地址为 c1 容器的 IP 地址，node-server 项目中直接通过 Consul 提供的 API 获得了 hello 服务的地址，拿到服务后我们需要过滤出健康的服务的地址，再随机从所有获得的地址中选择一个进行调用。代码中没有做对 Consul 的监听，监听的实现可以通过不断的轮询上面的那个 API 过滤出健康服务的地址去更新 urls 数组来做到。现在启动 node-server 就可以调用到 go-server 服务。