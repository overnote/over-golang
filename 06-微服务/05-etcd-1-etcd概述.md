## 一 etcd简介

#### 1.1 etcd是什么

etcd是一个分布式KV存储库，内部采用Raft协议作为一致性算法选举leader，同步key-value，其特性是：高可用，强一致。  

集群一般采取大多数模型(quorum)来选举leader，即集群需要2N+1个节点，这时总能产生1个leader，多个follower。etcd也不例外，每个etcd cluster都由若干个member组成，每个member是一个独立运行的etcd实例，单机上也可以运行多个member。  

在正常运行的状态下，集群中会有一个 leader，其余的 member 都是 followers。leader 向 followers 同步日志，保证数据在各个 member 都有副本。leader 还会定时向所有的 member 发送心跳报文，如果在规定的时间里 follower 没有收到心跳，就会重新进行选举。客户端所有的请求都会先发送给 leader，leader 向所有的 followers 同步日志，等收到超过半数的确认后就把该日志存储到磁盘，并返回响应客户端。  

每个 etcd 服务有三大主要部分组成：
- raft 实现
- WAL 日志存储：在本地磁盘（--data-dir）上存储日志内容（wal file）和快照（snapshot）
- 数据的存储和索引

etcd调用阶段：
- 阶段1：调用者调用leader，leader会将kv数据存储在日志中，并利用实时算法raft进行复制
- 阶段2：当复制给了N+1个节点后，本地提交，返回给客户端，最后leader异步通知follower完成通知

注意：日志只要复制给了大多数就不会丢。

raft日志概念：
- replication：日志在leader生成，向follower复制，最终达到各个节点日志序列一致
- term：任期，重新选举产生的leader，其term单调递增
- log index：日志行在日志序列的下标


## 二 etcd安装

### 2.1 Linux安装

启动
```
## 启动：强制其监听在公网端口
nohup ./etcd --listen-client-urls 'http://0.0.0.0:2379' --advertise-client-urls  'http://0.0.0.0:2379' &

## 查看日志，确认etcd是否启动成功
less nohup.out
```


### 2.2 Mac安装

安装etcd：
```
brew search etcd
brew install etcd
```

运行：
```
etcd
```

### 2.3 启动后的一些默认显示

在启动etcd后，会显示一些配置信息：
- etcdserver: name = default name表示节点名称，默认为default
- data-dir：保存日志和快照的目录，默认为当前工作目录default.etcd/
- 通信相关:
    - 在http://localhost:2380和集群中其他节点通信
    - 在http://localhost:2379提供HTTP API服务，供客户端交互。等会配置webui就是这个地址
- etcdserver: heartbeat = 100ms leader发送心跳到followers的间隔时间
- etcdserver: election = 1000ms 重新投票的超时时间，如果follow在该时间间隔没有收到心跳包，会触发重新投票，默认为1000ms

### 2.4  etcd webui

这里使用了一个nodejs开发的web：
```
git clone https://github.com/henszey/etcd-browser.git
cd etcd-browser/
vim server.js  

var etcdHost = process.env.ETCD_HOST || '127.0.0.1';  # etcd 主机IP
var etcdPort = process.env.ETCD_PORT || 4001;          # etcd 主机端口
var serverPort = process.env.SERVER_PORT || 8000;      # etcd-browser 监听端口

# 启动
node server.js
```

## 三 etcd客户端操作

常用命令：
```
ETCDCTL_API=3 ./etcdctl # 查看所有命令
ETCDCTL_API=3 ./etcdctl put "hello" "world"
ETCDCTL_API=3 ./etcdctl get "hello"

# 顺序存储的键可以使用前缀模糊查询
ETCDCTL_API=3 ./etcdctl put "/users/user1" "zs"
ETCDCTL_API=3 ./etcdctl put "/users/user2" "ls"
ETCDCTL_API=3 ./etcdctl get "/users/" --prefix      # 查询全部该前缀
ETCDCTL_API=3 ./etcdctl watch "/users/" --prefix    # 监听该前缀数据变化，此时另起命令行操作数据，则当前命令行能监听到
```