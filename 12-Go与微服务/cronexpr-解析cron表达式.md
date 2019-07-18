## 一 cronexpr库的使用

#### 1.1 cronexpr入门示例

cronexpr库是Golang的第三方Cron表达式解析库。  

以下demo实现规定时间后，执行任务：
```go
package main

import (
	"fmt"
	"github.com/gorhill/cronexpr"
	"time"
)

func main() {

	// 当前时间
	var nowTime time.Time
	// 下次调度时间				
	var nextTime time.Time				

	//// 5秒钟后执行：支持 7位 *号，也支持6位： "*/5 * * * *" 表示1分钟后
	if expr, err := cronexpr.Parse("*/5 * * * * * *"); err != nil {
		fmt.Println("err1:", err)
	}

	nowTime = time.Now()				// 当前时间
	nextTime = expr.Next(nowTime)		// 下次调度时间：传入当前时间，即可知道下次执行时间

	time.AfterFunc(nextTime.Sub(nowTime), func() {
		fmt.Println("任务被调度：", nextTime)
	})

	// main运行20秒，防止直接退出
	time.Sleep(20 * time.Second)
}

```

#### 1.2 cronexpr调度多任务

```go
package main

import (
	"fmt"
	"github.com/gorhill/cronexpr"
	"time"
)

type CronJob struct {
	Expr *cronexpr.Expression
	NextTime time.Time
}

func main() {

	var cronJob *CronJob
	var expr *cronexpr.Expression
	var nowTime time.Time
	var scheduleTable map[string]*CronJob	//调度表

	nowTime = time.Now()				// 当前时间
	scheduleTable = make(map[string]*CronJob)

	// 需要1个协程，定时检查所有的CronJob任务，谁过期了执行谁

	// 定义第一个任务
	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		Expr:expr,
		NextTime:expr.Next(nowTime),
	}
	scheduleTable["job1"] = cronJob		// 任务放入调度表

	// 定义第二个任务
	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		Expr:expr,
		NextTime:expr.Next(nowTime),
	}
	scheduleTable["job2"] = cronJob		// 任务放入调度表

	// 启动协程
	go func() {
		var (
			jobName string
			jobCron *CronJob
			nowTime time.Time
		)

		// 定时检查调度表
		for {
			nowTime = time.Now()
			for jobName,jobCron = range scheduleTable {

				// 判断任务是否过期
				if jobCron.NextTime.Before(nowTime) || jobCron.NextTime.Equal(nowTime) {
					// 启动协程去执行该任务
					go func(jobName string) {
						fmt.Println("执行：", jobName)
					}(jobName)
				}

				// 必须计算下次执行时间
				jobCron.NextTime = cronJob.Expr.Next(nowTime)
				//fmt.Println("下次执行时间是：", jobCron.NextTime)

			}

			// 提升该协程性能：只让每100毫秒才来遍历一次
			select {
			case <- time.NewTimer(100 * time.Millisecond).C:	// 100毫秒后返回，防止for循环占用CPU

			}
		}
	}()

	// main运行100秒，防止直接退出
	time.Sleep(100 * time.Second)
}

```