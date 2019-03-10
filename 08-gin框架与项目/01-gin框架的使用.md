## 一 gin框架初识

gin框架中的路由是基于httprouter开发的。  

httprouter地址：https://github.com/julienschmidt/httprouter

gin提供了Martini风格API，但是速度比Martini块40倍。gin非常轻量，使用简介。  

HelloWorld：
```go
package main

import (
	"github.com/gin-gonic/gin"
	"fmt"
)

func main() {

	r := gin.Default()	//Default返回一个默认https://github.com/golang/net.git路由引擎

	r.GET("/", func(c *gin.Context) {

		username := c.Query("username")

		fmt.Println(username)

		c.JSON(200, gin.H{
			"msg":"hello world",
		})
o
	})

	r.Run()			//默认位于0.0.0.0:8080

}
```

## 二 参数获取

get请求参数获取：

```
c.Query("username")
c.QueryDefault("username","lisi")       //如果username为空，则赋值为lisi
```

