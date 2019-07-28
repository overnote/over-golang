## 一 gin.Engine

Engine是框架的入口，是gin框架的核心，通过Engine对象来定义服务路由信息、组装插件、运行服务。不过Engine的本质只是对内置HTTP服务的包装。  

gin.Default() 函数会生成一个默认的 Engine 对象，里面包含了 2 个默认的常用插件，分别是 Logger 和 Recovery，Logger 用于输出请求日志，Recovery 确保单个请求发生 panic 时记录异常堆栈日志，输出统一的错误响应。  

```go
func Default() *Engine {
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

## 二 HTTP错误  

当 URL 请求对应的路径不能在路由树里找到时，就需要处理 404 NotFound 错误。当 URL 的请求路径可以在路由树里找到，但是 Method 不匹配，就需要处理 405 MethodNotAllowed 错误。Engine 对象为这两个错误提供了处理器注册的入口。  

```go
func (engine *Engine) NoMethod(handlers ...HandlerFunc)
func (engine *Engine) NoRoute(handlers ...HandlerFunc)
```

异常处理器和普通处理器一样，也需要和插件函数组合在一起形成一个调用链。如果没有提供异常处理器，Gin 就会使用内置的简易错误处理器。  

注意这两个错误处理器是定义在 Engine 全局对象上，而不是 RouterGroup。对于非 404 和 405 错误，需要用户自定义插件来处理。对于 panic 抛出来的异常需要也需要使用插件来处理。  

## 三 HTTPS

Gin 不支持 HTTPS，官方建议是使用 Nginx 来转发 HTTPS 请求到 Gin。  
 

