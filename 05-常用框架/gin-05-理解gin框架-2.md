## 三 gin.Context

gin.Context内保存了请求的上下文信息，是所有请求处理器的入口参数：
```go
type HandlerFunc func(*Context)

type Context struct {
  ...
  Request *http.Request // 请求对象
  Writer ResponseWriter // 响应对象
  Params Params // URL匹配参数
  ...
  Keys map[string]interface{} // 自定义上下文信息
  ...
}
```

Context 对象提供了非常丰富的方法用于获取当前请求的上下文信息，如果你需要获取请求中的 URL 参数、Cookie、Header 都可以通过 Context 对象来获取。这一系列方法本质上是对 http.Request 对象的包装：
```go
// 获取 URL 匹配参数  /book/:id
func (c *Context) Param(key string) string
// 获取 URL 查询参数 /book?id=123&page=10
func (c *Context) Query(key string) string
// 获取 POST 表单参数
func (c *Context) PostForm(key string) string
// 获取上传的文件对象
func (c *Context) FormFile(name string) (*multipart.FileHeader, error)
// 获取请求Cookie
func (c *Context) Cookie(name string) (string, error) 
...
```

Context 对象提供了很多内置的响应形式，JSON、HTML、Protobuf 、MsgPack、Yaml 等。它会为每一种形式都单独定制一个渲染器。通常这些内置渲染器已经足够应付绝大多数场景，如果你觉得不够，还可以自定义渲染器。
```go
func (c *Context) JSON(code int, obj interface{})
func (c *Context) Protobuf(code int, obj interface{})
func (c *Context) YAML(code int, obj interface{})
...
// 自定义渲染
func (c *Context) Render(code int, r render.Render)

// 渲染器通用接口
type Render interface {
    Render(http.ResponseWriter) error
    WriteContentType(w http.ResponseWriter)
}
```

所有的渲染器最终还是需要调用内置的 http.ResponseWriter（Context.Writer） 将响应对象转换成字节流写到套接字中。
```go
type ResponseWriter interface {
 // 容纳所有的响应头
 Header() Header
 // 写Body
 Write([]byte) (int, error)
 // 写Header
 WriteHeader(statusCode int)
}
```

## 四 插件与请求链

gin的插件机制中，函数链的尾部是业务处理，前面部分是插件函数。在 Gin 中插件和业务处理函数形式是一样的，都是 func(*Context)。当我们定义路由时，Gin 会将插件函数和业务处理函数合并在一起形成一个链条结构。
```go
type Context struct {
  ...
  index uint8 // 当前的业务逻辑位于函数链的位置
  handlers HandlersChain // 函数链
  ...
}

// 挨个调用链条中的处理函数
func (c *Context) Next() {
    c.index++
    for s := int8(len(c.handlers)); c.index < s; c.index++ {
        c.handlers[c.index](c)
    }
}
```

所以在业务代码中，一般一个处理函数时，路由节点也需要挂载一个函数链条。  

Gin 在接收到客户端请求时，找到相应的处理链，构造一个 Context 对象，再调用它的 Next() 方法就正式进入了请求处理的全流程。  

![](../images/go/gin-04.jpeg)  

