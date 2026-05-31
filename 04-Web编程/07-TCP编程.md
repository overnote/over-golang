## 一 TCP编程

### 1.1 服务端代码

```go
package main

import (
	"fmt"
	"net"
)

func main() {

	/**
	Unix网络编程步骤：Server->Bind->Listen->Accept
	Go语言简化为了：Listen->Accept
	 */

	// 此处创建了第一个套接字：设置了通信协议、IP地址、port
	listener, err := net.Listen("tcp", "127.0.0.1:3000")
	defer listener.Close()			// 套接字也是文件，需要关闭
	if err != nil {
		fmt.Println("net listen err:", err)
		return
	}

	// 此处创建了第二个套接字：用于阻塞监听客户端连接请求。注意listener并未监听，accept实现了监听
	conn, err := listener.Accept()
	defer conn.Close()			// 套接字也是文件，需要关闭
	if err != nil {
		fmt.Println("listener accept err:", err)
		return
	}

	// 读取客户端数据
	buf := make([]byte, 4096)
	n, err := conn.Read(buf)
	if err != nil {
		fmt.Println("conn.Read err:", err)
		return
	}

	// 业务逻辑
	fmt.Println("Read msg:", string(buf[:n]))
	conn.Write([]byte("word"))

}
```

运行服务端后，使用命令行工具模拟请求：`nc 127.0.0.1 3000`。

### 1.2 客户端代码

在1.1中，只是使用nc命令模拟了客户端，下面直接使用Go开发一个TCP客户端：
```go
package main

import (
	"fmt"
	"net"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:3000")
	if err != nil {
		fmt.Println("Dial err:", err)
		return
	}
	defer conn.Close()

	// 主动向服务器发送数据
	conn.Write([]byte("hello"))

	// 接收服务器返回数据
	buf := make([]byte, 4096)
	n, err := conn.Read(buf)
	if err != nil {
		fmt.Println("client read err:", err)
		return
	}
	fmt.Println("client receive：", string(buf[:n]))
}
```

客户端运行后，即可与TCP服务端进行通信。  

### 1.3 优化服务端

在1.1中，服务端接收了一次请求后即关闭了，显然不符合现在服务端能够同时接收大量请求的业务要求。使用for循环不断创建连接，等待带新的请求即可实现多客户端接入。具体的业务逻辑则可以交给一个go协程处理，这样服务端就可以专门用于循环等待请求、创建连接，而每个go程则负责具体的业务逻辑！    

并发服务端：
```go
package main

import (
	"fmt"
	"io"
	"net"
)

func main() {

	// 创建监听套接字
	listener, err := net.Listen("tcp", "127.0.0.1:3000")
	defer listener.Close()
	if err != nil {
		fmt.Println("net listen err:", err)
		return
	}

	// 监听客户端连接请求
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("listener accept err:", err)
			return
		}

		// 业务逻辑
		go handler(conn)
	}
}

func handler(conn net.Conn) {

	if conn == nil {
		panic("conn is nil")
	}
	defer conn.Close()

	// 循环读取客户端数据
	buf := make([]byte, 4096)
	for {
		n, err := conn.Read(buf)

		if err == io.EOF {		// 此时n=0
			fmt.Println("read EOF")
			break
		}

		if err != nil {
			fmt.Println("conn.Read err:", err)
			break
		}

		fmt.Println("Server receive msg:", string(buf[:n]))
		conn.Write([]byte("word"))
	}
}
```

## 二 理解Golang TCP编程

### 2.1 IPv4与IPv6

目前的全球因特网所采用的协议族是TCP/IP协议。IP是TCP/IP协议中网络层的协议，是TCP/IP协议族的核心协议。目前主要采用的IP协议的版本号是4(简称为IPv4)。  

IPv4的地址位数为32位，也就是最多有2的32次方的网络设备可以联到Internet上。近十年来由于互联网的蓬勃发展，IP位址的需求量愈来愈大，使得IP位址的发放愈趋紧张，前一段时间，据报道IPV4的地址已经发放完毕。  

IPv4地址格式类似这样：`127.0.0.1` `171.121.121.111`

IPv6是下一版本的互联网协议，也可以说是下一代互联网的协议，它是为了解决IPv4在实施过程中遇到的各种问题而被提出的，IPv6采用128位地址长度，几乎可以不受限制地提供地址。按保守方法估算IPv6实际可分配的地址，整个地球的每平方米面积上仍可分配1000多个地址。在IPv6的设计过程中除了一劳永逸地解决了地址短缺问题以外，还考虑了在IPv4中解决不好的其它问题，主要有端到端IP连接、服务质量（QoS）、安全性、多播、移动性、即插即用等。

地址格式类似这样：2008:c0e8:82e7:0:0:0:c7e8:82e7  

Go中提供了`ParseIP(s string) IP`函数会把一个IPv4或者IPv6的地址转化成IP类型。



大部分底层网络编程都依赖于Socket编程，包括：HTTP，IM通信，视频流传输，游戏服务器等。因为对于HTTP协议来说，直接使用Socket编程能够节省性能开支。  
Socket起源于UNIX，本着UNIX一切皆文件的哲学，可以用`打开-读写-关闭`的方式操作。网络的Socket数据传输是一种特殊的I/O，Socket也是一种文件描述符。Socket也具有一个类似于打开文件的函数调用：`Socket()`，该函数返回一个整型的Socket描述符，随后的连接建立、数据传输等操作都是通过该Socket实现的。  
网络之间的进程如果要通信，需要先对socket进行唯一标识。在本地，网络之间通信可以通过`PID`来标识唯一，但是到了网络中，进程通过网络层的`IP`，传输层的`协议+端口`来标识（三元组：ip地址，协议，端口可以标识网络的唯一进程）。  

