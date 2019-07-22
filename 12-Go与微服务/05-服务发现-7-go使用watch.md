#### 1.3 示例2 watch机制

```go
package main

import (
	"context"
	"fmt"
	"github.com/etcd-io/etcd/clientv3"
	"time"
	"github.com/coreos/etcd/mvcc/mvccpb"
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

	// 模拟变化
	go func() {
		for {
			kv.Put(context.TODO(), "/cron/jobs/job7", "job7")
			kv.Delete(context.TODO(), "/cron/jobs/job7")
			time.Sleep(1 * time.Second)
		}
	}()
	
	// 获取当前值
	getR, err := kv.Get(context.TODO(),  "/cron/jobs/job7")
	if err != nil {
		fmt.Println("get err:", err)
		return
	}
	if len(getR.Kvs) != 0 {		// key存在
		fmt.Println("当前值:", string(getR.Kvs[0].Value))
	}

	// 监听后续变化： revision是当前etcd集群事务ID，该ID是单调递增
	wathStartRevision := getR.Header.Revision + 1
	watcher := clientv3.NewWatcher(cli)			// 创建wathcer
	fmt.Println("从该版本向后监听：", wathStartRevision)
	watchChan := watcher.Watch(context.TODO(),  "/cron/jobs/job7", clientv3.WithRev(wathStartRevision))

	// 如果有变化，则会将变化丢到watchChan
	for watchResult := range watchChan{
		for _, event := range watchResult.Events {
			switch event.Type {
			case mvccpb.PUT:
				fmt.Println("修改为：", string(event.Kv.Value), "Revision:", event.Kv.CreateRevision, event.Kv.ModRevision)
			case mvccpb.DELETE:
				fmt.Println("删除了Revision:", event.Kv.ModRevision )
			}

		}
	}
}
```

如果要取消监听，同样是通过取消contex来实现：
```go
ctx, cancelFunc := context.WithTimeout(context.TODO(), 5 * time.Second)
// 5秒后执行退出函数
time.AfterFunc(5 * time.Second, func(){
    cancelFunc()
})
watchChan := watcher.Watch(ctx,  "/cron/jobs/job7", clientv3.WithRev(wathStartRevision))
```