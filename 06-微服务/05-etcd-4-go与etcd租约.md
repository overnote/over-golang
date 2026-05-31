## 一 租约机制（自动过期）

```go
package main

import (
	"context"
	"fmt"
	"github.com/etcd-io/etcd/clientv3"
	"time"
)

func connect() (client *clientv3.Client, err error){

	client, err = clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379", "127.0.0.1:22379", "127.0.0.1:32379"},
		DialTimeout: 5 * time.Second,
	})

	if err != nil {
		fmt.Println("connect err:", err)
		return nil, err
	}

	return client, err
}

func main() {

	// 连接
	cli, err := connect()
	defer cli.Close()
	if err != nil {
		return
	}

	// 获取etcd读写对象
	kv := clientv3.NewKV(cli)

	// 申请一个10秒租约
	lease := clientv3.NewLease(cli)
	leaseR, err := lease.Grant(context.TODO(), 10)
	if err != nil {
		fmt.Println("lease err:", err)
		return
	}

	// 使用该租约put一个kv
	putR, err := kv.Put(context.TODO(), "/cron/lock/job1", "10001", clientv3.WithLease(leaseR.ID))
	if err != nil {
		fmt.Println("put err:", err)
		return
	}
	fmt.Println("写入成功：", putR.Header.Revision)

	// 定时查看key是否过期
	for {
		getR, err := kv.Get(context.TODO(), "/cron/lock/job1")
		if err != nil {
			fmt.Println("get err:", err)
			return
		}
		if getR.Count == 0 {
			fmt.Println("key过期")
			break
		} else {
			fmt.Println("还未过期")
			time.Sleep(2 * time.Second)
		}
	}
}
```

#### 1.3 租约续租

我们希望能够续约，并能根据需要删除：
```go
package main

import (
	"context"
	"fmt"
	"github.com/etcd-io/etcd/clientv3"
	"time"
)

func connect() (client *clientv3.Client, err error){

	client, err = clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379", "127.0.0.1:22379", "127.0.0.1:32379"},
		DialTimeout: 5 * time.Second,
	})

	if err != nil {
		fmt.Println("connect err:", err)
		return nil, err
	}

	return client, err
}

func main() {

	// 连接
	cli, err := connect()
	defer cli.Close()
	if err != nil {
		return
	}

	// 获取etcd读写对象
	kv := clientv3.NewKV(cli)

	// 申请一个10秒租约
	lease := clientv3.NewLease(cli)
	leaseR, err := lease.Grant(context.TODO(), 10)
	if err != nil {
		fmt.Println("lease err:", err)
		return
	}
	
	// 自动续租 返回值是个只读的chan，因为写入只能是etcd实现
	keepChan, err := lease.KeepAlive(context.TODO(), leaseR.ID )
	if err != nil {
		fmt.Println("keep err:", err)
		return
	}
	// 启动一个协程去消费chan的应答
	go func(){
		for {
			select {
			case keepR := <- keepChan:
				if keepChan == nil {		// 此时系统异常或者主动取消context
					fmt.Println("租约失效")
					goto END
				} else {		// 每秒续租一次
					fmt.Println("收到自动续租应答：", keepR.ID)
				}
			}
		}
		END:
	}()

	// 使用该租约put一个kv
	putR, err := kv.Put(context.TODO(), "/cron/lock/job1", "10001", clientv3.WithLease(leaseR.ID))
	if err != nil {
		fmt.Println("put err:", err)
		return
	}
	fmt.Println("写入成功：", putR.Header.Revision)

	// 定时查看key是否过期
	for {
		getR, err := kv.Get(context.TODO(), "/cron/lock/job1")
		if err != nil {
			fmt.Println("get err:", err)
			return
		}
		if getR.Count == 0 {
			fmt.Println("key过期")
			break
		} else {
			fmt.Println("还未过期")
			time.Sleep(2 * time.Second)
		}
	}
}
```

如果我们要主动让context取消，则会让租约失效，现在定义一个5秒后取消的context：
```go
	// 续租了5秒，然后手动停止续租，即总共有15秒生命
	ctx, _ := context.WithTimeout(context.TODO(), 5 * time.Second)
	
	// 自动续租 返回值是个只读的chan，因为写入只能是etcd实现
	keepChan, err := lease.KeepAlive(ctx, leaseR.ID )
```