Web开发中，Socket编程主要面向OSI模型的第三层和第四层协议，即：IP协议，TCP协议，UDP协议，常见的分类有：
- 流式Socket（SOCK_STREAM）：面向连接，主要用于TCP服务
- 数据式Socket（SOCK_DGRAM）：无连接，主要用于UDP服务

## 三 UDP

Go语言包中处理UDP Socket和TCP Socket不同的地方就是在服务器端处理多个客户端请求数据包的方式不同,UDP缺少了对客户端连接请求的Accept函数。其他基本几乎一模一样，只有TCP换成了UDP而已。UDP的几个主要函数如下所示：
```Go

func ResolveUDPAddr(net, addr string) (*UDPAddr, os.Error)
func DialUDP(net string, laddr, raddr *UDPAddr) (c *UDPConn, err os.Error)
func ListenUDP(net string, laddr *UDPAddr) (c *UDPConn, err os.Error)
func (c *UDPConn) ReadFromUDP(b []byte) (n int, addr *UDPAddr, err os.Error)
func (c *UDPConn) WriteToUDP(b []byte, addr *UDPAddr) (n int, err os.Error)
```
一个UDP的客户端代码如下所示,我们可以看到不同的就是TCP换成了UDP而已：
```Go
func main() {
	if len(os.Args) != 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s host:port", os.Args[0])
		os.Exit(1)
	}
	service := os.Args[1]
	udpAddr, err := net.ResolveUDPAddr("udp4", service)
	checkError(err)
	conn, err := net.DialUDP("udp", nil, udpAddr)
	checkError(err)
	_, err = conn.Write([]byte("anything"))
	checkError(err)
	var buf [512]byte
	n, err := conn.Read(buf[0:])
	checkError(err)
	fmt.Println(string(buf[0:n]))
	os.Exit(0)
}
func checkError(err error) {
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error %s", err.Error())
		os.Exit(1)
	}
}

```
我们来看一下UDP服务器端如何来处理：
```Go
func main() {
	service := ":1200"
	udpAddr, err := net.ResolveUDPAddr("udp4", service)
	checkError(err)
	conn, err := net.ListenUDP("udp", udpAddr)
	checkError(err)
	for {
		handleClient(conn)
	}
}
func handleClient(conn *net.UDPConn) {
	var buf [512]byte
	_, addr, err := conn.ReadFromUDP(buf[0:])
	if err != nil {
		return
	}
	daytime := time.Now().String()
	conn.WriteToUDP([]byte(daytime), addr)
}
func checkError(err error) {
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error %s", err.Error())
		os.Exit(1)
	}
}

```





#### 1.3 Go的TCP响应解释

Go语言通过net包中的DialTCP函数来建立一个TCP连接，并返回一个TCPConn类型的对象，当连接建立时服务器也创建一个同类型的对象，此时客户端和服务端通过各自拥有的TCPConn对象进行数据交换，只有当任意一端关闭连接，才会失效。

`TCPConn`类型拥有的函数有：
```go
func (c *TCPConn) Write(b []byte) (int, error)
func (c *TCPConn) Read(b []byte) (int, error)
```

`TCPAddr`类型表示一个TCP的地址信息:
```go
type TCPAddr struct {
	IP IP
	Port int
	Zone string // IPv6 scoped addressing zone
}
```
在Go语言中通过`ResolveTCPAddr`获取一个`TCPAddr`

```Go

func ResolveTCPAddr(net, addr string) (*TCPAddr, os.Error)

// net参数是"tcp4"、"tcp6"、"tcp"中的任意一个，分别表示TCP(IPv4-only), TCP(IPv6-only)或者TCP(IPv4, IPv6的任意一个)。
// addr表示域名或者IP地址，例如"www.google.com:80" 或者"127.0.0.1:22"。
```

Go语言中通过net包中的`DialTCP`函数来建立一个TCP连接，并返回一个`TCPConn`类型的对象，当连接建立时服务器端也创建一个同类型的对象，此时客户端和服务器端通过各自拥有的`TCPConn`对象来进行数据交换。一般而言，客户端通过`TCPConn`对象将请求信息发送到服务器端，读取服务器端响应的信息。服务器端读取并解析来自客户端的请求，并返回应答信息，这个连接只有当任一端关闭了连接之后才失效，不然这连接可以一直在使用。建立连接的函数定义如下：

```Go
func DialTCP(network string, laddr, raddr *TCPAddr) (*TCPConn, error)

// network参数是"tcp4"、"tcp6"、"tcp"中的任意一个，分别表示TCP(IPv4-only)、TCP(IPv6-only)或者TCP(IPv4,IPv6的任意一个)
// laddr表示本机地址，一般设置为nil
// raddr表示远程的服务地址
```

TCP有很多连接控制函数，我们平常用到比较多的有如下几个函数：
```Go

func DialTimeout(net, addr string, timeout time.Duration) (Conn, error)

```
设置建立连接的超时时间，客户端和服务器端都适用，当超过设置时间时，连接自动关闭。
```Go

func (c *TCPConn) SetReadDeadline(t time.Time) error
func (c *TCPConn) SetWriteDeadline(t time.Time) error

```
用来设置写入/读取一个连接的超时时间。当超过设置时间时，连接自动关闭。
```Go

func (c *TCPConn) SetKeepAlive(keepalive bool) os.Error
```	
设置keepAlive属性，是操作系统层在tcp上没有数据和ACK的时候，会间隔性的发送keepalive包，操作系统可以通过该包来判断一个tcp连接是否已经断开，在windows上默认2个小时没有收到数据和keepalive包的时候人为tcp连接已经断开，这个功能和我们通常在应用层加的心跳包的功能类似。
