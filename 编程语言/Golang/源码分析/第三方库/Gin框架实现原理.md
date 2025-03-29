# 高性能Gin框架原理学习教程

转载自腾讯程序员jiayan [腾讯技术工程](javascript:void(0);) 的[高性能Gin框架原理学习教程](https://mp.weixin.qq.com/s/T-rJjiyvp5dIdnGIVY-OcA)

![Image](https://mmbiz.qpic.cn/sz_mmbiz_gif/j3gficicyOvasVeMDmWoZ2zyN8iaSc6XWYj79H3xfgvsqK9TDxOBlcUa6W0EE5KBdxacd2Ql6QBmuhBJKIUS4PSZQ/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

作者：jiayan

> 工作中的部分项目使用到了gin框架，因此从源码层面学习一下其原理。

### **1. 概述**

[Gin](https://github.com/gin-gonic/gin)是一款高性能的Go语言Web框架，Gin的一些特性：

- 快速 基于 Radix 树的路由，小内存占用，没有反射，可预测的 API 性能。
- 支持中间件 传入的 HTTP 请求可以由一系列中间件和最终操作来处理，例如：Logger，Authorization，GZIP，最终操作 DB。
- 路由组 组织路由组非常方便，同时这些组可以无限制地嵌套而不会降低性能。
- 易用性 gin封装了标准库的底层能力。标准库暴露给开发者的函数参数是`(w http.ResponseWriter, req *http.Request)`，易用性低，需要直接从请求中读取数据、反序列化，响应时手动序列化、设置Content-Type、写响应内容，返回码等。使用gin可以帮我们更专注于业务处理。

### **2. 源码学习**

下面从一个简单的包含基础路由和路由组路由的demo开始分析：

```
func main() {
 // 初始化
 mux := gin.Default()

 // 设置全局通用handlers，这里是设置了engine的匿名成员RouterGroup的Handlers成员
 mux.Handlers = []gin.HandlerFunc{
  func(c *gin.Context) {
   log.Println("log 1")
   c.Next()
  },
  func(c *gin.Context) {
   log.Println("log 2")
   c.Next()
  },
 }

 // 绑定/ping 处理函数
 mux.GET("/ping", func(c *gin.Context) {
  c.String(http.StatusOK, "ping")
 })
 mux.GET("/pong", func(c *gin.Context) {
  c.String(http.StatusOK, "pong")
 })
 mux.GET("/ping/hello", func(c *gin.Context) {
  c.String(http.StatusOK, "ping hello")
 })

 mux.GET("/about", func(c *gin.Context) {
  c.String(http.StatusOK, "about")
 })

 // system组
 system := mux.Group("system")
 // system->auth组
 systemAuth := system.Group("auth")
 {
  // 获取管理员列表
  systemAuth.GET("/addRole", func(c *gin.Context) {
   c.String(http.StatusOK, "system/auth/addRole")
  })
  // 添加管理员
  systemAuth.GET("/removeRole", func(c *gin.Context) {
   c.String(http.StatusOK, "system/auth/removeRole")
  })
 }
 // user组
 user := mux.Group("user")
 // user->auth组
 userAuth := user.Group("auth")
 {
  // 登陆
  userAuth.GET("/login", func(c *gin.Context) {
   c.String(http.StatusOK, "user/auth/login")
  })
  // 注册
  userAuth.GET("/register", func(c *gin.Context) {
   c.String(http.StatusOK, "user/auth/register")
  })
 }

 mux.Run("0.0.0.0:8080")
}
```

#### **2.1 初始化Gin（Default函数）**

初始化步骤主要是初始化engine与加载两个默认的中间件：

```
func Default(opts ...OptionFunc) *Engine {
    debugPrintWARNINGDefault()
 // 初始化engine实例
    engine := New()
 // 默认加载log & recovery中间件
    engine.Use(Logger(), Recovery())
    return engine.With(opts...)
}
```

##### **2.1.1 初始化engine**

engine是gin中的核心对象，gin通过 Engine 对象来定义服务路由信息、组装插件、运行服务，是框架的核心发动机，整个 Web 服务的都是由它来驱动的 关键字段:

- RouterGroup: RouterGroup 是对路由树的包装，所有的路由规则最终都是由它来进行管理。RouteGroup 对象里面还会包含一个 Engine 的指针，可以调用engine的addRoute函数。
- trees: 基于前缀树实现。每个节点都会挂接若干请求处理函数构成一个请求处理链 HandlersChain。当一个请求到来时，在这棵树上找到请求 URL 对应的节点，拿到对应的请求处理链来执行就完成了请求的处理。
- addRoute: 用于添加 URL 请求处理器，它会将对应的路径和处理器挂接到相应的请求树中。
- RouterGroup: 内部有一个前缀路径属性(basePath)，它会将所有的子路径都加上这个前缀再放进路由树中。有了这个前缀路径，就可以实现 URL 分组功能。

```
func New(opts ...OptionFunc) *Engine {
    debugPrintWARNINGNew()
    engine := &Engine{
       RouterGroup: RouterGroup{
          Handlers: nil,
      // 默认的basePath为/，绑定路由时会用到此参数来计算绝对路径
          basePath: "/",
          root:     true,
       },
       FuncMap:                template.FuncMap{},
       RedirectTrailingSlash:  true,
       RedirectFixedPath:      false,
       HandleMethodNotAllowed: false,
       ForwardedByClientIP:    true,
       RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
       TrustedPlatform:        defaultPlatform,
       UseRawPath:             false,
       RemoveExtraSlash:       false,
       UnescapePathValues:     true,
       MaxMultipartMemory:     defaultMultipartMemory,
       trees:                  make(methodTrees, 0, 9),
       delims:                 render.Delims{Left: "{{", Right: "}}"},
       secureJSONPrefix:       "while(1);",
       trustedProxies:         []string{"0.0.0.0/0", "::/0"},
       trustedCIDRs:           defaultTrustedCIDRs,
    }
    engine.RouterGroup.engine = engine
 // 池化gin核心context对象，有新请求来时会使用到该池
    engine.pool.New = func() any {
       return engine.allocateContext(engine.maxParams)
    }
    return engine.With(opts...)
}
```

##### **2.1.2 初始化中间件**

###### **Engine.Use函数**

Engine.Use函数用于将中间件添加到当前的路由上，位于gin.go中，代码如下：

```
// Use attaches a global middleware to the router. ie. the middleware attached though Use() will be
// included in the handlers chain for every single request. Even 404, 405, static files...
// For example, this is the right place for a logger or error management middleware.
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
 engine.RouterGroup.Use(middleware...)
 engine.rebuild404Handlers()
 engine.rebuild405Handlers()
 return engine
}
```

###### **RouterGroup.Use函数**

实际上，还需要进一步调用`engine.RouterGroup.Use(middleware...)`完成实际的中间件注册工作，函数位于gin.go中，代码如下：

```
// Use adds middleware to the group, see example code in GitHub.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
 group.Handlers = append(group.Handlers, middleware...)
 return group.returnObj()
}
```

实际上就是把中间件(本质是一个函数）添加到HandlersChain类型（实质上为数组`type HandlersChain []HandlerFunc`）的group.Handlers中。

###### **HandlerFunc**

```
type HandlerFunc func(*Context)
```

如果需要实现一个中间件，那么需要实现该类型，函数参数只有*Context。

###### **gin.Context**

1. 贯穿一个 http 请求的所有流程，包含全部上下文信息。
2. 提供了很多内置的数据绑定和响应形式，JSON、HTML、Protobuf 、MsgPack、Yaml 等，它会为每一种形式都单独定制一个渲染器
3. engine的 `ServeHTTP` 函数，在响应一个用户的请求时，都会先从临时对象池中取一个context对象。使用完之后再放回临时对象池。为了保证并发安全，如果在一次请求新起一个协程，那么一定要copy这个context进行参数传递。

```
type Context struct {
    writermem responseWriter
    Request   *http.Request  // 请求对象
    Writer    ResponseWriter // 响应对象
    Params   Params  // URL 匹配参数
    handlers HandlersChain // // 请求处理链
    ...
}
```

#### **2.2 绑定处理函数到对应的HttpMethod上**

##### **2.2.1 普通路由实现**

调用绑定函数：

```
mux.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "ping")
})
```

函数实际上走到了engine对象的匿名成员RouterGroup的handle函数中

```
// POST is a shortcut for router.Handle("POST", path, handlers). 
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodPost, relativePath, handlers)
}

// GET is a shortcut for router.Handle("GET", path, handlers). 
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodGet, relativePath, handlers)
}

// DELETE is a shortcut for router.Handle("DELETE", path, handlers). 
func (group *RouterGroup) DELETE(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodDelete, relativePath, handlers)
}

// PATCH is a shortcut for router.Handle("PATCH", path, handlers). 
func (group *RouterGroup) PATCH(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodPatch, relativePath, handlers)
}

// PUT is a shortcut for router.Handle("PUT", path, handlers). 
func (group *RouterGroup) PUT(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodPut, relativePath, handlers)
}
```

绑定逻辑：

```
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    // 使用相对路径与路由组basePath 计算绝对路径
    absolutePath := group.calculateAbsolutePath(relativePath)
 // 将函数参数中的 "处理函数" handlers与本路由组已有的Handlers组合起来，作为最终要执行的完整handlers列表
    handlers = group.combineHandlers(handlers)
 // routerGroup会存有engine对象的引用，调用engine的addRoute将绝对路径与处理函数列表绑定起来
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
```

从源码中 `handlers = group.combineHandlers(handlers)` 可以看出我们也可以给gin设置一些全局通用的handlers，这些handlers会绑定到所有的路由方法上，如下：

```
// 设置全局通用handlers，这里是设置了engine的匿名成员RouterGroup的Handlers成员
mux.Handlers = []gin.HandlerFunc{
    func(c *gin.Context) {
       log.Println("log 1")
       c.Next()
    },
    func(c *gin.Context) {
       log.Println("log 2")
       c.Next()
    },
}
```

addRoute函数：

```
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    assert1(path[0] == '/', "path must begin with '/'")
    assert1(method != "", "HTTP method can not be empty")
    assert1(len(handlers) > 0, "there must be at least one handler")

    debugPrintRoute(method, path, handlers)
    
 // 每个HTTP方法（如：GET，POST）的路由信息都各自由一个树结构来维护，该树结构的模型与函数实现位于gin/tree.go中，此处不再继续展开。不同http方法的树根节点组成了 engine.trees 这个数组
 // 从engine的路由树数组中遍历找到该http方法对应的路由树的根节点
    root := engine.trees.get(method)
    if root == nil {
    // 如果根节点不存在，那么新建根节点
       root = new(node)
       root.fullPath = "/"
    // 将根节点添加到路由树数组中
       engine.trees = append(engine.trees, methodTree{method: method, root: root})
 }
 // 调用根节点的addRoute函数，将绝对路径与处理函数链绑定起来
    root.addRoute(path, handlers)
 
 ...
}
// 路由树数组数据结构
type methodTree struct {
    method string     root   *node 
}

type methodTrees []methodTree
```

##### **2.2.2 路由组的实现**

```
// system组
system := mux.Group("system")
// system->auth组
systemAuth := system.Group("auth")
{
 // 获取管理员列表
 systemAuth.GET("/addRole", func(c *gin.Context) {
  c.String(http.StatusOK, "system/auth/addRole")
 })
 // 添加管理员
 systemAuth.GET("/removeRole", func(c *gin.Context) {
  c.String(http.StatusOK, "system/auth/removeRole")
 })
}
// user组
user := mux.Group("user")
// user->auth组
userAuth := user.Group("auth")
{
 // 登陆
 userAuth.GET("/login", func(c *gin.Context) {
  c.String(http.StatusOK, "user/auth/login")
 })
 // 注册
 userAuth.GET("/register", func(c *gin.Context) {
  c.String(http.StatusOK, "user/auth/register")
 })
} 
```

Group函数会返回一个新的RouterGroup对象，每一个RouterGroup都会基于原有的RouterGroup而生成：

- 这里demo中生成的systemGroup，会基于根routerGroup（basePath:"/"）而生成，那么systemGroup的basePath为 `joinPaths("/" + "system")`

- 基于systemGroup生成的systemAuthGroup，basePath为sysetmGroup的basePath: "/system" 与函数参数中"auth"组成 `join("/system", "auth")`，

  ```
  // Group creates a new router group. You should add all the routes that have common middlewares or the same path prefix. // For example, all the routes that use a common middleware for authorization could be grouped. 
  func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
      return &RouterGroup{
      // 新routerGroup的handlers由原routerGroup的handlers和本次传入的handlers组成
         Handlers: group.combineHandlers(handlers),
      // 新routerGroup的basePath由原routerGroup的basePath + relativePath 计算出来
         basePath: group.calculateAbsolutePath(relativePath),
      // 新routerGroup依旧持久全局engine对象
         engine:   group.engine,
      }
  }
  
  // 合并handlers，将group的handlers与传入参数的handlers合并
  func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
      finalSize := len(group.Handlers) + len(handlers)
      assert1(finalSize < int(abortIndex), "too many handlers")
      mergedHandlers := make(HandlersChain, finalSize)
      copy(mergedHandlers, group.Handlers)
      copy(mergedHandlers[len(group.Handlers):], handlers)
      return mergedHandlers
  }
  
  // 拼装routerGroup的basePath与传入的相对路径relativePath
  func (group *RouterGroup) calculateAbsolutePath(relativePath string) string {
      return joinPaths(group.basePath, relativePath)
  }
  ```

  需要注意的一点是，每个routerGroup都持有全局engine对象，调用Group()生成新RouterGroup后，再调用GET, POST..绑定路由时，依旧会使用全局engine对象的handle方法，最终会走到：

  ```
  group.engine.addRoute(httpMethod, absolutePath, handlers)
  ```

  根据http方法找到对应的路由树根节点，然后再更新路由树。

#### **2.3. 启动Gin**

##### **2.3.1 Engine.Run函数**

```
// Run attaches the router to a http.Server and starts listening and serving HTTP requests. // It is a shortcut for http.ListenAndServe(addr, router) // Note: this method will block the calling goroutine indefinitely unless an error happens. 
func (engine *Engine) Run(addr ...string) (err error) {
 ...
    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    err = http.ListenAndServe(address, engine.Handler())
    return 
}
```

可以看到，最核心的监听与服务实质上是调用Go语言内置库net/http的`http.ListenAndServe`函数实现的。

##### **2.3.2 net/http的ListenAndServe函数**

```
// ListenAndServe listens on the TCP network address addr and then calls
// Serve with handler to handle requests on incoming connections.
// Accepted connections are configured to enable TCP keep-alives.
//
// The handler is typically nil, in which case the DefaultServeMux is used.
//
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
 server := &Server{Addr: addr, Handler: handler}
 return server.ListenAndServe()
}


// ListenAndServe listens on the TCP network address srv.Addr and then
// calls Serve to handle requests on incoming connections.
// Accepted connections are configured to enable TCP keep-alives.
//
// If srv.Addr is blank, ":http" is used.
//
// ListenAndServe always returns a non-nil error. After Shutdown or Close,
// the returned error is ErrServerClosed.
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
       return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
       addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
       return err
    }
    return srv.Serve(ln)
}
```

ListenAndServe函数实例化Sever，调用其`ListenAndServe`函数实现监听与服务功能。在gin中，Engine对象以**Handler接口**的对象的形式被传入给了net/http库的Server对象，作为后续Serve对象处理网络请求时调用的函数。

##### **2.3.3 标准库中的Handler接口**

net/http的Server结构体类型中有一个Handler接口类型的Handler。

```
// A Server defines parameters for running an HTTP server.
// The zero value for Server is a valid configuration.
type Server struct {
 // Addr optionally specifies the TCP address for the server to listen on,
 // in the form "host:port". If empty, ":http" (port 80) is used.
 // The service names are defined in RFC 6335 and assigned by IANA.
 // See net.Dial for details of the address format.
 Addr string

 Handler Handler // handler to invoke, http.DefaultServeMux if nil
    // ...
}


//Handler接口有且只有一个函数，任何类型，只需要实现了该ServeHTTP函数，就实现了Handler接口，就可以用作Server的Handler，供HTTP处理时调用。
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

##### **2.3.4 标准库中的Server.Serve函数**

`Server.Serve`函数用于监听、接受和处理网络请求，代码如下：

```
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each. The service goroutines read requests and
// then call srv.Handler to reply to them.
//
// HTTP/2 support is only enabled if the Listener returns *tls.Conn
// connections and they were configured with "h2" in the TLS
// Config.NextProtos.
//
// Serve always returns a non-nil error and closes l.
// After Shutdown or Close, the returned error is ErrServerClosed.
func (srv *Server) Serve(l net.Listener) error {
 //...
 for {
  rw, err := l.Accept()
  if err != nil {
   select {
   case <-srv.getDoneChan():
    return ErrServerClosed
   default:
   }
   if ne, ok := err.(net.Error); ok && ne.Temporary() {
    if tempDelay == 0 {
     tempDelay = 5 * time.Millisecond
    } else {
     tempDelay *= 2
    }
    if max := 1 * time.Second; tempDelay > max {
     tempDelay = max
    }
    srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
    time.Sleep(tempDelay)
    continue
   }
   return err
  }
  connCtx := ctx
  if cc := srv.ConnContext; cc != nil {
   connCtx = cc(connCtx, rw)
   if connCtx == nil {
    panic("ConnContext returned nil")
   }
  }
  tempDelay = 0
  c := srv.newConn(rw)
  c.setState(c.rwc, StateNew, runHooks) // before Serve can return
  go c.serve(connCtx)
 }
}
```

在`Server.Serve`函数的实现中，启动了一个无条件的for循环持续监听、接受和处理网络请求，主要流程为：

1. **接受请求**：`l.Accept()`调用在无请求时保持阻塞，直到接收到请求时，接受请求并返回建立的连接；
2. **处理请求**：启动一个goroutine，使用标准库中conn连接对象的serve函数进行处理（`go c.serve(connCtx)`）；

##### **2.3.5 conn.serve函数**

```
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr()) 
    ... 
    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx() 
    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)  
    for {
        // 读取请求
        w, err := c.readRequest(ctx) 
        ... 
        // 根据请求路由调用处理器处理请求
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        if c.hijacked() {
            return
        }
        w.finishRequest() 
        ...
    }
}
```

一个连接建立之后，该连接中所有的请求都将在这个协程中进行处理，直到连接被关闭。在 for 循环里面会循环调用 readRequest 读取请求进行处理。可以在第16行看到请求处理是通过调用 serverHandler结构体的ServeHTTP函数 进行的。

```
// serverHandler delegates to either the server's Handler or
// DefaultServeMux and also handles "OPTIONS *" requests.
type serverHandler struct {
    srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
       handler = DefaultServeMux
    }
    if !sh.srv.DisableGeneralOptionsHandler && req.RequestURI == "*" && req.Method == "OPTIONS" {
       handler = globalOptionsHandler{}
    }

    handler.ServeHTTP(rw, req)
}
```

可以看到上面第八行 `handler := sh.srv.Handler`，在gin框架中，sh.srv.Handler其实就是engine.Handler()。

```
func (engine *Engine) Handler() http.Handler {
    if !engine.UseH2C {
       return engine
    }
 // 使用了标准库的h2c(http2 client)能力，本质还是使用了engine对象自身的ServeHTTP函数
    h2s := &http2.Server{}
    return h2c.NewHandler(engine, h2s)
}
```

engine.Handler()函数使用了http2 server的能力，实际的逻辑处理还是依赖engine自身的ServeHTTP()函数。

##### **2.3.6 Gin的Engine.ServeHTTP函数**

gin在gin.go中实现了`ServeHTTP`函数，代码如下：

```
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
 c := engine.pool.Get().(*Context)
 c.writermem.reset(w)
 c.Request = req
 c.reset()

 engine.handleHTTPRequest(c)

 engine.pool.Put(c)
}
```

主要步骤为：

1. **建立连接上下文**：从全局engine的缓存池中提取上下文对象，填入当前连接的`http.ResponseWriter`实例与`http.Request`实例；
2. **处理连接**：以上下文对象的形式将连接交给函数处理，由`engine.handleHTTPRequest(c)`封装实现了；
3. **回收连接上下文**：处理完毕后，将上下文对象回收进缓存池中。

gin中对每个连接都需要的上下文对象进行缓存化存取，通过缓存池节省高并发时上下文对象频繁创建销毁造成内存频繁分配与释放的代价。

##### **2.3.7 Gin的Engine.handleHTTPRequest函数**

`handleHTTPRequest`函数封装了对请求进行处理的具体过程，位于gin/gin.go中，代码如下:

```
func (engine *Engine) handleHTTPRequest(c *Context) {
 httpMethod := c.Request.Method
 rPath := c.Request.URL.Path
 unescape := false
 if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
  rPath = c.Request.URL.RawPath
  unescape = engine.UnescapePathValues
 }

 if engine.RemoveExtraSlash {
  rPath = cleanPath(rPath)
 }

 // Find root of the tree for the given HTTP method
 // 根据http方法找到路由树数组中对应的路由树根节点
 t := engine.trees
 for i, tl := 0, len(t); i < tl; i++ {
  if t[i].method != httpMethod {
   continue
  }
        // 找到了根节点
  root := t[i].root
  // Find route in tree
  // 使用请求参数和请求路径从路由树中找到对应的路由节点
  value := root.getValue(rPath, c.params, unescape)
  if value.params != nil {
   c.Params = *value.params
  }
  
  // 调用context的Next()函数，实际上也就是调用路由节点的handlers方法链
  if value.handlers != nil {
   c.handlers = value.handlers
   c.fullPath = value.fullPath
   c.Next()
   c.writermem.WriteHeaderNow()
   return
  }
  // ...
  break
 }

 if engine.HandleMethodNotAllowed {
  for _, tree := range engine.trees {
   if tree.method == httpMethod {
    continue
   }
   if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
    c.handlers = engine.allNoMethod
    serveError(c, http.StatusMethodNotAllowed, default405Body)
    return
   }
  }
 }
 c.handlers = engine.allNoRoute
 serveError(c, http.StatusNotFound, default404Body)
}
```

`Engine.handleHTTPRequest`函数的主要逻辑位于中间的for循环中，主要为：

1. 遍历查找`engine.trees`以找出当前请求的HTTP Method对应的处理树；
2. 从该处理树中，根据当前请求的路径与参数查询出对应的处理函数`value`；
3. 将查询出的处理函数链（`gin.HandlerChain`）写入当前连接上下文的`c.handlers`中；
4. 执行`c.Next()`，调用handlers链上的下一个函数（中间件/业务处理函数），开始形成LIFO的函数调用栈；
5. 待函数调用栈全部返回后，`c.writermem.WriteHeaderNow()`根据上下文信息，将HTTP状态码写入响应头。

### **4. Gin 路由树**

gin的路由树源码上面没有展开，实际上就是实现了radix tree的数据结构：

- trie tree: 前缀树，一颗多叉树，用于字符串搜索，每个树节点存储一个字符，从根节点到任意一个叶子结点串起来就是一个字符串。

- radix tree: 基数树(压缩前缀树)，对空间进一步压缩，从上往下提取公共前缀，非公共部分存到子节点，这样既节省了空间，同时也提高了查询效率(左边字符串sleep查询需要5步, 右边只需要3步)。

  

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/j3gficicyOvatGB3ZgrcuTIqjAJAPTORC9QUZPYzbmI5QbGUHYLENDPaBORgOf11zmLwAbAkMM4AKMA7rAoB8NqA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

实际看下demo中的代码会生成的radix tree：

```
mux.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "ping")
})
mux.GET("/pong", func(c *gin.Context) {
    c.String(http.StatusOK, "pong")
})
mux.GET("/ping/hello", func(c *gin.Context) {
    c.String(http.StatusOK, "ping hello")
})

