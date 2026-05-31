## 一 Gin对象的构建

Gin框架是基于golang官方的http包搭建起来的，http包最重要的实现是：
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
利用该方法，以及参数中的Handler接口，实现Gin的Engine，Context：
```go
package engine

import "net/http"

type Engine struct {

}

func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {

}
```

context对象其实就是对ServeHTTP方法参数的封装，因为这2个参数在web开发中一个完整请求链中都会用到：
```go
type Context struct {
    writermem   responseWriter
    Request     *http.Request
    Writer      ResponseWriter
}

type responseWriter struct {
    http.ResponseWriter
    size        int
    status      int
}
```

这里多了一个属性 `writermem`，如果仅仅只从功能上考虑，这里有了Request、Writer已经足够使用了，但是框架需要应对多变的返回数据情况，必须对其进行封装，比如：
```go
type ResponseWriter interface {

    http.ResponseWriter

    Pusher() http.Pusher

    Status() int
    Size() int
    WriteString(string) (int, error)
    Written() bool
    WriteHeaderNow()
}
```

这里对外暴露的是接口RespnserWriter，内部的`http.ResponseWriter`实现原生的ResponseWriter接口，在reset()的时候进行拷贝即可：
```go
func (c *Context) reset() {
    c.Writer = &c.writermem
    c.Params = c.Params[0:0]
    c.handlers = nil
    c.index = -1
    c.Keys = nil
    c.Errors = c.Errors[0:0]
    c.Accepted = nil
}
```

这样做能够更好的符合面向接口编程的概念。  

Context也可以通过对象池复用提升性能：
```go
type Engine struct {
    pool             sync.Pool
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()

    engine.handleHTTPRequest(c)

    engine.pool.Put(c)
}
```

紧接着就可以在Context的基础上实现其大量的常用方法了：
```go
func (c *Context) Param(key string) string{
	return ""
}

func (c *Context) Query(key string) string {
	return ""
}

func (c *Context) DefaultQuery(key, defaultValue string) string {
	return ""
}


func (c *Context) PostFormArray(key string) []string{
	return nil
}
```