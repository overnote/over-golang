## 一 protobuf简单使用

新建一个protobuf文件：hello.proto
```
syntax = "proto3";                  

message HelloRequest {
  string name = 1; 
  int32 height = 2;       
  string email = 3;          
  repeated int32 weight = 4 [packed=true];      
}

message TestResponse {
    string text = 1;
}
```

说明：
- 上述示例中，创建了2个消息HelloRequest和TestResponse
- 消息中的值，1-4分别是键对应的数字id

## 二 protobuf语法

#### 2.1 修饰前缀

- required：表示该字段有且只有1个，在3.0中该修饰符被移除
- optional：表示该字段可以是0或1个，后面可加default默认值，如果不加，使用默认值
- repeated：表示该字段可以是0到多个，packed=true 代表使用高效编码格式

注意：
- id在1-15之间编码只需要占一个字节，包括Filed数据类型和Filed对应数字id
- id在16-2047之间编码需要占两个字节，所以最常用的数据对应id要尽量小一些
- 使用required规则的时候要谨慎，因为以后结构若发生更改，这个Filed若被删除的话将可能导致兼容性的问题。

#### 2.2 默认值

- strings：默认是一个空string 
- bytes：默认是一个空的bytes 
- bools：默认是false 
- 数值类型：默认是0

#### 2.3 保留字段与id

每个字段对应唯一的数字id，但是如果该结构在之后的版本中某个Filed删除了，为了保持向前兼容性，需要将一些id或名称设置为保留的，即不能被用来定义新的Field。

```
message Person {
  reserved 2, 15, 9 to 11;
  reserved "samples", "email";
}
```

#### 2.4 枚举类型
比如电话号码，只有移动电话、家庭电话、工作电话三种，因此枚举作为选项，如果没设置的话枚举类型的默认值为第一项。在上面的例子中在个人message中加入电话号码这个Filed。如果枚举类型中有不同的名字对应相同的数字id，需要加入option allow_alias = true这一项，否则会报错。枚举类型中也有reserverd Filed和number，定义和message中一样。
```
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}
```

#### 2.5 引用其他message类
在同一个文件中，可以直接引用定义过的message类型，在同一个项目中，可以用import来导入其它message类型。

```
import "myproject/other_protos.proto";
```

#### 2.6 message扩展

```
message Person {
  // ...
  extensions 100 to 199;
}
```

在另一个文件中，import 这个proto之后，可以对Person这个message进行扩展。
```
extend Person {
  optional int32 bar = 126;
}
```


#### 2.7 使用其他消息类型与嵌套

可以将其他消息类型作为字段类型，例如：在每一个PersonInfo消息中包含Person消息，此时可以在相同 的.proto文件中定义一个Result消息类型，然后在PersonInfo消息中指定一个Person类型的字段。  

```
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
```
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

## 三 编码原理

#### 3.1 可变长整数编码

每个字节有8bits，其中第一个bit是most significant bit(msb)，0表示结束，1表示还要读接下来的字节。  

对message中每个Filed来说，需要编码它的数据类型、对应id以及具体数据。  

比如对于下面这个例子来说，如果给a赋值150，那么最终得到的编码是什么呢？
```
message Test {
  optional int32 a = 1;
}
```

首先数据类型编码是000，因此和id联合起来的编码是00001000. 然后值150的编码是1 0010110，采用小端序交换位置，即0010110 0000001，前面补1后面补0，即10010110 00000001，即96 01，加上最前面的数据类型编码字节，总的编码为08 96 01。  

#### 3.2 有符号整数编码

如果用int32来保存一个负数，结果总是有10个字节长度，被看做是一个非常大的无符号整数。使用有符号类型会更高效。它使用一种ZigZag的方式进行编码。即-1编码成1，1编码成2，-2编码成3这种形式。  

也就是说，对于sint32来说，n编码成 (n << 1) ^ (n >> 31)，注意到第二个移位是算法移位。  