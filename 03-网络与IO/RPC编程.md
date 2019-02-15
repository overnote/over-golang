## 一 RPC编程
#### 1.1 RPC编程简介
Socket与HTTP编程都是客户端与服务端之间的通信，但是有些应用使用类似常规的函数调用的方式来完成想要的功能。RPC编程中，客户端就像调用本地函数一样，把参数打包后通过网络传递给服务器，服务器解包到处理过程中执行，并将执行结果返回给客户端。
RPC:远程过程调用协议，是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议，它假定某些传输协议存在，如TCP或UDP，以便为通信程序之间携带信息数据，通过它可以使函数调用模式网络化。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。
运行时，一次客户机对服务器的RPC调用，步骤如下：
> 1 调用客户端句柄，执行传送参数
> 2 调用本地系统内核发送网络消息
> 3 消息传送到远程主机
> 4 服务器句柄得到消息并得到参数
> 5 执行远程过程
> 6 返回执行结果给服务器句柄
> 7 服务器句柄返回结果，调用远程系统内核
> 8 消息传回本地主机
> 9 客户句柄由内核接收消息
> 10 客户接收句柄返回的数据
#### 1.2 Go RPC
Go支持三个级别的RPC：TCP,HTTP,JSONRPC,Go的RPC函数格式如下：
```Go
/**
* 函数必须是导出的
* 第一个参数是接收的参数，第二个参数是返回给客户端的参数，第二个参数必须是指针类型
* 必须有error返回值
*/
func (t *T) MethodName(argType T1, replyType *T2) error
```
#### 1.3 HTTP RPC
Server端：
```Go
type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("除数不能为0")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}

func main(){

	arith := new(Arith)			//创建一个Arith的RPC服务
	rpc.Register(arith)
	rpc.HandleHTTP()			//注册到HTTP协议上

	err := http.ListenAndServe(":1234", nil)

	if err != nil {
		fmt.Println(err.Error())
	}

}
```
Client端：
```Go
type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}

func checkError(err error) {
	if err != nil {
		fmt.Println( "错误：%s", err.Error())
		os.Exit(1)
	}
}

func main(){

	addr := "127.0.0.1:1234"

	client, err := rpc.DialHTTP("tcp", addr)
	checkError(err)

	args := Args{17, 8}
	var reply int
	err = client.Call("Arith.Multiply", args, &reply)
	checkError(err)
	fmt.Println("Arith:%d*%d=%d\n",args.A, args.B, reply)

	var quot Quotient
	err = client.Call("Arith.Divide", args, &quot)
	checkError(err)
	fmt.Println("Arith:%d/%d=%d 余 %d\n",args.A, args.B, quot.Quo, quot.Rem)

}
```
#### 1.4 TCP RPC
与HTTP服务器不同，需要自己控制连接，当有客户端连接上来后，需要把这个连接交给RPC来处理。
Server端：
```Go
type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("除数不能为0")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}


func checkError(err error) {
	if err != nil {
		fmt.Println( "错误：%s", err.Error())
		os.Exit(1)
	}
}

func main(){

	arith := new(Arith)			//创建一个Arith的RPC服务
	rpc.Register(arith)

	tcpAddr, err := net.ResolveTCPAddr("tcp", ":1234")
	checkError(err)

	listener, err := net.ListenTCP("tcp", tcpAddr)
	checkError(err)

	for {
		conn, err := listener.Accept()
		if err != nil {
			continue
		}
		rpc.ServeConn(conn)
	}

}
```
Client端：
```Go
//只需要修改http rpc方式如下：
client, err := rpc.Dial("tcp", addr)
```
#### 1.5 JSON RPC
Server端：
```Go
type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("除数不能为0")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}


func checkError(err error) {
	if err != nil {
		fmt.Println( "错误：%s", err.Error())
		os.Exit(1)
	}
}

func main(){

	arith := new(Arith)			//创建一个Arith的RPC服务
	rpc.Register(arith)

	tcpAddr, err := net.ResolveTCPAddr("tcp", ":1234")
	checkError(err)

	listener, err := net.ListenTCP("tcp", tcpAddr)
	checkError(err)

	for {
		conn, err := listener.Accept()
		if err != nil {
			continue
		}
		jsonrpc.ServeConn(conn)
	}

}
```
Client端:
```Go
client, err := jsonrpc.Dial("tcp", addr)
```

