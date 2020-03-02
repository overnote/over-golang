## 一 Gin中间件

### 1.1 中间件的概念

gin框架允许在处理请求时，加入用户自己的钩子函数，该钩子函数即中间件。他的作用与Java中的拦截器，Node中的中间件相似。  

中间件需要返回`gin.HandlerFunc`函数，多个中间件通过Next函数来依次执行。  

### 1.2 入门使用案例

现在设计一个中间件，在每次路由函数执行前打印一句话，在上一节的项目基础上新建`middleware`文件夹，新建一个中间件文件`MyFmt.go`：
```go
package middleware

import (
	"fmt"
	"github.com/gin-gonic/gin"
)

// 定义一个中间件
func MyFMT() gin.HandlerFunc {
	return func(c *gin.Context) {
		host := c.Request.Host
		fmt.Printf("Before: %s\n",host)
		c.Next()
		fmt.Println("Next: ...")
	}
}
```

在路由函数中使用中间件：
```go
r.GET("/user/login", middleware.MyFMT(),  userLogin)
```

打印结果：
```
Before: localhost:8080
Next: ...
[GIN] 2019/07/28 - 16:28:16 | 200 |      266.33µs |             ::1 | GET      /user/login
```

### 1.2 中间件的详细使用方式

全局中间件：直接使用 `gin.Engine`结构体的`Use()`方法，中间件将会在项目的全局起作用。
```go
func InitRouter() *gin.Engine {

	r := gin.Default()

	// 全局中间件
	r.Use(middleware.MyFMT())

	// 路由模块化
	userRouter(r)
	orderRouter(r)

	return r
}
```

路由分钟中使用中间件：
```go
router := gin.New()
user := router.Group("user", gin.Logger(),gin.Recovery())
{
    user.GET("info", func(context *gin.Context) {

    })
    user.GET("article", func(context *gin.Context) {

    })
}
```

单个路由使用中间件(支持多个中间件的使用)：
```go
router := gin.New()
router.GET("/test",gin.Recovery(),gin.Logger(),func(c *gin.Context){
    c.JSON(200,"test")
})
```

### 1.3 内置中间件

Gin也内置了一些中间件，可以直接使用：
```go
func BasicAuth(accounts Accounts) HandlerFunc
func BasicAuthForRealm(accounts Accounts, realm string) HandlerFunc
func Bind(val interface{}) HandlerFunc //拦截请求参数并进行绑定
func ErrorLogger() HandlerFunc       //错误日志处理
func ErrorLoggerT(typ ErrorType) HandlerFunc //自定义类型的错误日志处理
func Logger() HandlerFunc //日志记录
func LoggerWithConfig(conf LoggerConfig) HandlerFunc
func LoggerWithFormatter(f LogFormatter) HandlerFunc
func LoggerWithWriter(out io.Writer, notlogged ...string) HandlerFunc
func Recovery() HandlerFunc
func RecoveryWithWriter(out io.Writer) HandlerFunc
func WrapF(f http.HandlerFunc) HandlerFunc //将http.HandlerFunc包装成中间件
func WrapH(h http.Handler) HandlerFunc //将http.Handler包装成中间件
```

## 二 请求的拦截与后置

中间件的最大作用就是拦截过滤请求，比如我们有些请求需要用户登录或者需要特定权限才能访问，这时候便可以中间件中做过滤拦截。  

下面三个方法中断请求后，直接返回200，但响应的body中不会有数据：
```go
func (c *Context) Abort()
func (c *Context) AbortWithError(code int, err error) *Error
func (c *Context) AbortWithStatus(code int)
func (c *Context) AbortWithStatusJSON(code int, jsonObj interface{})		// 中断后可以返回json数据
```

如果在中间件中调用gin.Context的Next()方法，则可以请求到达并完成业务处理后，再经过中间件后置拦截处理：
```go
func MyMiddleware(c *gin.Context){
    //请求前
    c.Next()
    //请求后
}
```