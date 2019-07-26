## 一 Go操作consul

#### 1.1 注册服务 

搭建好conusl集群后，用户或者程序就能到consul中去查询或者注册服务。可以通过提供服务定义文件或者调用 HTTP API 来注册一个服务。  

启动consul开发模式：
```
consul agent -dev

# 访问可视化界面
http://127.0.0.1:8500/ui/dc1/nodes
```

#### 1.2 服务端

```go
package main

import (
	"fmt"
	consular "github.com/hashicorp/consul/api"
	"io"
	"log"
	"net"
	"net/http"
)

func registerServer() {
	config := consular.DefaultConfig()
	client, err := consular.NewClient(config)

	if err != nil {
		log.Fatal("consul client error: ", err)
		return
	}

	checkPort := 8080

	registration := new(consular.AgentServiceRegistration)
	registration.ID = "node01"
	registration.Name = "serverNode"
	registration.Port = 9527
	registration.Tags = []string{"serverNode"}
	registration.Address = "127.0.0.1"

	registration.Check = &consular.AgentServiceCheck{
		HTTP: fmt.Sprintf("http://%s:%d%s", registration.Address, checkPort, "/check"),
		Timeout: "3s",
		Interval: "5s",
		DeregisterCriticalServiceAfter: "30s", //check失败后30秒删除本服务
	}

	err = client.Agent().ServiceRegister(registration)

	if err != nil {
		log.Fatal("register server error : ", err)
		return
	}
	http.HandleFunc("/check", func (w http.ResponseWriter, r *http.Request) {
		fmt.Println( "consulCheck: ", w)
	})

	http.ListenAndServe(fmt.Sprintf(":%d", checkPort), nil)
}


func main() {

	// 注册服务
	go registerServer()

	// 生成一个服务：服务1
	ln, err := net.Listen("tcp", "0.0.0.0:9527")
	if err != nil {
		panic("error：" + err.Error())
	}
	for {
		conn, err := ln.Accept()
		if err != nil {
			panic("error：" + err.Error())
		}
		go EchoServer(conn)
	}

}

func EchoServer(conn net.Conn) {

	buf := make([]byte, 1024)
	defer conn.Close()

	for {
		n, err := conn.Read(buf)
		switch err {
		case nil:
			log.Println("get and echo:", "EchoServer "+string(buf[0:n]))
			conn.Write(append([]byte("EchoServer "), buf[0:n]...))
		case io.EOF:
			log.Printf("Warning: End of data: %s\n", err)
			return
		default:
			log.Printf("Error: Reading data: %s\n", err)
			return
		}
	}
}

```

#### 1.3 客户端

```go
package main

import (
	"fmt"
	"log"
	"net"
	"time"

	consular "github.com/hashicorp/consul/api"
)

func main() {

	client, err := consular.NewClient(consular.DefaultConfig())

	if err != nil {
		log.Fatal("consul client error : ", err)
	}

	for {

		time.Sleep(time.Second * 3)
		var services map[string]*consular.AgentService
		var err error

		services, err = client.Agent().Services()

		if nil != err {
			log.Println("in consual list Services:", err)
			continue
		}

		if _, found := services["node01"]; !found {
			log.Println("node01 not found")
			continue
		}

		sendData(services["node01"])

	}
}

func sendData(service *consular.AgentService) {
	conn, err := net.Dial("tcp", fmt.Sprintf("%s:%d", service.Address, service.Port))

	if err != nil {
		log.Println(err)
		return
	}

	defer conn.Close()

	buf := make([]byte, 1024)
	i := 0
	for {
		i++
		msg := fmt.Sprintf("Hello World, %03d", i)
		n, err := conn.Write([]byte(msg))
		if err != nil {
			println("Write Buffer Error:", err.Error())
			break
		}

		n, err = conn.Read(buf)
		if err != nil {
			println("Read Buffer Error:", err.Error())
			break
		}
		log.Println("get:", string(buf[0:n]))

		//等一秒钟
		time.Sleep(time.Second)
	}
}

```