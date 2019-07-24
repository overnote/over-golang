## 一 Go安装

推荐使用官方的安装包直接安装，下载地址：https://golang.org/dl/  

```
Win安装：
        打开Win安装包下一步下一步即可，默认安装在目录：c:\Go

Mac安装：
        需要预装Xcode，然后打开Mac安装包下一步下一步即可  

Linux安装：
        wget https://storage.googleapis.com/golang/go1.12.1.linux-amd64.tar.gz
        tar zxvf go*.tar.gz -C /usr/local/ 
        # 配置环境
        vim /etc/profile
        export GOROOT=/usr/local/go         # 默认/usr/local/go/go,（新版go目录下还有一个go目录）
        export PATH=$PATH:$GOROOT/bin
        export GOPATH=$HOME/goproject       # 可选配置：Go项目目录，1.12版本后推荐使用go mod，不再使用GOPATH
​        source /etc/profile 

测试安装： 
        go version
```

## 二  Go环境

#### 2.1 GOPATH

```
GOROOT: Go的安装目录，比如c:/Go
GOPATH: Go的项目目录，1.1-1.7必须设置，
        1.8后Unix上默认为$HOME/go,Win上默认为%USERPROFILE%/go
        当有多个项目目录时，请注意分隔符（Windows是分号，Linux是冒号），且默认会将go get的内容放在第一个目录下。
```


GOPATH目录约定有三个子目录：
- src:存放源代码（比如：.go .c .h .s等）
- pkg:编译后生成的文件（比如：.a）
- bin:编译后生成的可执行文件（为了方便，可以把此目录加入到`$PATH` 变量中，如果有多个gopath，那么使用`${GOPATH//://bin:}/bin`添加所有的bin目录）

win的GOPATH重设办法：进入环境变量
![](../images/Golang/lang-01.png)


#### 2.2 GO MOD

在go1.11版本之后，可以不再依赖gopath，从此摆脱gopath的噩梦。go mod相关介绍，参见 目录[02-12节:依赖管理](https://github.com/overnote/golang/blob/master/02-Go语法/12-包与依赖管理.md)