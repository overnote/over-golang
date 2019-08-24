## 一 数据交互格式

常见数据交互格式有：
- xml：在webservice中应用最为广泛，但是相比于json，它的数据更加冗余，因为需要成对的闭合标签。
- json：一般的web项目中，最流行的主要还是json。因为浏览器对于json数据支持非常好，有很多内建的函数支 持。
- protobuf：后起之秀，谷歌开源的一种数据格式

## 二 protobuf简介

protobuf是google于2008年开源的可扩展序列化结构数据格式。相较于传统的xml和json，protobuf更适合高性能，对响应速度有要求的数据传输场景。  

利用protobuf，可以自定义数据结构，使用protobuf对数据结构进行一次描述，即可利用各种不同语言或从各种不同数据流中对结构化数据进行轻松读写。  

protobuf优点：
- 1:序列化后体积相比Json和XML很小，适合网络传输 
- 2:支持跨平台多语言 
- 3:消息格式升级和兼容性还不错 
- 4:序列化反序列化速度很快，快于Json的处理速速

protobuf也有其不可忽视的缺点：
- 功能简单，无法用来表示复杂的概念。
- protobuf是二进制数据格式，需要编码和解码，数据本身不具有可读性，因此只能反序列化之后得到真正可读的数据。而XML已经是行业的标准工具，且具备自我解释性，可以被人们直接读取编辑。

## 三 protobuf安装

#### 3.1 linux安装protobuf

下面列出centOS的安装方式：
```
# 安装依赖
yum install autoconf automake libtool curl make g++ unzip libffi-dev  glibc-headers gcc-c++ -y

# 下载
git clone https://github.com/protocolbuffers/protobuf.git

# 安装
unzip protobuf.zip
cd protobuf
./autogen.sh
./configure
make && make install
ldconfig        # 刷新共享库

# 测试
protoc --version
```

#### 3.2 mac安装protobuf

mac推荐使用homebrew安装：  
```
# 安装
brew install protobuf
brew install automake
brew install libtool

# 测试
protoc --version
```

#### 3.3 win安装protobuf

下载地址: https://github.com/google/protobuf/releases  

下载win版本后，配置文件中的bin目录到Path环境变量下即可。

```
protoc --version    #查看protoc的版本
```
