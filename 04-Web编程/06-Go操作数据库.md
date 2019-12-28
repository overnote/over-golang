## 零 Go数据库接口

Go官方没有提供数据库驱动，而是为开发数据库驱动定义了一些标准接口，开发者可以根据定义的接口来开发相应的数据库驱动，这样做的好处是：框架迁移极其方便。  

Go数据库标准包位于以下两个包中：
- database/sql：提供了保证SQL或类SQL数据库的泛用接口
- database/sql/driver：定义了应被数据库驱动实现的接口，这些接口会被sql包使用

关于包的详细使用位于![07-标准库](https://github.com/overnote/over-golang/tree/master/07-标准库)中。 

## 一 Go操作MySQL

Go中支持MySQL的驱动目前比较多，有如下几种，有些是支持database/sql标准，而有些是采用了自己的实现接口。  

推荐使用：
- https://github.com/go-sql-driver/mysql （Go编写，维护频繁，支持Gosql接口，底层支持keepalive）
- https://github.com/jmoiron/sqlx（推荐）

代码示例：

```Go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	
	db, err := sql.Open("mysql", "root:123456@/mydata?charset=utf8")
	checkErr(err)

	//插入数据
	sql1 := "INSERT INTO user SET name=?,age=?"
	stmt, err := db.Prepare(sql1)
	checkErr(err)
	res, err := stmt.Exec("zs", "30",)
	checkErr(err)

	id, err := res.LastInsertId()
	checkErr(err)
	fmt.Println("插入id=", id)


	//更新数据
	sql2 := "UPDATE user SET name=? WHERE id=?";
	stmt, err = db.Prepare(sql2)
	checkErr(err)
	res, err = stmt.Exec("lisi", id)
	checkErr(err)
	affect, err := res.RowsAffected()
	checkErr(err)
	fmt.Println("更新行数=", affect)

	//查询数据
	rows, err := db.Query("SELECT * FROM user")
	checkErr(err)
	for rows.Next() {
		var id int
		var name string
		var age int
		err = rows.Scan(&id, &name, &age)
		checkErr(err)
		fmt.Println("id=", id)
		fmt.Println("name=", name)
		fmt.Println("age=", age)
	}

	//删除数据
	// stmt, err = db.Prepare("DELETE FROM user WHERE uid=?")
	// checkErr(err)
	// res, err = stmt.Exec(id)
	// checkErr(err)
	// affect, err = res.RowsAffected()
	// checkErr(err)
	// fmt.Println(affect)

	db.Close()

}

func checkErr(err error) {
	if err != nil {
		panic(err)
	}
}
```

相关API：
- sql.Open()：打开一个注册过的数据库驱动，第二个参数格式有：
  - `user@unix(/path/to/socket)/dbname?charset=utf8`
  - `user:password@tcp(localhost:5555)/dbname?charset=utf8`
  - `user:password@/dbname`
  - `user:password@tcp([de:ad:be:ef::ca:fe]:80)/dbname`
- db.Prepare()：返回准备要执行的sql操作，然后返回准备完毕的执行状态
- db.Query()：直接执行Sql返回Rows结果
- stmt.Exec()：执行stmt准备好的SQL语句

## 二 Go操作redis

#### 2.1 Go操作Redis

推荐驱动： 
- https://github.com/go-redis/redis
- https://github.com/gomodule/redigo

基本操作：
```go
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
)


func main() {

	c, err := redis.Dial("tcp", "localhost:6379")

	if err != nil {
		fmt.Println("Redis connect err:", err)
		return
	}

	defer c.Close()

	_, err = c.Do("Set", "first", "hello")
	if err != nil {
		fmt.Println(err)
		return
	}

	r, err := redis.Int(c.Do("Get", "first"))
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(r)
}	
```

#### 2.2 连接池

```Go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gomodule/redigo/redis"
)

var (
	Pool *redis.Pool
)

func init() {
	redisHost := ":6379"
	Pool = newPool(redisHost)
	listenForClose()
}

func newPool(server string) *redis.Pool {

	return &redis.Pool{
		MaxIdle:     3,
		IdleTimeout: 240 * time.Second,

		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", server)
			if err != nil {
				return nil, err
			}
			return c, err
		},

		TestOnBorrow: func(c redis.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}
}

func listenForClose() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	signal.Notify(c, syscall.SIGTERM)
	signal.Notify(c, syscall.SIGKILL)
	go func() {
		<-c
		_ = Pool.Close()
		os.Exit(0)
	}()
}

func Get(key string) ([]byte, error) {

	conn := Pool.Get()
	defer conn.Close()
	var data []byte
	data, err := redis.Bytes(conn.Do("GET", key))
	if err != nil {
		return data, fmt.Errorf("error get key %s: %v", key, err)
	}
	return data, err
}

func main() {
	test, err := Get("test")
	fmt.Println(test, err)
	ExampleNewClient()
	// Output
	// [] error get key test: redigo: nil returned
	// PONG <nil>
	// 4396 <nil>
	// key: overnote2 not exist
	// 0 redigo: nil returned
}

func ExampleNewClient() {
	c, _ := redis.Dial("tcp", "localhost:6379",
		redis.DialDatabase(0),
	)
	pong, err := redis.String(c.Do("ping"))
	fmt.Println(pong, err)
	// Output: PONG <nil>
	key := "overnote"
	val := 4396
	_, err = c.Do("SET", key, val)
	if err != nil {
		panic(err)
	}
	res, err := redis.Int(c.Do("GET", key))
	if err == redis.ErrNil {
		fmt.Printf("key: %s not exist\n", key)
	} else {
		if err != nil {
			panic(err)
		}
	}
	fmt.Println(res, err)
	// 4396 nil
	notExistKey := "overnote2"
	res, err = redis.Int(c.Do("GET", notExistKey))
	if err == redis.ErrNil {
		fmt.Printf("key: %s not exist\n", notExistKey)
		// key: overnote2 not exist
	} else {
		if err != nil {
			panic(err)
		}
	}
	fmt.Println(res, err)
	// 0 redigo: nil returned
}

```

## 三 Go操作MongoDB

MongoDB为Go提供了官方驱动：  

https://github.com/mongodb/mongo-go-driver  