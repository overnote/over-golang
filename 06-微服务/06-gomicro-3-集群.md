## 一 模拟三台集群

本章模拟微服务跨主机通信（直接使用上一章中的rpc也可行）：

```
# 以server模式运行agent，启动第一个节点node1：
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n1 -bind=192.168.186.128 -ui -config-dir /etc/consul.d -rejoin -join 192.168.186.128 -client 0.0.0.0

# 以server模式运行agent，启动第二个节点node1：
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n2 -bind=192.168.186.129 -ui -rejoin -join 192.168.186.128

# 以client模式运行cosnul agent，-join 加入到已有的集群中去
consul agent -data-dir /tmp/consul -node=n3 -bind=192.168.186.130 -config-dir /etc/consul.d -rejoin -join 192.168.186.128
```

## 二 启动服务

启动项目：
```
# 在第一台主机上启动微服务mysrv：
cd ~/go/src/test/mysrv
go run main.go

# 在第二台主机上启动微服务myweb：
cd ~/go/src/test/myweb
go run main.go
```

在第一台主机上创建json文件：/etc/consul.d/config.json：
```json
{
    "services":[
        {
          "name": "mysrv",
          "tags": [
              "srv"
          ],
          "address": "127.0.0.1",
          "port": 3000,
          "checks": [
            {
              "http":"http://localhost:3000/health",
              "interval":"10s"
            }
          ]
        },
        {
           "name": "myweb",
           "tags": [
               "web"
           ],
           "address": "127.0.0.1",
           "port": 8080,
           "checks": [
             {
               "http":"http://localhost:3000/health",
               "interval":"10s"
             }
           ]
         }
    ]
}
```

加载服务：
```
consul reload
```

此时在第二台主机上输入：localhost:8080，就可以开始操作了！


## 三 升级为grpc版本

mysrv的main中服务的创建方式修改为以下方式即可：
```go

    // "github.com/micro/go-micro/service/grpc"

    // 创建新服务
    service := grpc.NewService(                     // rpc版本是： micro.NewService()
        //当前微服务的注册名 
        micro.Name("go.micro.srv.srv"), 
        //当前微服务的版本号 
        micro.Version("latest"),
    )
```


myweb的handler中调用方式的改变：
```go
    // "github.com/micro/go-micro/service/grpc"

    server := grpc.NewService()
	server.Init()
	mysrvClient := mysrv.NewMysrvService("go.micro.srv.mysrv", server.Client())
```