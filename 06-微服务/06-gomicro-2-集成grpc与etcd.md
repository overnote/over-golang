## 一 go micro 示例

### 1.0 说明

本章的 go micro 示例使用 etcd 为服务发现机制， grpc 为通信协议，并且基于go1.13版本，使用go mod管理包。  

项目目录结构：
```
|-microdemo
    |- common               通用文件夹
        |- code             项目状态码文件夹
            - commonCode.go
        |- config           项目通用配置文件夹   
            - commonConfig.go
    |- simple：             简单示例服务文件夹
        |- handler          简单示例服务的服务句柄文件夹
            |- testHandler
                - testHandler.go
        |- proto            简单示例服务的grpc协议文件夹
            |- testProto
                - test.micro.go
                - test.pb.go
                - test.proto
        - main.go           主文件
    |- web：                web服务文件夹
    |- go.mod               项目管理文件
```

贴士：本项目基于etcd，必须先启动etcd！！！

### 1.1 创建基础配置

创建一个名为 microdemo 的项目
```
go mod init microdemo           
```

commonCode.go：
```go
package code

type res struct {
	Code int
	Msg string
}

var OK *res
var SERERR *res
var DBERR *res
var INFOERR *res
var FILTERERR *res
var INFONOTFOUND *res

func init() {

	// 正确请求
	OK = &res{1000, "成功"}

	// 数据校验 3
	FILTERERR = &res{ 3001, "校验未通过", }

	// 资源状态 4
	INFONOTFOUND = &res{4001, "资源不存在",}

	// 服务器状态 5
	SERERR = &res{5001, "服务器错误",}
	DBERR = &res{5002, "数据库错误",}

}
```

commonConfig.go
```go
package config

import "fmt"

var ENV = "TEST"

var EtcdAddr []string = []string{
	"127.0.0.1:2379",
	"127.0.0.1:2379",
	"127.0.0.1:2379",
}

var RedisAddr string 	= "127.0.0.1"
var RedisPort string 	= "6379"
var RedisDB string 		= "0"

var FastDfsAddr string 	= "127.0.0.1"
var FastDfsPort string 	= "9090"

var StaticAddr string = "http://" + FastDfsAddr + ":" + FastDfsPort + "/"

func init() {

	if ENV == "PROD" {
		fmt.Println("执行生产环境配置")
	}

}
```

### 1.2 创建第一个微服务：simple

simple服务只是一个微服务的简单示例。  

生成协议文件：
```go
# simple/proto/simpleProto/simple.proto
syntax = "proto3";

package simpleProto;

service SimpleService {
	rpc SimpleFunc(SimpleRequest) returns (SimpleResponse) {}
}

message SimpleRequest {
    int32 id = 1;
}

message SimpleResponse {
    int32 code = 1;
	string msg = 2;
}

# 生成go协议文件
cd simple/proto/simpleProto
protoc simple.proto --proto_path=. --go_out=. --micro_out=.
```

书写句柄函数：即本服务具体做什么
```go
// simple/handler/simplerHandler/simplerHandler.go
package simpleHandler

import (
	"context"
	"microdemo/simple/proto/simpleProto"
)

// 简单微服务
type SimpleService struct{}
func (s *SimpleService) SimpleFunc(ctx context.Context, req *simpleProto.SimpleRequest, rsp *simpleProto.SimpleResponse) error {

	// 执行业务操作....

	// 返回业务数据给web服务
	rsp.Code = 1
	rsp.Msg = "成功"
	return nil
}
```

main文件：
```go
package main

import (
	"github.com/micro/go-micro"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/service/grpc"
	"github.com/micro/go-micro/util/log"
	"github.com/micro/go-plugins/registry/etcdv3"
	"microdemo/simple/handler/simpleHandler"
	"microdemo/simple/proto/simpleProto"
	"microdemo/common/config"
)

func main() {

	// 替换micro默认的服务发现框架consul为etcd
	reg := etcdv3.NewRegistry(func(op *registry.Options){
		op.Addrs = config.EtcdAddr
	})

	// 创建服务
	service := grpc.NewService(
		micro.Name("demo.srv.simple"),
		micro.Registry(reg),
		micro.Version("latest"),
		micro.Address(":" + "30066"),
	)
	service.Init()

	// 注册服务句柄
	err := simpleProto.RegisterSimpleServiceHandler(service.Server(), new(simpleHandler.SimpleService))
	if err != nil {
		log.Error("注册句柄错误：", err)
		return
	}

	// 运行服务
	if err := service.Run(); err != nil {
		log.Error("运行服务错误：", err)
		return
	}
}
```

### 1.3 创建第二个微服务：web

handler句柄文件：
```go
// web/handler/simpleHandler/simpleHandler.go
package simpleHandler

import (
	"context"
	"encoding/json"
	"fmt"
	"github.com/julienschmidt/httprouter"
	"github.com/micro/go-micro/service/grpc"
	"microdemo/simple/proto/simpleProto"
	"net/http"
)

// 简单微服务方法
func Simple(w http.ResponseWriter, r *http.Request, p httprouter.Params) {

	fmt.Println("参数：", p)

	//  grpc 服务初始化
	service := grpc.NewService()
	service.Init()

	// 获取服务句柄
	simpleClient := simpleProto.NewSimpleService("demo.srv.simple", service.Client())

	// 调用服务
	rsp, err := simpleClient.SimpleFunc(context.TODO(), &simpleProto.SimpleRequest{})
	if err != nil {
		fmt.Println("调用服务错误：", err)
		http.Error(w, err.Error(), 500)
	}

	// 创建返回给前端的数据
	result := map[string]interface{}{
		"code": 	rsp.Code,
		"msg": 		rsp.Msg,
	}
	if err := json.NewEncoder(w).Encode(result); err != nil {
		http.Error(w, err.Error(), 500)
	}
}
```

main.go
```go
package main

import (
        "github.com/julienschmidt/httprouter"
        "github.com/micro/go-micro/registry"
        "github.com/micro/go-micro/util/log"
        "github.com/micro/go-micro/web"
        "github.com/micro/go-plugins/registry/etcdv3"
        "microdemo/account/config"
        "microdemo/web/handler/simpleHandler"
        "net/http"
)

func main() {

        // 替换micro默认的服务发现框架consul为etcd
        reg := etcdv3.NewRegistry(func(op *registry.Options){
                op.Addrs = config.EtcdAddr
        })

        // 创建web服务
        service := web.NewService(
                web.Name("demo.web.web"),
                web.Registry(reg),
                web.Version("latest"),
                web.Address(":" + "3000"),
        )
        if err := service.Init(); err != nil {
                log.Error("服务初始化错误：", err)
        }

        // 创建路由
        router := httprouter.New()
        router.NotFound = http.FileServer(http.Dir("public"))

        // 测试路由
        router.GET("/simple/:id", simpleHandler.Simple)

        service.Handle("/", router)

        // 运行服务
        if err := service.Run(); err != nil {
                log.Error("服务运行错误：", err)
        }
}
```

### 1.4 服务启动与访问

启动etcd后，启动服务：
```
# 启动simple服务
cd simple
go run main.go --registry=etcd --registry_address=127.0.0.1:2379

# 启动web服务
cd web
go run main.go --registry=etcd --registry_address=127.0.0.1:2379
```

访问：
```
localhost:3000/simple/10001
```