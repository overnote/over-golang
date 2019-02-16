## 一 Socket编程
#### 1.1 Socket简介
很多游戏服务器使用Socket编写服务器，因为对于HTTP协议来说，能够节省性能开支。大部分底层网络编程都依赖于Socket编程：HTTP，IM通信，视频流传输的底层都是Socket。
Socket起源于UNIX，本着UNIX一切皆文件的哲学，可以用 打开-读写-关闭 的方式操作。网络的Socket数据传输是一种特殊的I/O，Socket也是一种文件描述符。Web开发中，Socket编程主要面向OSI模型的第三层和第四层协议，即：IP协议，TCP协议，UDP协议。
流式Socket：面向连接，主要用于TCP服务；
数据式Socket：无连接，主要用于UDP服务。
本地的进程可以通过PID来标识唯一，而网络中的进程通过网络层的 IP ，传输层的 协议+端口 来标识。
#### 1.2 TCP
Go语言通过net包中的DialTCP函数来建立一个TCP连接，并返回一个TCOConn类型的对象，当连接建立时服务器也创建一个同类型的对象，此时客户端和服务端通过各自拥有的TCPConn对象进行数据交换。只有当任意一端关闭连接，才会失效。
服务端：
```Go
func main() {

	address := net.TCPAddr{
		IP: net.ParseIP("127.0.0.1"),
		Port: 8000,
	}

	listener, err := net.ListenTCP("tcp4", &address)
	if err != nil {
		log.Fatal(err)
	}

	for {

		conn, err := listener.AcceptTCP()
		if err != nil {
			log.Fatal(err)          //服务端记录错误，但是不退出
		}

		fmt.Println("远程地址：", conn.RemoteAddr())
		go echo(conn)               //实现多并发执行连接
	}

}

func echo(conn *net.TCPConn){
	tick := time.Tick(5 * time.Second)  	//5秒请求一次
	for now := range tick {
		n, err := conn.Write([]byte(now.String()))
		if err != nil {
			log.Println(err)
			conn.Close()
			return
		}
		fmt.Println("second %d bytes to %s\n", n ,conn.RemoteAddr())
	}
}
```
客户端：
```Go
func main(){

	addr := "127.0.0.1:8000"

	tcpAddr, err := net.ResolveTCPAddr("tcp4", addr)
	checkError(err)

	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	checkError(err)

	conn.Write([]byte("HEAD / HTTP/1.0\r\n\r\n"))

	result, err := ioutil.ReadAll(conn)
	checkError(err)

	fmt.Println(string(result))

	os.Exit(0)

}

func checkError(err error) {
	if err != nil {
		fmt.Println(os.Stderr, "错误：%s", err.Error())
		os.Exit(1)
	}
}
```
#### 1.3 UDP
UDP缺少了对客户端连接请求的Accept函数，其他几乎一样
#### 1.4 HTTP
内置的net/http包可以创建HTTP服务：
见Web编程
