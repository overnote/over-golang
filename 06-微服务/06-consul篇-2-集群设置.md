## 一 consul集群架设示例

#### 1.1 启动2个服务端

以server模式运行agent，启动第一个节点node1：
```
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n1 -bind=192.168.186.128 -ui -config-dir /etc/consul.d -rejoin -join 192.168.186.128 -client 0.0.0.0
```

以server模式运行agent，启动第二个节点node1：
```
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n2 -bind=192.168.186.129 -ui -rejoin -join 192.168.186.128
```

参数解释：
```
server：定义agent运行在server模式
bootstrap-expect：期望提供的server节点数，提供该值后，consul一直等到sever数目达到该值时才会引导整个集群，注意不能和bootstrap共用 
data-dir：存放agent状态的目录
node：节点在集群中的名称，必须是唯一的，默认是该节点的主机名 
bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
ui：启动web界面
config-dir：配置文件目录，里面所有以.json结尾的文件都会被加载 
rejoin:使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服 务，如果你要对外提供服务改成0.0.0.0
join：启动时加入这个哪个集群
```

#### 1.2 启动1个客户端

node3：
```
# 以client模式运行cosnul agent，-join 加入到已有的集群中去。
consul agent -data-dir /tmp/consul -node=n3 -bind=192.168.186.130 -config-dir /etc/consul.d -rejoin -join 192.168.186.128
```

#### 1.3 简单操作集群

```
# 查看集群成员：
consul members      # 需要在新开的终端窗口操作

# 停止Agent，退出集群
consul leave        # 可以优雅关闭，避免引起潜在的可用性故障影响达成一致性协议

# 查看所有结点
consul catalog nodes
```

使用Ctrl+C也可以关闭Agent，在退出中，Consul会提醒其他集群成员，这个节点离开了。如果强行杀掉进程，则集群的其他成员应该能检测到这个节点失效：
- 当一个成员离开，他的服务和检测也会从目录中移除
- 当一个成员失效了，他的健康状况被简单的标记为危险，但是不会从目录中移除
- Consul会自动尝试对失效的节点进行重连

当然也可以在web界面操作：http://127.0.0.1:8500/

## 二 集群服务的注册

搭建好conusl集群后，用户或者程序就能到consul中去查询或者注册服务。可以通过提供服务定义文件或者调用HTTP API来注册一个服务。  

操作步骤：
```
# 为consul配置一个目录，consul会载入配置目录中的所有配置文件
mkdir /etc/consul.d

# 假设有一个名为 mysrv 的服务，运行在3000端口，给该服务设置一个标签，以作为额外的查询方式，编写服务配置文件 mysrv.json：
{
  "service": {  
    "name": "mysrv",
    "tags": [
        "master"
    ],
    "address": "127.0.0.1",
    "port": 3000,
    "checks": [
      {
        "http":"http://localhost:3000/health",
        "interval":"10s"
      }
    ]
  }
}
```

测试程序：
```go
package main
import (
    "fmt"
    "net/http"
) 

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("hello mysrv! This is n3或者n2")
    fmt.Fprintf(w, "Hello mysrv! This is n3或者n2")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("health check! n3或者n2")
} 

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/health", healthHandler)
    http.ListenAndServe(":3000", nil)
}
```

运行测试程序，并更新consul：
```
consul reload     # 通过发送SIGHUP给agent来进行更新
```

## 三 查询服务

一旦agent启动并且服务同步了，我们可以通过DNS或者HTTP的API来查询服务。在consul的DNS API中，服务的DNS名字是 NAME.service.consul，虽然是可配置的，但默认的所有DNS名字会都在consul命名空间下，这个子域告诉Consul，我们在查询服务，NAME则是服务的名称。对于我们上面注册的Web服务，它的域名是 mysrv.service.consul :
```
dig @127.0.0.1 -p 8600 mysrv.service.consul
``` 

也可以使用 DNS API 来接收包含 地址和端口的 SRV记录：
```
dig @127.0.0.1 -p 8600 mysrv.service.consul SRV
```