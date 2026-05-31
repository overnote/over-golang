## 一 etcd中的事务

## 二 示例
```go
	// 第一步：加锁（创建租约，确保租约不过期，使用租约抢占key）

	// 申请一个5秒租约
	lease := clientv3.NewLease(cli)
	leaseR, err := lease.Grant(context.TODO(), 5)
	if err != nil {
		fmt.Println("lease err:", err)
		return
	}

	// 第三步中的释放锁 准备一个用于取消自动续租的context
	ctx, cancelFunc := context.WithCancel(context.TODO())
	defer cancelFunc()
	defer lease.Revoke(context.TODO(), leaseR.ID)		// 释放租约

	// 自动续租 返回值是个只读的chan，因为写入只能是etcd实现
	keepChan, err := lease.KeepAlive(ctx, leaseR.ID )
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

	// 使用事务判断key是否存在；判断其
	key := "/cron/lock/jobX"
	kv := clientv3.NewKV(cli)
	txn := kv.Txn(context.TODO())		// 分布式事务
	txn.If(clientv3.Compare(clientv3.CreateRevision(key), "=", 0)).
		Then(clientv3.OpPut(key, "xxx", clientv3.WithLease(leaseR.ID))).		// 一般这里val记录是哪个ID抢到
		Else(clientv3.OpGet(key))		// 否则抢锁失败

	// 提交事务
	txnR, err := txn.Commit()
	if err != nil {
		fmt.Println("txn失败:", err)
		return
	}
	// 判断是否抢到了锁
	if !txnR.Succeeded {
		fmt.Println("没抢到锁，锁已被占用；", string(txnR.Responses[0].GetResponseRange().Kvs[0].Value))
		return
	}

	// 第二步：业务代码书写
	fmt.Println("模拟处理任务")
	time.Sleep(time.Second * 5)

    // 第三步：释放锁（取消续租，释放租约）
```

多次执行上述方法，观察结果