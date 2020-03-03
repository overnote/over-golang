## 一 请求流程梳理

首先从gin最开始的创建engine对象部分开始：
```go
router := gin.Default()
```

该方法返回了Engine结构体，常见属性有：
```go
type Engine struct {
    //路由组
    RouterGroup
    RedirectTrailingSlash bool
    RedirectFixedPath bool
    HandleMethodNotAllowed bool
    ForwardedByClientIP    bool
    AppEngine bool
    UseRawPath bool
    UnescapePathValues bool
    MaxMultipartMemory int64
    delims           render.Delims
    secureJsonPrefix string
    HTMLRender       render.HTMLRender
    FuncMap          template.FuncMap
    allNoRoute       HandlersChain
    allNoMethod      HandlersChain
    noRoute          HandlersChain
    noMethod         HandlersChain
    // 对象池 用来创建上下文context
    pool             sync.Pool
    //记录路由方法的 比如GET POST 都会是数组中的一个 每个方法对应一个基数树的一个root的node
    trees            methodTrees
}
```

Default方法其实就是创建了该对象，并添加了一些默认中间件：
```go
func Default() *Engine {
    debugPrintWARNINGDefault()
    engine := New()
    engine.Use(Logger(), Recovery())
    return engine
}
```

注意，这里Default方法内部调用了New方法，**该方法默认添加了路由组"/"**
```go
func New() *Engine {
    debugPrintWARNINGNew()
    engine := &Engine{
        RouterGroup: RouterGroup{
            Handlers: nil,
            basePath: "/",
            root:     true,
        },
        FuncMap:                template.FuncMap{},
        RedirectTrailingSlash:  true,
        RedirectFixedPath:      false,
        HandleMethodNotAllowed: false,
        ForwardedByClientIP:    true,
        AppEngine:              defaultAppEngine,
        UseRawPath:             false,
        UnescapePathValues:     true,
        MaxMultipartMemory:     defaultMultipartMemory,
        trees:                  make(methodTrees, 0, 9),
        delims:                 render.Delims{Left: "{{", Right: "}}"},
        secureJsonPrefix:       "while(1);",
    }
    engine.RouterGroup.engine = engine
    engine.pool.New = func() interface{} {
        return engine.allocateContext()
    }
    return engine
}
```

context对象存储了上下文信息，包括：engine指针、request对象，responsewriter对象等，context在请求一开始就被创建，且贯穿整个执行过程，包括中间件、路由等等：
```go
type Context struct {
    writermem responseWriter
    Request   *http.Request
    Writer    ResponseWriter

    Params   Params
    handlers HandlersChain
    index    int8

    engine *Engine

    // Keys is a key/value pair exclusively for the context of each request.
    Keys map[string]interface{}

    // Errors is a list of errors attached to all the handlers/middlewares who used this context.
    Errors errorMsgs

    // Accepted defines a list of manually accepted formats for content negotiation.
    Accepted []string
}
```

接下来是Use方法：
```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    //调用routegroup的use方法
    engine.RouterGroup.Use(middleware...)
    engine.rebuild404Handlers()
    engine.rebuild405Handlers()
    return engine
}

func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    //为group的handlers添加中间件 
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
```

最后到达最终路由，有GET,POST等多种方法，但是每个方法的处理方式都是相同的，即把group和传入的handler合并，计算出路径存入tree中等待客户端调用：
```go
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    //调用get方法
    return group.handle("GET", relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    //计算路径地址，比如group地址是 router.Group("/api")
    //结果为/api/test/ 就是最终计算出来的结果 使用path.join 方法拼接 其中加了一些判断
    absolutePath := group.calculateAbsolutePath(relativePath)
    //把group中的handler和传入的handler合并 
    handlers = group.combineHandlers(handlers)
    //把方法 路径 和处理方法作为node 加入到基数树种，基数树在下次单独学习分析
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    finalSize := len(group.Handlers) + len(handlers)
    if finalSize >= int(abortIndex) {
        panic("too many handlers")
    }
    mergedHandlers := make(HandlersChain, finalSize)
    copy(mergedHandlers, group.Handlers)
    copy(mergedHandlers[len(group.Handlers):], handlers)
    return mergedHandlers
}
```

run方法则是启动服务，在http包中会有一个for逻辑不停的监听端口：
```go
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()

    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    err = http.ListenAndServe(address, engine)
    return
}

func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

## 二 书写类似源码

