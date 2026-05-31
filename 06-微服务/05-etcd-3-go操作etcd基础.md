## 一 golang对etcd的增删改查

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
		 // etcd的集群数组，我们这里只有1个
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

	// 添加键值对
	r1, err := kv.Put(context.TODO(), "/lesson/math", "100")		// 添加数学课程为100分
	if err != nil {
		fmt.Println("put key1 err:", err)
		return
	}

	// 继续添加键值对
	r2, err := kv.Put(context.TODO(), "/lesson/music", "50")		// 添加音乐课程为50分
	if err != nil {
		fmt.Println("put key2 err:", err)
		return
	}

	fmt.Println("添加结果r1: ", r1)			// &{cluster_id:14841639068965178418 member_id:10276657743932975437 revision:9 raft_term:2  <nil>}
	fmt.Println("添加结果r2: ", r2)			// &{cluster_id:14841639068965178418 member_id:10276657743932975437 revision:10 raft_term:2  <nil>}

	// 获取整个 /lesson目录下的数据
	getAll, err := kv.Get(context.TODO(), "/lesson/", clientv3.WithPrefix())
	if err != nil {
		fmt.Println("select all err: ", err)
		return
	}
	// [key:"/lesson/math" create_revision:9 mod_revision:25 version:8 value:"100"  key:"/lesson/music" create_revision:26 mod_revision:26 version:1 value:"50" ]
	fmt.Println("查询所有：", getAll.Kvs)

	// 删除键值对,如果添加参数:clientv3.WithPrevKV()，则delResult结果中包含删除前的结果：PrevKvs
	delResult, err := kv.Delete(context.TODO(), "/lesson/music")
	if err != nil {
		fmt.Println("del key2 err:", err)
		return
	}
	fmt.Println("删除r2结果:", delResult)

	// 修改键值对，修改仍然是Put
	updRerulst, err := kv.Put(context.TODO(), "/lesson/math", "30")
	if err != nil {
		fmt.Println("upd key2 err:", err)
		return
	}
	fmt.Println("修改r1结果:", updRerulst)

	// 查询当前r1的值 该函数支持重载，第三个参数都是 clientv3.With***，用来限制返回结果
	getR1, err := kv.Get(context.TODO(), "/lesson/math")
	if err != nil {
		fmt.Println("select r1 err: ", err)
		return
	}
	//  [key:"/lesson/math" create_revision:9 mod_revision:13 version:3 value:"100" ]
	fmt.Println("查询r1结果：", getR1.Kvs)	

	// 查询被删除的r2的值
	getR2, err := kv.Get(context.TODO(), "/lesson/music")
	if err != nil {
		fmt.Println("select r2 err: ", err)
		return
	}
	//  []
	fmt.Println("查询r2结果：", getR2.Kvs)	

}
```

## 二 批量操作

- 批量删除：`kv.Delete(context.TODO(), "/lesson/", clientv3.WithPrevfix())`
- 批量按顺序删除，并删除2个：`kv.Delete(context.TODO(), "/lesson/lesson1", clientv3.WithFromKey(),clientv3.WithLimit(2))`

## 三  使用OP操作代替原有的增删改查

```go

	// 创建Op
	putOp := clientv3.OpPut("/cron/jobs/job8", "888")
	// 执行Op
	opR, err := kv.Do(context.TODO(), putOp)
	if err != nil {
		fmt.Println("putOp err:", err)
		return
	}
	fmt.Println("写入Revision：", opR.Put().Header.Revision)

	// 创建Op
	getOp := clientv3.OpGet("/cron/jobs/job8")
	// 执行Op
	opR2, err := kv.Do(context.TODO(), getOp)
	if err != nil {
		fmt.Println("getOp err:", err)
		return
	}
	fmt.Println("获取Revisoon：", opR2.Get().Kvs[0].ModRevision)
	fmt.Println("获取Value：", opR2.Get().Kvs[0].Value)
```
