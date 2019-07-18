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

## 三 golang简单操作etcd示例

#### 3.1 示例1

```go
package main

import (
	"context"
	"fmt"
	"github.com/etcd-io/etcd/clientv3"
	"time"
)

func main() {

	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"192.168.1.1:2379", "192.168.1.1:22379", "192.168.1.1:32379"},	// etcd的集群数组，我们这里只有1个
		DialTimeout: 5 * time.Second,
	})

	if err != nil {
		fmt.Println("err:", err)
	}

	defer cli.Close()

	// 通过客户端获取键值对对象
	r1,err := cli.Put(context.TODO(), "/users/user1", "zhangsan")
	if err != nil {
		fmt.Println("err1:", err)
		return
	}
	r2,err := cli.Put(context.TODO(), "/users/user2", "lisi", clientv3.WithPrevKV())
	if err != nil {
		fmt.Println("err2:", err)
		return
	}
	// r1: &{cluster_id:14841639068965178418 member_id:10276657743932975437 revision:3 raft_term:2  <nil>}
	fmt.Println("r1:", r1)

	fmt.Println("r2:", r2)
	// 获取前一个版本的值，必须设置  clientv3.WithPrevKV()
	fmt.Println("r2.PreKv:", r2.PrevKv)
}

```

#### 3.2 示例2

```go
package main

import (
	"context"
	"fmt"
	"github.com/etcd-io/etcd/clientv3"
	"time"
)



func main() {

	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"192.168.1.1:2379", "192.168.1.1:22379", "192.168.1.1:32379"},	// etcd的集群数组，我们这里只有1个
		DialTimeout: 5 * time.Second,
	})

	if err != nil {
		fmt.Println("err:", err)
	}

	defer cli.Close()

	// 申请1个10秒的租约
	leaseR, err := cli.Lease.Grant(context.TODO(), 10)
	if err != nil {
		fmt.Println("err:", err)
		return
	}
	leaseID := leaseR.ID
	// 设置1个10秒后过期的kv
	r, err := cli.Put(context.TODO(),"test4", "lease", clientv3.WithLease(leaseID))
	if r != nil {
		fmt.Println("err:", err)
		return
	}
	fmt.Println("r:", r)

	//其他API： lease
}
```