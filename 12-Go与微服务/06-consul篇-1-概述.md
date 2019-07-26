## 一 consul介绍 

#### 1.1 consul简介

Consul是HashiCorp公司推出的开源工具，用于实现分布式系统的服务发现与配置。 Consul是分布式、高可用、可横向扩展的。  

它具备以下特性：
- 服务发现（service discovery）：consul通过DNS或者HTTP接口使服务注册和服务发现变的很容易，一些外部服务，例如saas提供的也可以一样注册。
- 健康检查（health checking）：健康检测使consul可以快速的对集群中的操作发出警报。和服务发现的集成，可以防止服务转发到 故障的服务上面。
- 键值存储（key/value storage）：consul有一个用来存储动态配置的系统，并提供了简单的HTTP接口来操作
- 多数据中心（multi-datacenter）：无需复杂的配置，即可支持任意数量的数据区域。  

Consul是强一致性的数据存储，使用gossip形成动态集群。它提供分级键/值存储方式，不仅可以存储数据，而且可以用于注册器件事各种任务，从发送数据改变通知到运行健康检查和自定义命令，具体如何取决于它们的输出。  

与Zookeeper和etcd不一样，Consul内嵌实现了服务发现系统，所以这样就不需要构建自己的系统或使用第三方系统。这一发现系统除了上述提到的特性之外，还包括节点健康检查和运行在其上的服务。  

Zookeeper和etcd只提供原始的键/值队存储，要求应用程序开发人员构建他们自己的系统提供服务发现功能。而Consul提供了一个内置的服务发现的框架。客户只需要注册服务并通过DNS或HTTP接口执行服务发现。其他两个工具需要一个亲手制作的解决方案或借助于第三方工具。  

Consul为多种数据中心提供了开箱即用的原生支持，其中的gossip系统不仅可以工作在同一集群内部的各个节点，而且还可以跨数据中心工作。  

Consul还有另一个不错的区别于其他工具的功能，它不仅可以用来发现已部署的服务以及其驻留的节点信息，还通过HTTP请求、TTLs（time-to-live）和自定义命令提供了易于扩展的健康检查特性。  

#### 1.2 consul应用场景

consul的三个主要应用场景：服务发现、服务隔离、服务配置。  

服务发现场景中，consul作为注册中心，服务地址被注册到consul中以后，可以使用consul提供的dns、http接口查询，consul支持health check。   

服务隔离场景中，consul支持以服务为单位设置访问策略，能同时支持经典的平台和新兴的平台，支持tls证书分发，service-to-service加密。   

服务配置场景中，consul提供key-value数据存储功能，并且能将变动迅速地通知出去，通过工具consul-template可以更方便地实时渲染配置文件。   

可以通过Introduction to Consul了解consul的一些技术细节: 
```
每个被注册到consul中的node上，都部署一个consul agent，这个agent负责对本地的服务进行监控检查，以及将查询请求转发给consul server。
consul server负责存放、备份数据(使用raft协议保证一致性)，通常要有多台形成集群，选举出一个leader。
查询服务地址的时候，可以直接向consul server发起查询，也可以通过consul agent查询，后者将转发给consul server。
如果是多数据中心，每个数据中心部署一组consul server。跨数据中心查询通过本数据中心的consul server进行。
注意：多数据中心的时候，不同数据中心的consul server之间不会同步key-value数据。
```

![](../images/Golang/micro-03.png)  

注意上图中有两种类型的gossip，一类是同一个数据中心内部的client之间进行gossip通信，一类是不同数据中心的server之间进行gossip通信。  

## 二 consul安装

下载地址：https://www.consul.io/downloads.html

mac和linux版只需要将二进制文件移动到 `/usr/local/bin`目录中即可，测试安装结果：
```
consul
```

consul提供了开发模式用于启动单节点服务，用于开发调试：
```
consul agent -dev

# 访问可视化界面
http://127.0.0.1:8500/ui/dc1/nodes
```

## 三 常用命令

查看consul组成服务的node：
```
consul catalog nodes

# 查询结果
Node       ID        Address    DC
*******  e8c9f98c  127.0.0.1    dc1
```

还可以通过dns查询成员node地址，默认后缀是node.consul：
```
dig @127.0.0.1 -p 8600 127-0-0-1.node.consul
```

要注册的服务可以直接做成本地配置文件：
```
sudo mkdir /etc/consul.d
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}'  | sudo tee /etc/consul.d/web.json
```

或者通过api注册，api是：agent/service：
```
cat web2.json
{
    "Name": "web2",
    "Tags": [
        "rails"
    ],
    "Address": "",
    "Port": 81,
    "ServiceEnableTagOverride": false
}

curl --request PUT --data @web2.json  http://127.0.0.1:8500/v1/agent/service/register
```

查询所有服务：
```
consul catalog services
```

dns查询指定服务地址，默认后缀为service.consul
```
dig @127.0.0.1 -p 8600 web.service.consul srv
```

http api查询：
```
curl http://127.0.0.1:8500/v1/catalog/service/web |python -m json.tool
```

## 四 Connect配置

Connect是consul的重要特性，简单说就是，consul可以为服务配置访问代理，并且负责中间的认证和加密。  

在本地启动一个echo服务：
```
yum install -y socat
socat -v tcp-l:8181,fork exec:"/bin/cat"
```

注册到consul中，注意connect字段不为空，表示consul需要为socat服务准备代理:
```
cat <<EOF | sudo tee /etc/consul.d/socat.json
{
  "service": {
    "name": "socat",
    "port": 8181,
    "connect": { "proxy": {} }
  }
}
EOF
```

重启consul，或者给consul发送SIGHUB信号，重新加载配置。  

用下面的命令，手动在本地启动一个proxy：
```
consul connect proxy -service web -upstream socat:9191
```

然后就可以通过9191端口访问8181端口的服务：
```
nc 127.0.0.1 9191
helo
```

操作到这里的时候报错，通过9191无法联通，consul日志显示：
```
[WARN] agent: Check "service:socat-proxy" socket connection failed
```

## 五 key操作

写入一个名为”k1”的key，value为”hello”：
```
curl -X PUT --data "hello" 127.0.0.1:8500/v1/kv/k1
```

读取：
```
curl 127.0.0.1:8500/v1/kv/k1
```

key的读取接口支持6个参数：
```
key (string: "")         - Specifies the path of the key to read.
dc (string: "")          - Specifies the datacenter to query. 
                           This will default to the datacenter of the agent being queried. 
                           This is specified as part of the URL as a query parameter.
recurse (bool: false)    - Specifies if the lookup should be recursive and key treated as a prefix instead of a literal match. 
                           This is specified as part of the URL as a query parameter.
raw (bool: false)        - Specifies the response is just the raw value of the key, without any encoding or metadata. 
                           This is specified as part of the URL as a query parameter.
keys (bool: false)       - Specifies to return only keys (no values or metadata). Specifying this implies recurse. 
                           This is specified as part of the URL as a query parameter.
separator (string: '/')  - Specifies the character to use as a separator for recursive lookups. 
                           This is specified as part of the URL as a query parameter.
```

例如查看指定路径下的所有key：
```
curl 127.0.0.1:8500/v1/kv/k1?keys
[
    "k1",
    "k1/k11"
]
```

删除key：
```
curl -X DELETE 127.0.0.1:8500/v1/kv/k1
```