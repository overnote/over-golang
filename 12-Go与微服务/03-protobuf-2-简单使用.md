## 一 安装

安装Go语言的proto API接口：
```
go get -v -u github.com/golang/protobuf/proto
```

安装protoc-gen-go插件：这是个go程序，编译之后将可执行文件复制到\bin目录。
```
go get -v -u github.com/golang/protobuf/protoc-gen-go
cd ~/go/github.com/golang/protobuf/protoc-gen-go/           # 注意没启用go mod时，位于gopath中
go build        # 生成可执行文件
cp protoc-gen-go /bin/
```

## 二 protobuf语法

#### 2.1 定义.proto文件

需要先定义hello.proto文件：
```go
syntax = "proto3";                  // 指定使用 proto3语法，不指定则编译器使用proto2，proto3不支持prto2的 枚举

// 第一个消息
message HelloRequest {
  string name = 1;                  // 
  int32 height = 2;                 // 
  repeated int32 weight = 3;        // repeated关键字表示重复的使用切片表示
}

// 第二个消息
message TestResponse {
    string text = 1;
}

```

HelloRequest消息格式有3个字段，在消息中承载的数据分别对应于每一个字段。其中每个字段都有一个名字和一种类型。  

当用protocol buffer编译器来运行.proto文件时，编译器将生成所选择语言的代码，这些代码可以操作在.proto文件中定义的消息类型，包括获取、设置字段值，将消息序列化到一个输出流中，以及从一个输入流中解析消息：
- 对C++来说，编译器会为每个.proto文件生成一个.h文件和一个.cc文件，.proto文件中的每一个消息有一个对应的 类。
- 对Python来说，有点不太一样——Python编译器为.proto文件中的每个消息类型生成一个含有静态描述符的模 块，，该模块与一个元类(metaclass)在运行时(runtime)被用来创建所需的Python数据访问类。
- 对go来说，编译器会为每个消息类型生成了一个.pd.go文件。  

#### 2.2 类型转换默认值

- strings：默认是一个空string 
- bytes：默认是一个空的bytes 
- bools：默认是false 
- 数值类型：默认是0

#### 2.3 使用其他消息类型与嵌套

可以将其他消息类型作为字段类型，例如：在每一个PersonInfo消息中包含Person消息，此时可以在相同 的.proto文件中定义一个Result消息类型，然后在PersonInfo消息中指定一个Person类型的字段。  

```go
message PersonInfo {
  repeated Person info = 1;
}
message Person {
  string name = 1;
  int32 shengao = 2;
  repeated int32 tizhong = 3;
}
```

可以在其他消息类型中定义、使用消息类型，在下面的例子中，Person消息就定义在PersonInfo消息内，如:
```go
message PersonInfo {
  message Person {
    string name = 1;
    int32 shengao = 2;
    repeated int32 tizhong = 3;
}
  repeated Person info = 1;
}
```
 
 如果你想在它的父消息类型的外部重用这个消息类型，你需要以PersonInfo.Person的形式使用它，如:
 ```go
 message PersonMessage {
  PersonInfo.Person info = 1;
}
 ```

 当然，你也可以将消息嵌套任意多层，如:
 ```go
 message Grandpa {
  message Father {  // Level 1
    message son {   // Level 2
      string name = 1;
      int32  age = 2;
} }
  message Uncle {  // Level 1
    message Son {   // Level 2
      string name = 1;
      int32  age = 2;
      } 
    }
}
 ```

#### 三 使用protobuf

#### 3.1 定义服务(Service)

如果想要将消息类型用在RPC(远程方法调用)系统中，可以在.proto文件中定义一个RPC服务接口，protocol buffer 编译器将会根据所选择的不同语言生成服务接口代码及存根。如，想要定义一个RPC服务并具有一个方法，该方法 能够接收 SearchRequest并返回一个SearchResponse，此时可以在.proto文件中进行如下定义:
```go
service SearchService {
//rpc 服务的函数名 (传入参数)返回(返回参数)
  rpc Search (SearchRequest) returns (SearchResponse);
}
```
最直观的使用protocol buffer的RPC系统是gRPC一个由谷歌开发的语言和平台中的开源的RPC系统，gRPC在使用 protocl buffer时非常有效，如果使用特殊的protocol buffer插件可以直接为您从.proto文件中产生相关的RPC代码。  

如果你不想使用gRPC，也可以使用protocol buffer用于自己的RPC实现，你可以从proto2语言指南中找到更多信 息

#### 3.2 生成访问类

可以通过定义好的.proto文件来生成Java,Python,C++, Ruby, JavaNano, Objective-C,或者C# 代码，需要基 于.proto文件运行protocol buffer编译器protoc。如果你没有安装编译器，下载安装包并遵照README安装。对于 Go,你还需要安装一个特殊的代码生成器插件。你可以通过GitHub上的protobuf库找到安装过程。  

通过如下方式调用protocol编译器:
```
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR  --python_out=DST_DIR --
go_out=DST_DIR   path/to/file.proto
```
IMPORT_PATH声明了一个.proto文件所在的解析import具体目录。如果忽略该值，则使用当前目录。如果有多个 目录则可以多次调用--proto_path，它们将会顺序的被访问并执行导入。-I=IMPORT_PATH是--proto_path的简化 形式。

当然也可以提供一个或多个输出路径:
- cpp_out 在目标目录DST_DIR中产生C++代码
- python_out 在目标目录 DST_DIR 中产生Python代码
- go_out 在目标目录 DST_DIR 中产生Go代码

必须提议一个或多个.proto文件作为输入，多个.proto文件可以只指定一次。虽然文件路径是相对于当前目录 的，每个文件必须位于其IMPORT_PATH下，以便每个文件可以确定其规范的名称。

#### 3.3 开始使用

编译test.proto文件：
```
protoc --go_out=./ *.proto          # 生成test.pb.go文件
```

执行转换：
```go
import (
    "fmt"
    "github.com/golang/protobuf/proto"
    "myproto"
)

func main() {
    test := &myproto.Test{
    Name : "test",
    Stature : 180,
    Weight : []int64{120,125,198,180,150,180}, Motto : "天行健，地势坤",
}
//将Struct test 转换成 protobuf data,err:= proto.Marshal(test) if err!=nil{ fmt.Println("转码失败",err)
}
//得到一个新的Test结构体 newTest newtest:= &myproto.Test{} //将data转换为test结构体
err = proto.Unmarshal(data,newtest) if err!=nil { fmt.Println("转码失败",err)
}
fmt.Println(newtest.String())
//得到name字段 fmt.Println("newtest->name",newtest.GetName()) fmt.Println("newtest->Stature",newtest.GetStature()) fmt.Println("newtest->Weight",newtest.GetWeight()) fmt.Println("newtest->Motto",newtest.GetMotto())
}        
```