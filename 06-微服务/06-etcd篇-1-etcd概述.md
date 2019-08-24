## 一 etcd简介

#### 1.1 etc是什么

etcd是一个分布式KV存储软件，其利用raft算法在集群中同步key-value，raft算法是强一致的集群日志同步算法。  

集群一般采取大多数模型(quorum)来选举leader，即集群需要2N+1个节点，这时总能产生1个leader，多个follower。  

etcd调用阶段：
- 阶段1：调用者调用leader，leader会将kv数据存储在日志中，并利用实时算法raft进行复制
- 阶段2：当复制给了N+1个节点后，本地提交，返回给客户端，最后leader异步通知follower完成通知

注意：日志只要复制给了大多数就不会丢。

raft日志概念：
- replication：日志在leader生成，向follower复制，最终达到各个节点日志序列一致
- term：任期，重新选举产生的leader，其term单调递增
- log index：日志行在日志序列的下标

#### 1.2 etc实现服务发现

etcd是一个采用HTTP协议的健/值对存储系统，它是一个分布式和功能层次配置系统，可用于构建服务发现系统。其很容易部署、安装和使用，提供了可靠的数据持久化特性。它是安全的并且文档也十分齐全。  

etcd比Zookeeper是比更好的选择，因为它很简单，然而，它需要搭配一些第三方工具才可以提供服务发现功能。  

现在，我们有一个地方来存储服务相关信息，我们还需要一个工具可以自动发送信息给etcd。但在这之后，为什么我们还需要手动把数据发送给etcd呢？即使我们希望手动将信息发送给etcd，我们通常情况下也不会知道是什么信息。记住这一点，服务可能会被部署到一台运行最少数量容器的服务器上，并且随机分配一个端口。理想情况下，这个工具应该监视所有节点上的Docker容器，并且每当有新容器运行或者现有的一个容器停止的时候更新etcd，其中的一个可以帮助我们达成目标的工具就是Registrator。  


Registrator通过检查容器在线或者停止运行状态自动注册和去注册服务，它目前支持etcd、Consul和SkyDNS 2。

Registrator与etcd是一个简单但是功能强大的组合，可以运行很多先进的技术。每当我们打开一个容器，所有数据将被存储在etcd并传播到集群中的所有节点。我们将决定什么信息是我们的。  

我们还需要一种方法来创建配置文件，与数据都存储在etcd，通过运行一些命令来创建这些配置文件。  

Confd是一个轻量级的配置管理工具，常见的用法是通过使用存储在etcd、consul和其他一些数据登记处的数据保持配置文件的最新状态，它也可以用来在配置文件改变时重新加载应用程序。换句话说，我们可以用存储在etcd（或者其他注册中心）的信息来重新配置所有服务。  

最后的组合如图所示：
![](../images/Golang/micro-02.png)  

当etcd、Registrator和Confd结合时，可以获得一个简单而强大的方法来自动化操作我们所有的服务发现和需要的配置。这个组合还展示了“小”工具正确组合的有效性，这三个小东西可以如我们所愿正好完成我们需要达到的目标，若范围稍微小一些，我们将无法完成我们面前的目标，而另一方面如果他们设计时考虑到更大的范围，我们将引入不必要的复杂性和服务器资源开销。



## 二 etcd客户端操作

启动
```
## 启动：强制其监听在公网端口
nohup ./etcd --listen-client-urls 'http://0.0.0.0:2379' --advertise-client-urls  'http://0.0.0.0:2379' &

## 查看日志，确认etcd是否启动成功
less nohup.out
```

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