mux.GET("/about", func(c *gin.Context) {
    c.String(http.StatusOK, "about")
})
```

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/j3gficicyOvatGB3ZgrcuTIqjAJAPTORC9iarKDb5jSXQzIUkWEU7BgGcYZXdcF9hvs4CRo5ibjpTIt0W9hJxkgZVQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

实际上源码中的基数树还涉及可变参数路由的处理，会更复杂一些。



### **5. Gin 中间件实现**

1. 每个路由节点都会挂载一个函数链，链的前面部分是插件函数，后面部分是业务处理函数。

2. 在 gin 中插件和业务处理函数形式是一样的，都是 `func(*Context)`。当我们定义路由时，gin 会将插件函数和业务处理函数合并在一起形成一个链条结构。gin 在接收到客户端请求时，找到相应的处理链，构造一个 Context 对象，再调用它的 Next() 函数就正式进入了请求处理的全流程。

3. 惯用法: 可以通过在处理器中调用`c.Next()`提前进入下一个处理器，待其执行完后再返回到当前处理器，这种比较适合需要对请求做前置和后置处理的场景，如请求执行时间统计，请求的前后日志等。

4. 有时候我们可能会希望，某些条件触发时直接返回，不再继续后续的处理操作。Context提供了`Abort`方法帮助我们实现这样的目的。原理是将 Context.index 调整到一个比较大的数字，gin中要求一个路由的全部处理器个数不超过63，每次执行一个处理器时，会先判断index是否超过了这个限制，如果超过了就不会执行。

   ```
   // Next should be used only inside middleware. // It executes the pending handlers in the chain inside the calling handler. // See example in GitHub. 
   func (c *Context) Next() {
       c.index++
       for c.index < int8(len(c.handlers)) {
          if c.handlers[c.index] == nil {
             continue        }
          c.handlers[c.index](c)
          c.index++
       }
   }
   
   func (c *Context) Abort() {
       c.index = abortIndex 
   }  
   const abortIndex int8 = math.MaxInt8 >> 1
   ```

### **总结**

- 从源码中可以看到gin与fasthttp库基于自身的http库来实现网络层不同，gin是基于标准http库的能力来构建的自己的网络层的，所以性能应该是与原生http库接近的。
- gin通过实现Go语言提供的接口快捷地接入Go的内置库功能，使得上层应用与底层实现之间互不依赖。
- gin在性能上针对HTTP Web框架常见的高并发问题进行了优化，例如：通过上下文对象的缓存池节省连接高并发时内存频繁申请与释放的代价
- gin的压缩前缀树数据结构设计，不同于标准库中基于map的路由，实现了高效的路由查找（匹配时间复杂度是O(k)），另外可以支持模糊匹配、参数匹配等功能，具有更高的灵活性和可扩展性。
- gin采用中间件（Middleware）来处理请求和响应，可以实现各种功能，例如日志记录、权限验证、请求限流等。

### **参考资料**

- [一文说透 Go 语言 HTTP 标准库](https://www.luozhiyun.com/archives/561)
- [终于搞懂了前缀树](https://juejin.cn/post/7263858148276142135)