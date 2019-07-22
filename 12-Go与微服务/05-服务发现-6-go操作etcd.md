## 一 golang简单操作etcd示例

#### 1.1 示例1

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

#### 1.2 示例2

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