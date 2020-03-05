## 一 go mod

go的项目依赖管理一直饱受诟病，在go1.11后正式引入了`go modules`功能，在go1.13版本中将会默认启用。从此可以不再依赖gopath，摆脱gopath的噩梦。  

`go mod` 初步使用：
```
# 开启go mod
export GO111MODULE=on			# 注意：如果是win，这里使用 set GO111MODULE=on

# 在新建的项目根目录下（src）下使用该命令
go mod init 项目名                      # 此时会生成一个go.mod文件

# 使用
在项目中可以随时import依赖，当 go run 时候，会自动安装依赖，比如：
import (
	"github.com/gin-gonic/gin"
)

```

go run 后的 go.mod:
```
module api_server

go 1.12

require (
	github.com/gin-contrib/sse v0.0.0-20190301062529-5545eab6dad3 // indirect
	github.com/gin-gonic/gin v1.3.0 // indirect
	github.com/golang/protobuf v1.3.1 // indirect
	github.com/mattn/go-isatty v0.0.7 // indirect
	github.com/ugorji/go/codec v0.0.0-20190320090025-2dc34c0b8780 // indirect
	gopkg.in/go-playground/validator.v8 v8.18.2 // indirect
	gopkg.in/yaml.v2 v2.2.2 // indirect
)
```

使用`go mod`后，run产生的依赖源码不会安装在当前项目中，而是安装在：`$GOPATH/pkg/mod`。  

贴士：如果我们安装的是go1.11以上版本，且想要开启go mod，那么可以给go配置环境如下：
```
export GOROOT=/usr/local/go                 # golang本身的安装位置
export GOPATH=~/go/                         # golang包的本地安装位置
export GOPROXY=https://goproxy.io           # golang包的下载代理
export GO111MODULE=on                       # 开启go mod模式
export PATH=$PATH:$GOROOT/bin               # go本身二进制文件的环境变量
export PATH=$PATH:$GOPATH/bin               # go第三方二进制文件的环境便令
```

注意：使用了go mod后，go get安装的包不再位于$GOPATHA/src 而是位于  $GOPATH/pkg/mod

## 二 翻墙问题解决

#### 2.1 推荐方式 GOPROXY

从 Go 1.11 版本开始，还新增了 GOPROXY 环境变量，如果设置了该变量，下载源代码时将会通过这个环境变量设置的代理地址，而不再是以前的直接从代码库下载。goproxy.io 这个开源项目帮我们实现好了我们想要的。该项目允许开发者一键构建自己的 GOPROXY 代理服务。同时，也提供了公用的代理服务 https://goproxy.io，我们只需设置该环境变量即可正常下载被墙的源码包了：

```
# 如果使用的是IDEA，开发时设置Goland的Prefrence-Go-proxy即可

# 如果使用的是VSCode，则
export GO111MODULE=on
export GOPROXY=https://goproxy.io			

# 如果是win，则：
set GO111MODULE=on
set GOPROXY=https://goproxy.io

# 关闭代理
export GOPROXY=
```

#### 2.2 replace方式

`go modules`还提供了 replace，可以解决包的别名问题，也能替我们解决 golang.org/x 无法下载的的问题。

go module 被集成到原生的 go mod 命令中，但是如果你的代码库在 $GOPATH 中，module 功能是默认不会开启的，想要开启也非常简单，通过一个环境变量即可开启 export GO111MODULE=on。

```go
module example.com/hello

require (
    golang.org/x/text v0.3.0
)

replace (
    golang.org/x/text => github.com/golang/text v0.3.0
)
```

#### 2.3 手动下载 旧版go的解决

我们常见的 golang.org/x/... 包，一般在 GitHub 上都有官方的镜像仓库对应。比如 golang.org/x/text 对应 github.com/golang/text。所以，我们可以手动下载或 clone 对应的 GitHub 仓库到指定的目录下。

mkdir $GOPATH/src/golang.org/x
cd $GOPATH/src/golang.org/x
git clone git@github.com:golang/text.git
rm -rf text/.git
当如果需要指定版本的时候，该方法就无解了，因为 GitHub 上的镜像仓库多数都没有 tag。并且，手动嘛，程序员怎么能干呢，尤其是依赖的依赖，太多了。

## 三 go mod引起的变化

引包方式变化：
- 不使用go mod 引包："./test"  引入test文件夹
- 使用go mod 引包："projectmodlue/test" 使用go.mod中的modlue名/包名

因为在go1.11后如果开启了`go mod`，需要在src目录下存在go.mod文件，并书写主module名（一般为项目名），否则无法build。

开启`go mod`编译运行变化：
- 使用vscode开发，必须在src目录下使用 `go build`命令执行，不要使用code runner插件
- 使用IDEA开发，项目本身配置go.mod文件扔不能支持，开发工具本身也要开启`go mod`支持（位于配置的go设置中）
