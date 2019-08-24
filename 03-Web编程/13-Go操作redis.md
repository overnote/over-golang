## 一 Go操作Redis
推荐驱动： 
- https://github.com/go-redis/redis
- https://github.com/gomodule/redigo

基本操作：
```go
package main

import (
	"fmt"
	"go get github.com/gomodule/redigo/redis"
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

## 二 连接池

```Go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
	"go get github.com/gomodule/redigo/redis"
)

var (
	Pool *redis.Pool
)

func init() {
	redisHost := ":6379"
	Pool = newPool(redisHost)
	close()
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
		}
	}
}

func close() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	signal.Notify(c, syscall.SIGTERM)
	signal.Notify(c, syscall.SIGKILL)
	go func() {
		<-c
		Pool.Close()
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
}

func ExampleNewClient() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})

	pong, err := client.Ping().Result()
	fmt.Println(pong, err)
	// Output: PONG <nil>
}

func ExampleClient() {
	err := client.Set("key", "value", 0).Err()
	if err != nil {
		panic(err)
	}

	val, err := client.Get("key").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("key", val)

	val2, err := client.Get("key2").Result()
	if err == redis.Nil {
		fmt.Println("key2 does not exist")
	} else if err != nil {
		panic(err)
	} else {
		fmt.Println("key2", val2)
	}
	// Output: key value
	// key2 does not exist
})
```