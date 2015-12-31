# ozzo-routing

[![GoDoc](https://godoc.org/github.com/go-ozzo/ozzo-routing?status.png)](http://godoc.org/github.com/go-ozzo/ozzo-routing)
[![Build Status](https://travis-ci.org/go-ozzo/ozzo-routing.svg?branch=master)](https://travis-ci.org/go-ozzo/ozzo-routing)
[![Coverage](http://gocover.io/_badge/github.com/go-ozzo/ozzo-routing)](http://gocover.io/github.com/go-ozzo/ozzo-routing)

## 其他语言

[简体中文](/docs/README-zh-CN.md) [Русский](/docs/README-ru.md)

## 说明

ozzo-routing is a Go package that supports request routing and processing for Web applications.
It has the following features:<br>
ozzo-routing 是一个用于帮助 Web 应用处理 HTTP 请求的路由和处理的 Go 包。
它包含以下功能:

* middleware pipeline architecture, similar to that in the [Express framework](http://expressjs.com).
* 中间件管线(middleware pipeline)架构，类似 [Express 框架](http://expressjs.com)
* highly extensible through pluggable handlers (middlewares)
* 通过自由组合的处理句柄（中间件）实现高度可扩展性
* modular code organization through route grouping
* 通过路由分组实现了模块化的代码组织
* URL path parameters
* 支持 URL 路径参数
* static file server
* 静态文件服务器
* error handling
* 错误处理
* compatible with `http.Handler` and `http.HandlerFunc`
* 与原生 `http.Handler` 和 `http.HandlerFunc` 的组合兼容

## 需求

Go 1.2 或以上。

## 安装

Run the following command to install the package:<br>
运行以下指令安装此包：

```
go get github.com/go-ozzo/ozzo-routing
```

## 准备开始

创建一个 `server.go` 文件，包含以下内容：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"github.com/go-ozzo/ozzo-routing"
)

func main() {
	r := routing.NewRouter()

	// install commonly used middlewares
	// 安装常用中间件
	r.Use(
		routing.AccessLogger(log.Printf),
		routing.TrailingSlashRemover(http.StatusMovedPermanently),
	)

	// set up routes and handlers
	// 配置路由与处理器
	r.Get("", func(c *routing.Context) {
		fmt.Fprint(c.Response, "Go ozzo!")
	})
	r.Get("/users", func(c *routing.Context) {
		fmt.Fprint(c.Response, "getting users")
	})
	r.Group("/admin", func(gr *routing.Router) {
		gr.Post("/users", func(c *routing.Context) {
			fmt.Fprint(c.Response, "creating users")
		})
		gr.Delete("/users", func(c *routing.Context) {
			fmt.Fprint(c.Response, "deleting users")
		})
	})

	// handle requests that don't match any route
	// 处理没有任何匹配的请求
	r.Use(routing.NotFoundHandler())

	// handle errors triggered by handlers
	// 处理由处理句柄发出的错误
	r.Error(routing.ErrorHandler(nil))

	// hook up the router and start up a Go Web server
	// 挂载路由处理器，并启动一个 Go 的 Web 服务器。
	http.Handle("/", r)
	http.ListenAndServe(":8080", nil)
}
```

Now run the following command to start the Web server:<br>
现在运行以下指令来启动该 Web 服务器

```
go run server.go
```

You should be able to access URLs such as `http://localhost:8080`, `http://localhost:8080/users`.<br>
此时你应该已经可以访问如右所示的 URL 了： `http://localhost:8080`, `http://localhost:8080/users`。


## 路由树

ozzo-routing works by building a *routing tree* and dispatching HTTP requests to the handlers on this tree.<br>
ozzo-routing 的运作机理是先在内部构建一个*路由树(routing tree)*，然后会把 HTTP 请求分发给这个树上的各个处理句柄。

A leaf node on the routing tree is called a *route*, while a non-leaf node is a *router*. On each node
(either a leaf or a non-leaf node), there is a list of *handlers* (aka middlewares) which contain the custom logic
for handling HTTP requests.<br>
路由树上的叶节点被称作*路由*，而非叶节点被称为*路由器*。在每一个节点（无论是叶节点还是非叶节点）上，都有一个*处理句柄*（即中间件）的列表，中间件里面则包含有可用于处理这些 HTTP 请求的自定义逻辑。

Dispatching an incoming HTTP request starts from the root of the routing tree in a depth-first traversal.<br>
分发传入的 HTTP 请求时，会从路由树的根节点开始，进行一个深度优先搜索。
The HTTP method and the URL path are used to match against the encountering nodes. The handlers on the matching
nodes will be invoked according to the node order and handler order. A handler should call either
`Context.Next()` or `Context.NextRoute()` to pass the control to the next eligible handler. <br>
途经的节点会尝试匹配传入请求的 HTTP 方法和 URL 路径。每个配对成功的节点会依次调用当前节点上的各个处理句柄，因此，整个路由处理过程会依据节点的顺序和处理句柄的顺序依次进行。每个处理器应该调用 `Context.Next()` 或 `Context.NextRoute()` 中的一个来将控制权传递给下一个符合要求的处理节点。
<br>Otherwise, the request handling is considered complete and no further handlers will be invoked.<br>
否则，此次请求的处理过程会被视作已完结，任何后续的处理句柄都不会继续被调用。

To build a routing tree, first call `routing.NewRouter()` to create the root node. Then call `Router.To()`, `Router.Get()`,
`Router.Post()`, etc., to create leaf nodes, or call `Router.Group()` to create a non-leaf node. For example,<br>
啊哟构建一个路由树，首先应该调用 `routing.NewRouter()` 方法新建一个根节点。之后可以调用 `Router.To()`、`Router.Get()`、`Router.Post()` 等等来创建叶节点。也可以调用 `Router.Group()` 创建一个非叶节点。举栗：

```go
// 根
r := routing.NewRouter()

// 叶 (路由)
r.Get("", handler1, handler2, ...)
r.Get("/users", handler1, handler2, ...)

// 一个内部节点（非叶节点） (子路由器)
r.Group("/admin", func(r *routing.Router) {
	// 内部节点之下的叶节点
	r.Post("/users", handler1, handler2, ...)
	r.Delete("/users", handler1, handler2, ...)
})
```

Because `Router` implements `http.Handler`, it can be readily used to serve subtrees on existing Go servers.
For example,<br>
因为 `Router` 实现了 `http.Handler`，他可以很轻易地用于提供其他已有 Go 服务器上的子树。

```go
http.Handle("/", r)
http.ListenAndServe(":8080", nil)
```


## 路由

A route has a path pattern that is used to match the URL path of incoming requests. Only requests matching the pattern
may be dispatched by the route. For example, a pattern `/users` matches any request whose URL path is `/users`.<br>
路由会包含一个可以用于匹配传入请求的 URL 路径的路径表达式。只有成功匹配该表达式的传入请求会被分发到该路由上。举例而言，`/users` 匹配所有 URL 路径为 `/users` 的传入请求。
Regular expression can be used in the pattern. For example, a pattern `/users/\\d+` matches URL path `/users/123`,
but not `/users` or `/users/abc`.<br>
表达式可以使用正则表达式。比如，一个 `/users/\\d+` 的表达式可以匹配 URL 路径 `/users/123`，但是不包含 `/users` 或 `/users/abc`。

Optionally, a route may have one or multiple HTTP methods (e.g. `GET`, `POST`) so that only requests using one of
those HTTP methods may be dispatched by the route.<br>
另外，一个路由可以包含一个或多个 HTTP 方法（如：`GET`, `POST`），这样只有使用列出的 HTTP 方法的传入请求可以被分发到该路由。
A route is usually associated with one or multiple handlers. When a route matches and dispatches a request, its handlers
will be called.<br>
一个路由通常会关联有一个或多个的处理句柄。当路由匹配并被分发某个请求时，他的处理句柄就会被调用。

You can create and add a route to a routing tree by calling `Router.To()` or one of its shortcut methods, such as `Router.Use()`,
`Router.Get()`. For example,<br>
你可以调用 `Router.To()` 方法创建并添加某个路由到已有的路由树。它也有一些简便写法，比如 `Router.Use()`、`Router.Get()`等，栗子来了：

```go
r := routing.New()

r.To("GET /users", func(*routing.Context) { })

// or equivalently using the Get() shortcut
// 或者使用等效的 Get() 简写方法
r.Get("/users", func(*routing.Context) { })
```

The above code adds a route that matches URL path `/users` and only applies to the GET HTTP method. You may
also call `Post()`, `Put()`, `Patch()`, `Head()`, or `Options()` to deal with other common HTTP methods.<br>
上述代码添加了一个匹配 URL 路径 `/users` 的路由，且该路由仅适用于 HTTP GET 方法。如你所想，你也可以调用 `Post()`、`Put()`、`Patch()`、`Head()`，或 `Options()` 方法来适配其它的常用 HTTP 方法。

If a route should match multiple HTTP methods, you can use the syntax like shown below:
若某路由可以对应多个 HTTP 方法，你可以使用如下所示的语法：

```go
// only match GET or POST
// 只匹配 GET 或 POST
r.To("GET,POST /users", func(*routing.Context) { })

// match any HTTP method
// 匹配所有 HTTP 方法
r.To("/users", func(*routing.Context) { })
```

If a route should match *any request*, call `Router.Use()` like the following:
如果某路由需要匹配**所有请求**，请想这样调用 `Router.Use()` 方法：

```go
r.Use(func(*routing.Context) { })
```


### URL 参数

The path pattern specified for a route can be used to capture URL parameters by embedding tokens in the format
of `<name:pattern>`, wherestands for the parameter name, and `pattern` is a regular expression which
the parameter value should match. You can omit the `pattern` part, which means the parameter should match a non-empty
string without any slash character.<br>
为路由所指定的路径表达式，也可用于捕获 URL 参数。只需在其中嵌入 `<name:pattern>` 格式的标记，即可创建一个名为 `name` 的参数，而 `pattern` 代表用于匹配此参数的正则表达式。当然，如果没有特殊需求，也可以省略 `pattern` 的部分，此时代表捕获斜杠内的任意非空字符串（字符串本身不能包含 `/`）。

When a route matches a URL path, the matching parameters on the URL path will be made available through
`Context.Params`. For example,
当一个路由匹配某 URL 路径，其 URL 中与路由相匹配的参数会被筛选出来，之后可以通过 `Context.Params` 进行访问，如：

```go
r := routing.NewRouter()

r.To("GET /cities/<name>", func (c *routing.Context) {
	fmt.Fprintf(c.Response, "Name: %v", c.Params["name"])
})

r.To("GET /users/<id:\\d+>", func (c *routing.Context) {
	fmt.Fprintf(c.Response, "ID: %v", c.Params["id"])
})
```


## 处理句柄(Handlers)

Handlers are functions associated with routers or routes. A handler is called when a request is dispatched
to the route or router that the handler is associated with.<br>
处理句柄（有时也被称作处理器或处理函数）是一种与路由器或路由相绑定的函数。当一个请求被分发给相应的路由或路由器时，与其绑定的句柄就会被调用。

Within a handler, you can call `Context.Next()` to pass the control to the next available
handler on the same route or the first handler on the next matching route.
You may also call `Context.NextRoute()` to invoke the first handler of the next matching route.<br>
在一个处理函数内部，你可以调用 `Context.Next()` 方法将控制权传递至同路由的下一个可用处理句柄，或下一个匹配路由的第一个处理句柄。你也可以用 `Context.NextRoute()` 把控制权强制传递给下一个路由的第一个句柄。

Usually, handlers serving as filters should call `Context.Next()` so that the next handlers
can get a chance to further process a request. Handlers that are controller actions often do not
call `Context.Next()` because they are the last step of request processing.
`Context.NextRoute()` is often called by handlers to determine if the current route/router
should be used to dispatch the request.<br>
通常，扮演过滤器（中间步骤）一类角色的句柄，都应该调用 `Context.Next()`，这样之后的处理句柄才能得到进一步处理传入请求的机会。而扮演控制器动作（Controller Action）的句柄则不需要调用 `Context.Next()` 因为他们通常是请求处理的最后一个步骤。一般来说，只有需要判断当前路由是否需要被执行的处理句柄才会时常调用 `Context.NextRoute()` 方法。（译者注：比如如果用户来自江浙沪，则跳过录入邮费的过程_(:з」∠)_）

比如：

```go
r := routing.NewRouter()
r.Get("/users", func(c *routing.Context) {
	fmt.Fprintln(c.Response, "/users1 start")
	c.Next()
	fmt.Fprintln(c.Response, "/users1 end")
}, func(c *routing.Context) {
	fmt.Fprintln(c.Response, "/users2 start")
	c.Next()
	fmt.Fprintln(c.Response, "/users2 end")
})

r.Get("/users", func(c *routing.Context) {
	fmt.Fprintln(c.Response, "/users3")
})

r.Get("/users", func(c *routing.Context) {
	fmt.Fprintln(c.Response, "/users4")
})
```

When dispatching the URL path `/users` with the above routing tree, it will output the following text:<br>
当使用以上路由树分发 `/users` 路径的请求时，它会输出以下文字：

```
/users1 start
/users2 start
/users3
/users2 end
/users1 end
```

Note that `/user4` is not displayed because the request dispatching ends after displaying `/user3`.
Also note that the handler outputs are properly nested.<br>
请注意，`/user4` 并没有显示，这是因为请求的分发在显示 `/user3` 的时候就已经停止了。同时，请注意，处理器的输出是合理地嵌套起来的。


### 上下文(Context)

For each incoming request, a new `routing.Context` object is created which includes contextual
information for handling the request, such as the current request, response, etc. The context object
is passed as the parameter to the handlers that are processing the request.<br>
对于每个传入请求，都创建了一个新的 `routing.Context` 对象，其中包括用于处理此请求的上下文信息，比如当前 request 和 response，等等。此上下文对象会被作为参数传入每一个用于处理此请求的句柄。

Using `Context`, handlers can share data between each other. A simple way is to exploit the `Context.Data` field.
For example, one handler stores `Context.Data["user"]` which can be accessed by another handler. For example,<br>
使用 `Context`，处理句柄可以在彼此之间分享数据。最简单的做法，当然是利用其 `Context.Data` 字段。比如，某句柄先存储数据到  `Context.Data["user"]`，就可以被其他句柄访问了。

```go
r := routing.NewRouter()
r.Use(func (c *routing.Context) {
	// 使用 Context.Data 分享数据
	c.Data["cache"] = &Cache{}
})
r.Use(func (c *routing.Context, cache *Cache) {
	// 访问 c.Data["cache"]
})
```

### 响应与返回值

Many handlers need to send output in response. This can be done using the following code:<br>
许多句柄需要在响应中发送输出值。这可以用以下代码实现：

```go
func (c *routing.Context) {
	fmt.Fprint(c.Response, "Hello world")
}
```

An alternative way is to call the `Context.Write()` method. This method differs from the previous one in that
it will call `Response.WriteData()` to write the data if `Context.Response` implements the `routing.DataWriter` interface.
This allows the response object to do some preprocessing (e.g. serializing data in JSON or XML format) before sending
it to output.<br>
还有另外一个方法是调用 `Context.Write()` 方法。此方法与前者的不同之处在于，若 `Context.Response` 实现了 `routing.DataWriter` 接口，则它会调用 `Response.WriteData()` 方法。这样就可以让响应对象在发送输出信息之前先做一些处理。（比如，把数据序列化为 JSON 或 XML 格式）


### 内建的处理句柄

ozzo-routing 包含少量自带的常用处理句柄：

* `routing.ErrorHandler`：错误处理器
* `routing.NotFoundHandler`：可以触发 404 错误的处理器
* `routing.TrailingSlashRemover`：从请求 URL 中移除尾部斜杠的处理器
* `routing.AccessLogger`：记录所有的传入请求的处理器
* `routing.Static`：把指定文件夹内的文件作为响应内容返回的处理器
* `routing.StaticFile`：提供指定文件的内容作为返回响应的处理器

These handlers may be used like the following:<br>
这些处理器的调用方式如下所示：

```go
r := routing.NewRouter()

r.Use(
	routing.AccessLogger(log.Printf),
	routing.TrailingSlashRemover(http.StatusMovedPermanently),
)

// ... 注册路由与句柄

r.Use(routing.NotFoundHandler())

r.Error(routing.ErrorHandler(nil))
```

Additional handlers related with RESTful API services may be found in the
[ozzo-rest Go package](https://github.com/go-ozzo/ozzo-rest).<br>
要找寻更多与 RESTful API 服务相关的处理句柄，请参阅 [ozzo-rest](https://github.com/go-ozzo/ozzo-rest) 包。

### 第三方的句柄

ozzo-routing supports third-party `http.HandlerFunc` and `http.Handler` handlers. Adapters are provided
to make using third-party handlers an easy task. For example,<br>
ozzo-routing 支持第三方的 `http.HandlerFunc` 和 `http.Handler` 处理句柄。同时，ozzo 还提供了适配器(Adapter)，使得调用其他第三方的处理句柄变得小菜一碟。举个栗子：

```go
r := routing.NewRouter()

// 使用 http.HandlerFunc
r.Use(routing.HTTPHandlerFunc(http.NotFound))

// 使用 http.Handler
r.Use(routing.HTTPHandler(http.NotFoundHandler))
```

## 路由组(Route Groups)

Routes matching the same URL path prefix can be grouped together by calling `Router.Group()`. The support for route
groups enables modular architecture of your application. For example, you can have an `admin` module which uses
the group of the routes having `/admin` as their common URL path prefix. The corresponding routing can be set up
like the following:<br>
匹配相同路由前缀的路由可以用 `Router.Group()` 方法分为一组。对于路由组的支持让模块化的组织应用构架成为可能。举栗而言，你可以创建一个 `admin` 模块，此模块便可使用 `/admin` 作为其路由组下所有路由的通用 URL 路径前缀，岂不美哉。配置方法如下：

```go
r := routing.NewRouter()

// ...其他路由...

// the /admin route group
r.Group("/admin", function(gr *routing.Router) {
	gr.Post("/users", func(*routing.Context) { })
	gr.Delete("/users", func(*routing.Context) { })
	// ...
})
```

Note that when you are creating a route within a route group, the common URL path prefix should be removed
from the path pattern, like shown in the above example.<br>
需要注意的是，在路由组内创建路由的时候，表达式里不要包括路径前缀，如上所示。

You can create multiple levels of route groups. In fact, as we have explained earlier, the whole routing system
is a tree structure, which allows you to organize your code in a multilevel modular fashion.<br>
你可以用路由组创建多个层级。事实上，如我们之前所解释的，整个路由系统是一个树状结构，这可以让你以一种多层级模块化的形式组织你的代码，屌不屌？！

## 提供静态文件

Static files can be served through the `routing.Static` or `routing.StaticFile` handler. The former serves files
under the specified directory according to the current request, while the latter serves a single file. For example,<br>
静态文件可以用 `routing.Static` 或 `routing.StaticFile` 句柄进行提供。前者根据当前的请求提供在特定目录下的文件访问，而后者只提供单个文件的访问。举例：

```go
r := routing.NewRouter()
// 提供 working-dir/web/assets 下的文件。
r.To("/assets(/.*)?", routing.Static("web"))
```


## 错误处理

ozzo-routing supports error handling via error handlers. An error handler is a handler registered
by calling the `Router.Error()` method. When a panic happens in a handler, the router will recover
from it and call the error handlers registered after the current route. Any normal handlers in between
will be skipped.<br>
ozzo-routing 支持利用错误处理器的错误处理。错误处理器是一种通过 `Router.Error()` 方法进行注册的特殊处理句柄。当在某个处理句柄中出现 `paneic` 时，路由器会进行 recover 操作，并调用注册给当前路由的错误处理器。所有二者之间的正常句柄都会被跳过。

Error handlers can obtain the error information from `Context.Error`. For example,<br>
错误处理器可以从 `Context.Error` 中获取错误信息。比如：

```go
r := routing.NewRouter()

// ... 注册路由与处理句柄

r.Error(func(c *routing.Context) {
	fmt.Println(c.Error)
})
```

When there are multiple error handlers, `Context.Next()` may be called in one error handler to
pass the control to the next error handler.<br>
当存在不止一个错误处理器时，可以用 `Context.Next()` 方法从一个跳转到另一个。

For convenience, `Context` provides a method named `Panic()` to simplify the way of triggering an HTTP error.
For example,
为了方便，`Context` 提供一个名叫 `Panic()` 的方法以简化发送 HTTP 错误的过程。

```go
func (c *routing.Context) {
	c.Panic(http.StatusNotFound)
	// 等效于以下代码：
	// panic(routing.NewHTTPError(http.StatusNotFound))
}
```


## 鸣谢

ozzo-routing 参考了 [Express](http://expressjs.com/)、[Martini](https://github.com/go-martini/martini) 和一些其他类似的项目。
