## 一 gin框架初识

gin框架中的路由是基于httprouter开发的。  

httprouter地址：https://github.com/julienschmidt/httprouter

gin提供了Martini风格API，但是速度比Martini快40倍。gin非常轻量，使用方便简洁。  

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

	r.Run()			//默认位于0.0.0.0:8080，可传入参数 ":3030"

}
```

## 二 参数获取

get请求参数获取：

```
c.Query("username")
c.QueryDefault("username","lisi")       //如果username为空，则赋值为lisi
```

路由地址为：/user/:name/:pass，获取参数：
```go
name := c.Param("name")
```

Post请求参数获取：
```go
name := c.PostForm("name")
```

## 三 上传文件
```go
func (c *gin.Context) {
	file, err := c.FormFile("file")
	if (err != nil) {
		c.JSON(http.StatusInternalServerError, gin.H{
			"msg": err.Error(),
		})
		return
	}
	dst := fmt.Sprintf("/uploads/&s", file.Filename)
	c.SavaeUpLoadedFile(file, dst)
	c.JSON(http.StatusOK, gin.H{
		"msg":"ok",
	})
}
```

## 四 路由分组

访问路径是：`/user/login` 和 `/user/signin`
```go
package main

import (
	"github.com/gin-gonic/gin"
)

func login(c *gin.Context) {
	c.JSON(300, gin.H{
		"msg": "login",
	})
}

func signin(c *gin.Context) {
	c.JSON(300, gin.H{
		"msg": "signin",
	})
}

func main() {

	router := gin.Default()

	user := router.Group("/user")
	{
		user.GET("/login", login)
		user.GET("/signin", signin)
	}

	router.Run(":3000")
}
```

## 五 参数绑定

参数绑定利用反射机制，自动提起querystring，form表单，json，xml等参数到结构体中，可以极大提升开发效率。  

```go
package main

import (
	"net/http"
	"github.com/gin-gonic/gin"
	"fmt"
)

type User struct {
	Username string `form:"username" json:"username" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

func login(c *gin.Context) {

	var user User

	fmt.Println(c.PostForm("username"))
	fmt.Println(c.PostForm("password"))

	if err := c.ShouldBind(&user); err == nil {
		c.JSON(http.StatusOK, gin.H{
			"username": user.Username,
			"password": user.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}


func main() {

	router := gin.Default()

	router.POST("/login", login)

	router.Run(":3000")
}
```

## 六 返回结果

返回JSON类型的结果：
```go
c.JSON(200,gin.H{"msg":"OK"})
c.JSON(200,结构体)
```

渲染模板：
```go
	router.LoadHTMLGlob("templates/**/*")

	router.GET("/test/index", func(c *gin.Context){
		c.HTML(http.StatusOK, "test/index.tmpl", gin.H{
			"msg": "test",
		})
	})
```

模板文件：index.tmpl
```html
{{define "test/index.tmpl"}}
<html>
<head>
</head>
<body>
test...
{{.}}
-----
{{.msg}}
</body>
</html>
{{end}}
```

注意事项：不要使用编辑器的run功能，会出现路径错误，推荐使用命令build，项目路径分配如下：
![](/images/Golang/框架-01.png)

## 七 静态文件

静态化当前目录下static文件夹：
```go
	router := gin.Default()

	router.Static("/static", "./static")

	router.Run(":3000")
```
注意：同样推荐使用go build，不要使用开发工具的run功能。

## 八 中间件

gin框架允许在处理请求时，加入用户自己的钩子函数，该钩子函数即中间件。  

```go
package main

import (
	"github.com/gin-gonic/gin"
	"fmt"
)

func Test() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set("before", "123")
		fmt.Println("执行了中间件")
		c.Next()
		fmt.Println("执行了next")
	}
}

func main() {

	router := gin.Default()

	router.Use(Test())

	router.GET("/", func(c *gin.Context){
		before := c.MustGet("before").(string)
		fmt.Println("before:", before)
	})

	router.Run(":3000")
}

```
依次输出：
```
执行了中间件
before: 123
执行了next
```

注意：next后不要设置一些值，无法获取
