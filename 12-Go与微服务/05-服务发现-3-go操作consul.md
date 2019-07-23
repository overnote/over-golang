## 一 Go操作consul

#### 1.1 注册服务 

搭建好conusl集群后，用户或者程序就能到consul中去查询或者注册服务。可以通过提供服务定义文件或者调用 HTTP API 来注册一个服务.  

首先，为Consul配置创建一个目录，Consul会载入配置文件夹里的所有配置文件：
```
mkdir /etc/consul.d     # .d 后缀意思是这个路径包含了一组配置文件
```

然后，编写服务定义配置文件：
```
{
    "service": {
        "name": "web",                                      # 名称 
        "tags": ["master"],                                 # 标记，额外标签可以迎来作为额外的查询方式 
        "address": "127.0.0.1",                             # ip 
        "port": 10000,                                      # 端口 
        "checks": [
            {
                "http": "http://localhost:10000/health", 
                "interval": "10s"                           #检查时间
            } 
        ]

    }
}
```

#### 1.2 测试程序 

```go

package main
import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("hello Web3! This is n3或者n2")
    fmt.Fprintf(w, "Hello Web3! This is n3或者n2") 
}

func healthHandler(w http.ResponseWriter, r *http.Request) { 
    fmt.Println("health check! n3或者n2")
}

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/health", healthHandler)
    http.ListenAndServe(":10000", nil)
}
```

#### 1.3 查询服务

一旦agent启动并且服务同步了，我们可以通过DNS或者HTTP的API来查询服务。


首先使用DNS API来查询。在DNS API中，服务的DNS名字是 NAME.service.consul. 虽然是可配置的，但默认所有的DNS名字会都在consul命名空间下，这个子域告诉Consul正在查询服务，NAME则是服务的名称。  

对于上面注册的Web服务，它的域名是 web.service.consul :
```
dig @127.0.0.1 -p 8600 web.service.consul
```

也可用使用 DNS API 来接收包含 地址和端口的 SRV记录:
```
dig @127.0.0.1 -p 8600 web.service.consul SRV
```
