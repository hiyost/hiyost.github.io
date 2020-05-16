---
title: k8s中的go-restful库
date: 2019-06-22 16:29:18
tags: code, package, rest 
---





[go-restful](<https://github.com/emicklei/go-restful>)库是一个用go语言实现的REST风格的Web服务框架，



## 1 关键数据结构

对于任何代码，数据结构是基础，搞清楚了数据结构才能搞清楚代码运行的逻辑。借用钟大师的一段话：

> 数据结构是什么？数据结构就是舞台上的角色，而函数方法就是这些角色之间演出的一幕幕戏。对象是有生命的，从创建到数据流转，从产生到消亡。而作为开发者来说，首先是搞懂这些人物设定，是关公还是秦琼，是红脸还是黑脸？看懂了人，就看懂了戏。   ——摘自钟大师的[博客](http://www.dockone.io/article/895)

因此在分析go-restful这个库的功能和使用方法之前，我们先来了解一下这个库的几个关键数据结构。

<!-- more -->

### 1.1 `Container`



`Container`对应的是一个server端中所有API的集合，一般一个server中包含一个`Container`即可。



```go
// Container holds a collection of WebServices and a http.ServeMux to dispatch http requests.
// The requests are further dispatched to routes of WebServices using a RouteSelector
type Container struct {
	webServicesLock        sync.RWMutex
	webServices            []*WebService
	ServeMux               *http.ServeMux
	isRegisteredOnRoot     bool
	containerFilters       []FilterFunction
	doNotRecover           bool // default is true
	recoverHandleFunc      RecoverHandleFunction
	serviceErrorHandleFunc ServiceErrorHandleFunction
	router                 RouteSelector // default is a CurlyRouter (RouterJSR311 is a slower alternative)
	contentEncodingEnabled bool          // default is false
}
```



这其中有2个元素特别关键：

- `webServices`，后文会继续说明
- `ServeMux`，这个其实就是go的`http`库中的`ServeMux`，也就是路由规则器，这个是go中起`server`的关键数据结构，其中包含了路由与对应处理函数的map，这个是注册API的关键

```go
// 源码位于go的原生库中：src/net/http/server.go
// ServeMux is an HTTP request multiplexer.
// It matches the URL of each incoming request against a list of registered
// patterns and calls the handler for the pattern that
// most closely matches the URL.
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}

// A Handler responds to an HTTP request.
//
// ServeHTTP should write reply headers and data to the ResponseWriter
// and then return. Returning signals that the request is finished; it
// is not valid to use the ResponseWriter or read from the
// Request.Body after or concurrently with the completion of the
// ServeHTTP call.
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

通过`ServeMux`数据结构可以很清楚地推断出server端再收到请求之后做的事情：先根据请求的URL再map中找到对应注册的`Handler`，再调用`Handler`实现的`ServeHTTP(ResponseWriter, *Request)`函数来处理请求。



### 1.2 `WebService`



`WebService`是

```go
// WebService holds a collection of Route values that bind a Http Method + URL Path to a function.
type WebService struct {
	rootPath       string
	pathExpr       *pathExpression // cached compilation of rootPath as RegExp
	routes         []Route
	produces       []string
	consumes       []string
	pathParameters []*Parameter
	filters        []FilterFunction
	documentation  string
	apiVersion     string

	typeNameHandleFunc TypeNameHandleFunction

	dynamicRoutes bool

	// protects 'routes' if dynamic routes are enabled
	routesLock sync.RWMutex
}
```



### 1.3 `Route`



`Route`中绑定了某一个资源的方法和路径所对应的handler函数，是最小的API操作单位。

```go
// Route binds a HTTP Method,Path,Consumes combination to a RouteFunction.
type Route struct {
	Method   string
	Produces []string
	Consumes []string
	Path     string // webservice root path + described path
	Function RouteFunction
	Filters  []FilterFunction

	// cached values for dispatching
	relativePath string
	pathParts    []string
	pathExpr     *pathExpression // cached compilation of relativePath as RegExp

	// documentation
	Doc                     string
	Notes                   string
	Operation               string
	ParameterDocs           []*Parameter
	ResponseErrors          map[int]ResponseError
	ReadSample, WriteSample interface{} // structs that model an example request or response payload

	// Extra information used to store custom information about the route.
	Metadata map[string]interface{}
}
```





## 2 使用方法



