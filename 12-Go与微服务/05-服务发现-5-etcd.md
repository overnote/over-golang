## 一 etcd简介

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