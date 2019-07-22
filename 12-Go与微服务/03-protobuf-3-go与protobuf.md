## 一 安装Go语言编译protobuf环境

安装Go语言的proto API接口：
```
go get -v -u github.com/golang/protobuf/proto
```

安装protoc-gen-go插件：这是个go程序
```
go get -v -u github.com/golang/protobuf/protoc-gen-go

# 拷贝命令，开启go mod时，该命令位于 go/bin/
cp protoc-gen-go /usr/local/bin/  

# 贴士，低版本golang需要编译该源码为可执行文件，然后拷贝该文件
cd /gopath/github.com/golang/protobuf/protoc-gen-go/           
go build        
```

## 二 编译proto文件

在项目根目录新建./protoes/hello.proto文件，内容如下：
```
syntax = "proto3";               

package protoes;                  // 指定包

message HelloRequest {
  string name = 1;                 // 1-4分别是键对应的数字id
  int32 u_count = 2;     
}
```

编译：
```
cd protoes
protoc --go_out=./ *.proto      # 在当前目录生成了文件 hello.pb.go
```

当用protocol buffer编译器来运行.proto文件时，编译器将生成所选择语言的代码，这些代码可以操作在.proto文件中定义的消息类型，包括获取、设置字段值，将消息序列化到一个输出流中，以及从一个输入流中解析消息：
- 对C++来说，编译器会为每个.proto文件生成一个.h文件和一个.cc文件，.proto文件中的每一个消息有一个对应的 类。
- 对Python来说，有点不太一样——Python编译器为.proto文件中的每个消息类型生成一个含有静态描述符的模 块，，该模块与一个元类(metaclass)在运行时(runtime)被用来创建所需的Python数据访问类。
- 对go来说，编译器会为每个消息类型生成了一个.pd.go文件。  

hello.pb.go主要内容如下：
```go
package protoes

import (
	fmt "fmt"
	proto "github.com/golang/protobuf/proto"
	math "math"
)

// Reference imports to suppress errors if they are not otherwise used.
var _ = proto.Marshal
var _ = fmt.Errorf
var _ = math.Inf

const _ = proto.ProtoPackageIsVersion3 // please upgrade the proto package

type HelloRequest struct {
	Name                 string   `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
	UCount               int32    `protobuf:"varint,2,opt,name=u_count,json=uCount,proto3" json:"u_count,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```

## 三 使用Go代码获取数据

```go
package main

import (
	"fmt"
	"github.com/golang/protobuf/proto"
	"test/protoes"							// test是go mod的项目名
)

func main() {

	HelloRequest := protoes.HelloRequest{
		Name: *proto.String("lisi"),
		UCount: *proto.Int32(17),
	}

	// 序列化
	data, err := proto.Marshal(&HelloRequest)
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println("marshal data:", data)		// 一串流数据

	// 反序列化
	var list protoes.HelloRequest
	err = proto.Unmarshal(data, &list)
	if err != nil {
		fmt.Println("unmarshal error:", err)
		return
	}
	fmt.Println("Name:", list.GetName())		// lisi
	fmt.Println("UCount:", list.GetUCount())	// 17

}
```