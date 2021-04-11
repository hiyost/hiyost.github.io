---
title: beego代码分析
date: 2021-01-08 10:23:22
tags: beego
---

最近在给BCE的前端做概览接口，发现之前设计的excution之类的API比较难做这个概览，在走读batch-apiserver的代码时发现之前了解的beego相关的知识已经不足以支撑自己做一些重构或者特性了，主要是面对多个controller的时候没有找到一个比较好的方法去设计接口来聚合，因此趁此机会好好走读一下beego的代码，当前业务中使用的都是beego 1.12.2的代码，因此本文也以这个版本作为基线（虽然beego 2.0已经发布有一段时日了）。



### 1、 前言

在分析beego的代码之前，我们需要先了解beego到底做了什么，根据beego的[官方介绍](https://beego.me/docs/intro/)：

> beego 是一个快速开发 Go 应用的 HTTP 框架，他可以用来快速开发 API、Web 及后端服务等各种应用，是一个 RESTful 的框架，主要设计灵感来源于 tornado、sinatra 和 flask 这三个框架，但是结合了 Go 本身的一些特性（interface、struct 嵌入等）而设计的一个框架。

从中不难看出，beego的本质还是一个http框架，用来搭建一个http server。要知道，Go语言的http库已经非常强大了，启动一个简单的server代码在10行左右就能搞定，既然如此，我们来对比一下使用Go官方的http库和使用beego分别启动，看看两者的差别。我们的要求如下：

- 在本地的8080端口启动一个server，对于"/hello"这个路由的GET请求返回"Hello world!"

#### 1.1、使用http库实现

以下是直接使用http库来实现的server，可以看到，寥寥几句确实简洁无比。

```go
package main

import (
	"fmt"
	"net/http"
)
// 自定义一个Handler
func indexHandler(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		fmt.Fprintf(w, "Hello world!")
	}
}

func main() {
    // 1、注册Handler
	http.HandleFunc("/hello", indexHandler)
    // 2、启动server
	http.ListenAndServe(":8080", nil)
}
```

Go的http库是一个十分值得仔细研究的库，可以参考[官方文档]()，关于http server的部分也可以参考[深入理解Golang之http server](https://juejin.cn/post/6844903998869209095)。

#### 1.2、使用beego来实现

使用beego来注册路由时可以控制的更加精细，譬如可以在注册时针对不同的请求动作（GET/POST/DELETE/...）来注册不同的方法。而http库就需要在Handler中通过请求的动作来分别处理。

```go
package main

import (
	"github.com/astaxie/beego"
)

// 自定义一个controller类型
type SampleController struct {
	beego.Controller
}
// 实现一个方法作为处理的Handler
func (this *SampleController) Hello() {
	this.Ctx.WriteString("Hello World!")
}
// 1、初始化（注册）Handler，将SampleController这个类型的Hello方法作为/hello这个路由GET请求的处理Handler
func init() {
	beego.Router("/hello", &SampleController{}, "get:Hello")
}
// 2、启动server
func main() {
	beego.Run()
}
```

不过，本质上beego还是http库的一个封装，最终还是由http库中的各个类型和接口来实现的。

### 2、代码剖析

#### 2.1、注册路由

##### 2.1.1、`Router`函数

从前面样例中可以看到，在注册一个路由之前，需要先定义好这个路由对应的类型`SampleController`以及其使用的方法`Hello()`。这个类型内嵌了`beego.Controller`，其结构体定义如下：

```go
// vendor/github.com/astaxie/beego/controller.go
// Controller defines some basic http request handler operations, such as
// http context, template and view, session and xsrf.
type Controller struct {
	// http ctx数据，beego的Context基于http.Context做了一层封装
	Ctx  *context.Context
	Data map[interface{}]interface{}

	// route controller info
	controllerName string   // 实例化的controller名字
	actionName     string   // 需要执行的Controller的方法名
	methodMapping  map[string]func() //method:routertree
	AppController  interface{}

	// template data  // view相关的信息
	TplName        string
	ViewPath       string
	Layout         string
	LayoutSections map[string]string // the key is the section name and the value is the template name
	TplPrefix      string
	TplExt         string
	EnableRender   bool

	// xsrf data
	_xsrfToken string
	XSRFExpire int
	EnableXSRF bool

	// session
	CruSession session.Store
}
```

需要注意的是，`beego.Controller`是一个过程中数据，每接收到一次请求，就会创建一个`beego.Controller`实例，生命周期与`http.Context`类似（其本质也是在`http.Context`的基础上做了进一步封装）。



接下来我们看注册的部分，`beego`中注册一个Handler时使用到的是`Router`函数，我们进入这个函数一探究竟。

```go
// vendor/github.com/astaxie/beego/app.go

// Router adds a patterned controller handler to BeeApp.
// it's an alias method of App.Router.
// Router支持以多种方式注册路由
// usage:
//  simple router
//  beego.Router("/admin", &admin.UserController{})
//  beego.Router("/admin/index", &admin.ArticleController{})
//
//  regex router  // 正则路由
//
//  beego.Router("/api/:id([0-9]+)", &controllers.RController{})
//
//  custom rules
//  beego.Router("/api/list",&RestController{},"*:ListFood")
//  beego.Router("/api/create",&RestController{},"post:CreateFood")
//  beego.Router("/api/update",&RestController{},"put:UpdateFood")
//  beego.Router("/api/delete",&RestController{},"delete:DeleteFood")
func Router(rootpath string, c ControllerInterface, mappingMethods ...string) *App {
	BeeApp.Handlers.Add(rootpath, c, mappingMethods...)
	return BeeApp
}
```

在`Router`函数中，入参有3个：

- `rootpath string`：路由的路径，可以是精准匹配，也可以是正则匹配，正则匹配时支持获取匹配的字符串内容，譬如正则路径是`/api/:id`，那么可以在对应的`Controller`中通过`c.Ctx.Input.Param(":id")`来获取；

- `c ControllerInterface`：对应于路由的处理方法，一般都是在`beego.Controller`这个类型的基础上内嵌实现，不过入参使用的是`beego.Controller`实现了的`ControllerInterface`这个接口

  - `ControllerInterface`接口定义如下：

    ```go
    // vendor/github.com/astaxie/beego/controller.go
    // ControllerInterface is an interface to uniform all controller handler.
    type ControllerInterface interface {
    	Init(ct *context.Context, controllerName, actionName string, app interface{})
    	Prepare()
    	Get()
    	Post()
    	Delete()
    	Put()
    	Head()
    	Patch()
    	Options()
    	Trace()
    	Finish()
    	Render() error
    	XSRFToken() string
    	CheckXSRFCookie() bool
    	HandlerFunc(fn string) bool
    	URLMapping()
    }
    ```
    
  - 每一个通过`Router`函数注册的路由的`Controller`都需要实现`ControllerInterface`接口，一般开发人员自己写`Controller`的时候都是将`beego.Controller`（已经统一实现了所有的`ControllerInterface`接口，不过默认返回405/StatusMethodNotAllowed）内嵌进来，然后再自己实现自己的特有方法，例如我们1.2章节中的实现就是：

    ```go
    type SampleController struct {
    	beego.Controller
    }
    ```

    

- `mappingMethods ...string`：变长入参，用来映射请求的动作（GET/POST/DELETE/...）和对应的处理方法函数，可以支持通过`;`隔开的方式映射不同的对应，如`"get:Hello;list:Hello"`，同时请求的动作可以通过`*`来匹配所有动作，如`"*:Hello"`。

我们回到`Router`函数，这个函数中最终其实调用的是`BeeApp.Handlers.Add`，其中`BeeApp`是beego这个库中十分关键的一个全局变量，其中一个成员`Server`就是Go http库中的`http.Server`，另一个成员`Handlers`则实现了`http.Handler`接口，这也是`BeeApp`中最重要的一个成员。

```go
// vendor/github.com/astaxie/beego/app.go
package beego

var (
	// BeeApp is an application instance
	BeeApp *App
)

func init() {
	// create beego application
	BeeApp = NewApp()
}

// App defines beego application with a new PatternServeMux.
type App struct {
	Handlers *ControllerRegister //自定义Handler，实现了http.Handler接口，通过反射来实现不同路由调用对应的处理函数
	Server   *http.Server   // 实际的http server
}

// NewApp returns a new beego application.
func NewApp() *App {
	cr := NewControllerRegister()
	app := &App{Handlers: cr, Server: &http.Server{}}
	return app
}
```

通过`App`的结构体中的成员可以料想到，注册完成后启动http server时，也一定是通过`BeeApp.Server`来启动server，这一点我们在下文中论证。



##### 2.1.2、`ControllerRegister`

`BeeApp.Handlers`其实是一个`ControllerRegister`，这是`beego`自己维护的一套路由映射规则，当程序启动时，通过`init()`函数注册的路由信息会全部保存到这里面，程序启动之后，当收到http请求时，处理函数会从中查询请求的URL对应的处理函数，然后调用这个函数来响应请求，这是`beego`的处理中枢。

```go
// vendor/github.com/astaxie/beego/router.go
// ControllerRegister containers registered router rules, controller handlers and filters.
type ControllerRegister struct {
	routers      map[string]*Tree  // 路由表，key是http.method，value是实际的路由和函数组成的树结构
	enablePolicy bool    // 是否开启policy
	policies     map[string]*Tree
	enableFilter bool    // 是否开启filter
	filters      [FinishRouter + 1][]*FilterRouter// 开启的filter表，主要包括BeforeStatic/BeforeRouter/BeforeExec/AfterExec/FinishRouter
	pool         sync.Pool // 
}

// NewControllerRegister returns a new ControllerRegister.
func NewControllerRegister() *ControllerRegister {
	return &ControllerRegister{
		routers:  make(map[string]*Tree),
		policies: make(map[string]*Tree),
		pool: sync.Pool{
			New: func() interface{} {
				return beecontext.NewContext()
			},
		},
	}
}

// Add controller handler and pattern rules to ControllerRegister.
// usage:
//	default methods is the same name as method
//	Add("/user",&UserController{})
//	Add("/api/list",&RestController{},"*:ListFood")
//	Add("/api/create",&RestController{},"post:CreateFood")
//	Add("/api/update",&RestController{},"put:UpdateFood")
//	Add("/api/delete",&RestController{},"delete:DeleteFood")
//	Add("/api",&RestController{},"get,post:ApiFunc"
//	Add("/simple",&SimpleController{},"get:GetFunc;post:PostFunc")
func (p *ControllerRegister) Add(pattern string, c ControllerInterface, mappingMethods ...string) {
	p.addWithMethodParams(pattern, c, nil, mappingMethods...)
}
```

`Router`函数调用的就是这里的`Add`，通过其注释可以看到支持的多种格式。



##### 2.1.3、`addWithMethodParams`

`Router`函数最终调用的就是`ControllerRegister`的`addWithMethodParams`，这个函数对于理解`beego`的实现十分关键，在`Router`函数注册路由的时候，之所以只需要传入`Controller`的空实例，是因为在`addWithMethodParams`中通过反射已经获取了这个`Controller`的`Type`和`Value`，而`beego`中正是通过反射来真正实现路由的映射管理的。

```go
// vendor/github.com/astaxie/beego/router.go
func (p *ControllerRegister) addWithMethodParams(pattern string, c ControllerInterface, methodParams []*param.MethodParam, mappingMethods ...string) {
	reflectVal := reflect.ValueOf(c)
	t := reflect.Indirect(reflectVal).Type() // 通过反射获取Controller的类型，后面接收请求时会 使用这个类型创建一个实例，并调用这个实例对应的方法
	methods := make(map[string]string)  // 请求类型和处理方法的映射，key是请求类型，如get，value是方法，如GetFunc
	if len(mappingMethods) > 0 {
		semi := strings.Split(mappingMethods[0], ";") // 分割映射对"get:GetFunc;post:PostFunc"
		for _, v := range semi {
			colon := strings.Split(v, ":")  // 分割映射"get:GetFunc"
			if len(colon) != 2 {
				panic("method mapping format is invalid")
			}
			comma := strings.Split(colon[0], ",")  // 处理多对一的场景 "get,post:ApiFunc"
			for _, m := range comma {
				if m == "*" || HTTPMETHOD[strings.ToUpper(m)] { // 处理全匹配的场景"*:ListFood"
                    // 通过反射查看当前这个Controller是否实现了对应的方法，如ListFood
					if val := reflectVal.MethodByName(colon[1]); val.IsValid() {
						methods[strings.ToUpper(m)] = colon[1]
					} else {
						panic("'" + colon[1] + "' method doesn't exist in the controller " + t.Name())
					}
				} else {
					panic(v + " is an invalid method mapping. Method doesn't exist " + m)
				}
			}
		}
	}

	route := &ControllerInfo{}  // 用来保存某一个路由对应的所有信息，这个类型在2.1.3.1中会介绍
	route.pattern = pattern
	route.methods = methods  // 支持的所有请求类型和对应的处理方法
	route.routerType = routerTypeBeego
	route.controllerType = t  // 处理请求的Controller类型反射
	route.initialize = func() ControllerInterface { // 初始化函数
		vc := reflect.New(route.controllerType) // 通过反射创建一个Controller类型实例
		execController, ok := vc.Interface().(ControllerInterface) // 强制转换成接口
		if !ok {
			panic("controller is not ControllerInterface")
		}

        // 后面这一段主要是将实例类型的方法赋予给execController
		elemVal := reflect.ValueOf(c).Elem()
		elemType := reflect.TypeOf(c).Elem()
		execElem := reflect.ValueOf(execController).Elem()

		numOfFields := elemVal.NumField()
		for i := 0; i < numOfFields; i++ {
			fieldType := elemType.Field(i)
			elemField := execElem.FieldByName(fieldType.Name)
			if elemField.CanSet() {
				fieldVal := elemVal.Field(i)
				elemField.Set(fieldVal)
			}
		}

		return execController
	}

	route.methodParams = methodParams
	if len(methods) == 0 {  // 对应于("/user",&UserController{})的场景，methods没有指定
		for m := range HTTPMETHOD {
			p.addToRouter(m, pattern, route)
		}
	} else {  // 对应于指定了method的场景
		for k := range methods {
			if k == "*" {
				for m := range HTTPMETHOD {
					p.addToRouter(m, pattern, route)
				}
			} else {
				p.addToRouter(k, pattern, route)
			}
		}
	}
}
```

最终将路由和对应的处理方法进行映射使用的是`addToRouter`函数，其中主要是调用了`beego`中最关键的`Tree`来维护请求的URL与对应处理函数之间的关系，这个在2.1.3.2中会关键讲述。

```go
// vendor/github.com/astaxie/beego/router.go
func (p *ControllerRegister) addToRouter(method, pattern string, r *ControllerInfo) {
	if !BConfig.RouterCaseSensitive {
		pattern = strings.ToLower(pattern)
	}
    // 注意method是GET/PUT/PATCH/DELETE等http方法，pattern是请求的URL
	if t, ok := p.routers[method]; ok {
		t.AddRouter(pattern, r)
	} else {
		t := NewTree()
		t.AddRouter(pattern, r)
		p.routers[method] = t
	}
}
```

这里最终执行到了`t.AddRouter(pattern, r)`，这个在2.1.3.2节中继续往下探讨。



###### 2.1.3.1、`ControllerInfo`

先来看一下`ControllerInfo`，在前面的代码中我们看到，注册路由时，将路由以及对应处理函数的Controller的反射类型都保存到了这里，这个类型中的所有成员如下：

```go
// ControllerInfo holds information about the controller.
type ControllerInfo struct {
	pattern        string   // URL
	controllerType reflect.Type  // 注册这个URL对应的Controller的反射类型
	methods        map[string]string // 注册这个URL时的http方法与处理函数的映射，如"get:GetFunc;post:PostFunc"
	handler        http.Handler  // routerTypeHandler这种类型的路由对应的http.Handler，本质就是1.1中的例子
	runFunction    FilterFunc // 通过GET/PUT等（不是通过Router方式）注册路由时对应的过滤函数
	routerType     int  // 注册的路由类型，包含routerTypeBeego/routerTypeRESTFul/routerTypeHandler
	initialize     func() ControllerInterface // 处理请求之前的初始化函数，主要对controllerType进行实例化
	methodParams   []*param.MethodParam // 通过Auto方式注册路由时用来保存所有http请求参数的成员
}
```

需要注意的是，`Tree`中保存的主要也是这个`ControllerInfo`。

###### 2.1.3.2、`Tree`

> 注意：在第一次读到Tree这里时，可以先跳过这一章节，将后面的内容都看完之后再回过头来看这部分内容。

`Tree`是beego中非常关键的一个维护请求路径和请求处理函数的一个类型，读懂了`Tree`，就基本能读懂beego的处理逻辑：

- 在注册路由时，将路由按照`/`进行分割，然后通过树的方式保存对应的`ControllerInfo`，类似于字典树
- 在收到请求时，匹配路由时，通过搜索树的方式来找到对应的处理函数

在了解`Tree`的处理逻辑之前，我们先来看一下beego支持的[正则路由](https://beego.me/docs/mvc/controller/router.md)规则：

- `web.Router(“/api/?:id”, &controllers.RController{})`

  默认匹配 //例如对于URL`/api/123`可以匹配成功，此时变量`:id`值为`123`，URL`/api/`可正常匹配

- `web.Router(“/api/:id”, &controllers.RController{})`

  默认匹配 //例如对于URL`/api/123`可以匹配成功，此时变量`:id`值为`123`，但URL`/api/`匹配失败

- `web.Router(“/api/:id([0-9]+)“, &controllers.RController{})`

  自定义正则匹配 //例如对于URL`/api/123`可以匹配成功，此时变量`:id`值为`123`

- `web.Router(“/user/:username([\\w]+)“, &controllers.RController{})`

  正则字符串匹配 //例如对于URL`/user/astaxie`可以匹配成功，此时变量`:username`值为`astaxie`

- `web.Router(“/download/*.*”, &controllers.RController{})`

  *匹配方式 //例如对于URL`/download/file/api.xml`可以匹配成功，此时变量`:path`值为`file/api`， `:ext`值为`xml`

- `web.Router(“/download/ceshi/*“, &controllers.RController{})`

  *全匹配方式 //例如对于URL`/download/ceshi/file/api.json`可以匹配成功，此时变量`:splat`值为`file/api.json`

- `web.Router(“/:id:int”, &controllers.RController{})`

  int 类型设置方式，匹配`:id`为`int` 类型，框架帮你实现了正则 `([0-9]+)`

- `web.Router(“/:hi:string”, &controllers.RController{})`

  string 类型设置方式，匹配`:hi` 为`string` 类型。框架帮你实现了正则 `([\w]+)`

- `web.Router(“/cms_:id([0-9]+).html”, &controllers.CmsController{})`

  带有前缀的自定义正则 //匹配 `:id` 为正则类型。匹配 `cms_123.html` 这样的 url `:id = 123`

可以在 Controller 中通过如下方式获取上面的变量：

```go
this.Ctx.Input.Param(":id")
this.Ctx.Input.Param(":username")
this.Ctx.Input.Param(":splat")
this.Ctx.Input.Param(":path")
this.Ctx.Input.Param(":ext") // 注意beego仅支持".json"、".xml"、".html"这3中后缀
```

可以看到，注册路由时，`beego`的正则不仅支持`:id`、`:username`这种自定义变量从URL中取值的方式，还支持对这个自定义变量进行正则表达式`/api/:id([0-9]+)`或者数据类型的匹配`/:id:int`，设置还支持`/download/ceshi/*`和`/download/*.*`这种全匹配的方式，所以`Tree`中也需要分别对这些场景进行适配。我们直接来看一下`Tree`的数据结构便能一窥一二。

```go
// vendor/github.com/astaxie/beego/tree.go
// Tree has three elements: FixRouter/wildcard/leaves
// fixRouter stores Fixed Router 固定路由
// wildcard stores params 
// leaves store the endpoint information
type Tree struct {
	prefix string  // 路由前缀，仅在静态路由子节点时才会有值，根节点为空
	fixrouters []*Tree  // 子节点有静态路由时有值
	wildcard *Tree  // 子节点有正则时有值，查找时，如果找不到固定路由才会搜索这个
	//if set, failure to match wildcard search 匹配通配符搜索失败？
	leaves []*leafInfo // 用来保存当前节点的Controller信息，一般只有在叶子节点才会有值
}

type leafInfo struct {
	// 当前这个叶子节点倒根节点的所有通配符. eg, ["id" "name"] for the wildcard ":id" and ":name"
	wildcards []string
	regexps *regexp.Regexp  // 从跟节点到当前叶子的第一个正则开始对应的正则表达式
	runObject interface{}  // 其实承载的是前面的&ControllerInfo
}
```

`Tree`这个类型有两个最关键的方法：

- 一个是`AddRouter`，目的是根据注册的URL将对应的`ControllerInfo`高效地保存到树中，以下是主要代码的分析

```go
// vendor/github.com/astaxie/beego/tree.go
// 注册路由时最终保存URL与对应ControllerInfo的方法，注意runObject传进来的就是&ControllerInfo
func (t *Tree) AddRouter(pattern string, runObject interface{}) {
    // 这里的splitPath将URL进行strings.Split(pattern, "/")处理，返回各级字符串的slice
	t.addseg(splitPath(pattern), runObject, nil, "")
}

// "/"
// "admin" ->
func (t *Tree) addseg(segments []string, route interface{}, wildcards []string, reg string) {
	if len(segments) == 0 {
        // 1、已经到达叶子，则直接将这个ControllerInfo作为子节点加入到这个父节点的子节点中
		if reg != "" {
			t.leaves = append(t.leaves, &leafInfo{runObject: route, wildcards: wildcards, regexps: regexp.MustCompile("^" + reg + "$")})
		} else { // 
			t.leaves = append(t.leaves, &leafInfo{runObject: route, wildcards: wildcards})
		}
	} else {
        // 2、还未到达叶子，则处理剩余URL中最前面这个字符串
		seg := segments[0]
        // iswild表示有没有:id或者*这种特殊的匹配符，也就是当前是否是正则路由
        // param返回的是正则中的变量，如":id"、":splat"、":path"、":ext"这种
        // regexpStr返回的则是匹配这个变量的正则表达式
		iswild, params, regexpStr := splitSegment(seg) // 这个函数的代码其实也挺关键的，但是考虑篇幅就不写了
		// “?:id”这种格式，说明当前节点可以为空，把这种情况也算在内
		if len(params) > 0 && params[0] == ":" {
			t.addseg(segments[1:], route, wildcards, reg)
            params = params[1:]  // 把":"去掉
		}
		//Rule: /login/*/access match /login/2009/11/access
		//if already has *, and when loop the access, should as a regexpStr
        // a：如果前面的路由中已经有*了，那么即使当前是静态路由，也需要设置成正则路由，并且需要在正则表达式中增加当前字符串
		if !iswild && utils.InSlice(":splat", wildcards) {
			iswild = true
			regexpStr = seg
		}
		//Rule: /user/:id/*
        // b：如果当前是第一个*，并且前面也有正则
		if seg == "*" && len(wildcards) > 0 && reg == "" {
			regexpStr = "(.+)"
		}
		if iswild {
            // 2.1、当前节点是正则路由的情况下
			if t.wildcard == nil {
				t.wildcard = NewTree()  // 正则路由子节点
			}
            if regexpStr != "" { // ":id:int"、":id([0-9]+"或者前面的a/b这两种情况
				if reg == "" { // 这是URL中第一个带有正则表达式的字符串
					rr := ""
					for _, w := range wildcards {
						if w == ":splat" {
							rr = rr + "(.+)/"
						} else {
							rr = rr + "([^/]+)/"
						}
					}
					regexpStr = rr + regexpStr
				} else {
					regexpStr = "/" + regexpStr
				}
			} else if reg != "" {
				if seg == "*.*" {
					regexpStr = "/([^.]+).(.+)"
					params = params[1:] // "*.*"的params是[. :path :ext]，因此去掉.这个分隔符
				} else {
					for range params {
						regexpStr = "/([^/]+)" + regexpStr
					}
				}
			} else {
				if seg == "*.*" {
					params = params[1:]
				}
			}
			t.wildcard.addseg(segments[1:], route, append(wildcards, params...), reg+regexpStr)
		} else {
            // 2.2、当前节点是固定路由的场景下，先查看有没有已经存在的同名固定路由的子节点，没有则创建
            // 需要注意的是，Tree中的prefix就是用来保存这个固定路由的字符串
			var subTree *Tree
			for _, sub := range t.fixrouters {
				if sub.prefix == seg {
					subTree = sub
					break
				}
			}
			if subTree == nil {
				subTree = NewTree()
				subTree.prefix = seg // 创建的新Tree以当前这个seg作为前缀
				t.fixrouters = append(t.fixrouters, subTree) // 将这个子节点加入到固定路由列表中
			}
            // 在子节点中递归处理后面的路由规则，前面的通配符和正则表达式都会继续后传递
			subTree.addseg(segments[1:], route, wildcards, reg)
		}
	}
}
```

- 另一个是`Match`，也就是根据请求的URL找到对应的`ControllerInfo`信息，以下是主要代码的分析

```go
// Match router to runObject & params
func (t *Tree) Match(pattern string, ctx *context.Context) (runObject interface{}) {
	if len(pattern) == 0 || pattern[0] != '/' {
		return nil
	}
    // 用来保存URL中正则对应的值，例如":id"、":splat"等对应的实际值
	w := make([]string, 0, 20)
	return t.match(pattern[1:], pattern, w, ctx)
}

// pattern用于记录剩余的URL，treePattern用于记录需要正则匹配的URL部分
func (t *Tree) match(treePattern string, pattern string, wildcardValues []string, ctx *context.Context) (runObject interface{}) {
    // 去掉最前面的'/'
	if len(pattern) > 0 {
		i := 0
		for ; i < len(pattern) && pattern[i] == '/'; i++ {
		}
		pattern = pattern[i:]
	}
	// URL已经匹配完成，先看leaves中是否有匹配的，然后在看正则的子节点中是否有匹配的
	if len(pattern) == 0 {
		for _, l := range t.leaves {
			if ok := l.match(treePattern, wildcardValues, ctx); ok {
				return l.runObject
			}
		}
		if t.wildcard != nil {
			for _, l := range t.wildcard.leaves {
				if ok := l.match(treePattern, wildcardValues, ctx); ok {
					return l.runObject
				}
			}
		}
		return nil
	}
    // 取出当前URL的第一截
	var seg string
	i, l := 0, len(pattern)
	for ; i < l && pattern[i] != '/'; i++ {
	}
	if i == 0 {
		seg = pattern
		pattern = ""
	} else {
		seg = pattern[:i]
		pattern = pattern[i:]
	}
    // 1、先查到固定路由的子节点的prefix中是否匹配当前这一截，注意由于这里是固定路由，treePattern需要去掉当前的prefix
	for _, subTree := range t.fixrouters {
		if subTree.prefix == seg {
			if len(pattern) != 0 && pattern[0] == '/' {
				treePattern = pattern[1:]
			} else {
				treePattern = pattern
			}
			runObject = subTree.match(treePattern, pattern, wildcardValues, ctx)
			if runObject != nil {
				break
			}
		}
	}
	if runObject == nil && len(t.fixrouters) > 0 {
		// Filter the .json .xml .html extension
		for _, str := range allowSuffixExt {
			if strings.HasSuffix(seg, str) {
				for _, subTree := range t.fixrouters {
					if subTree.prefix == seg[:len(seg)-len(str)] {
						runObject = subTree.match(treePattern, pattern, wildcardValues, ctx)
						if runObject != nil {
							ctx.Input.SetParam(":ext", str[1:])
						}
					}
				}
			}
		}
	}
    // 2、然后查找正则的子节点中是否有匹配的
	if runObject == nil && t.wildcard != nil {
		runObject = t.wildcard.match(treePattern, pattern, append(wildcardValues, seg), ctx)
	}
    // 3、最后查找叶子中是否有匹配的
	if runObject == nil && len(t.leaves) > 0 {
		wildcardValues = append(wildcardValues, seg)
		start, i := 0, 0
		for ; i < len(pattern); i++ {
			if pattern[i] == '/' {
				if i != 0 && start < len(pattern) {
					wildcardValues = append(wildcardValues, pattern[start:i])
				}
				start = i + 1
				continue
			}
		}
		if start > 0 {
			wildcardValues = append(wildcardValues, pattern[start:i])
		}
		for _, l := range t.leaves {
			if ok := l.match(treePattern, wildcardValues, ctx); ok {
				return l.runObject
			}
		}
	}
	return runObject
}
```

以上有一点没有体现出来的是，在`t.leaves`中执行match时，对于URL中的实际值会设置到ctx中的param中，这也是为什么入参中一定要传入ctx。



#### 2.2、启动server

在1.2章节的例子中，在`init()`中注册完路由之后，`main`函数会调用`beego.Run()`来启动这个http server，接下来我们就一起看一下实际是如何启动的。

##### 2.2.1、`Run`方法

`beego.Run()`这个方法是一个拥有变长入参的函数，可以在入参中输入字符串用来设置server的监听ip和端口。

```go
// vendor/github.com/astaxie/beego/beego.go
// Run beego application.
// beego.Run() default run on HttpPort
// beego.Run("localhost")
// beego.Run(":8089")
// beego.Run("127.0.0.1:8089")
func Run(params ...string) {
	initBeforeHTTPRun() // http server的初始化注册，如设置默认处理handler等

	// 中间代码略，主要根据params配置server的监听ip和端口

	BeeApp.Run() // 实际的运行函数，这个也就是2.1.1中的App类型的方法，前面讲过APP中有http.Server和自己的Handler成员
}
```



```go
// 
// Run beego application.
func (app *App) Run(mws ...MiddleWare) {
	addr := BConfig.Listen.HTTPAddr

	if BConfig.Listen.HTTPPort != 0 {
		addr = fmt.Sprintf("%s:%d", BConfig.Listen.HTTPAddr, BConfig.Listen.HTTPPort)
	}

	var (
		err        error
		l          net.Listener
		endRunning = make(chan bool, 1)
	)

	// run cgi server
	// 启动cgi服务器的代码略

    // 关键逻辑：将自己的Handler（也就是ControllerRegister）赋予给http.Server.Handler，注意http.Server.Handler是一个interface（即ServeHTTP(ResponseWriter, *Request)）
	app.Server.Handler = app.Handlers
	for i := len(mws) - 1; i >= 0; i-- {
		if mws[i] == nil {
			continue
		}
		app.Server.Handler = mws[i](app.Server.Handler)
	}
	app.Server.ReadTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	app.Server.WriteTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	app.Server.ErrorLog = logs.GetLogger("HTTP")

	// run graceful mode
	// graceful mode代码略，默认情况下不开，本文也不仔细研究
    // ......

	// run normal mode
	// 启动https安全端口代码略
    // ......
    
    // 考虑到篇幅有限，这里只讨论http这种方式，https的方式只是在证书处理上有区别
	if BConfig.Listen.EnableHTTP {
		go func() { // 启动一个协程来启动server
			app.Server.Addr = addr
			logs.Info("http server Running on http://%s", app.Server.Addr)
			if BConfig.Listen.ListenTCP4 {
				ln, err := net.Listen("tcp4", app.Server.Addr)
				if err != nil {
					logs.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
					return
				}
                // 关键逻辑：实际就是调用
				if err = app.Server.Serve(ln); err != nil {
					logs.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
					return
				}
			} else {
				if err := app.Server.ListenAndServe(); err != nil {
					logs.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
				}
			}
		}()
	}
	<-endRunning
}
```

在上述逻辑中，最关键的点在这一行：

```go
app.Server.Handler = app.Handlers
```

我们知道，`app.Server.Handler`其实是Go官方http库中的关键interface，因此`app.Handlers`（实际是`ControllerRegister`）实现的这个方法就是实际处理请求对应的Handler了。

```go
// src/net/http/server.go
// A Handler responds to an HTTP request.
//
// ServeHTTP should write reply headers and data to the ResponseWriter
// and then return. Returning signals that the request is finished; it
// is not valid to use the ResponseWriter or read from the
// Request.Body after or concurrently with the completion of the
// ServeHTTP call.
//
// Depending on the HTTP client software, HTTP protocol version, and
// any intermediaries between the client and the Go server, it may not
// be possible to read from the Request.Body after writing to the
// ResponseWriter. Cautious handlers should read the Request.Body
// first, and then reply.
//
// Except for reading the body, handlers should not modify the
// provided Request.
//
// If ServeHTTP panics, the server (the caller of ServeHTTP) assumes
// that the effect of the panic was isolated to the active request.
// It recovers the panic, logs a stack trace to the server error log,
// and either closes the network connection or sends an HTTP/2
// RST_STREAM, depending on the HTTP protocol. To abort a handler so
// the client sees an interrupted response but the server doesn't log
// an error, panic with the value ErrAbortHandler.
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```



##### 2.2.2、`ControllerRegister`的`http.Handler`

最终我们定位到`beego`启动的server的处理逻辑如下：

- 根据请求从`ControllerRegister`的`Tree`中搜索对应的`ControllerInfo`
- 根据`ControllerInfo`中记录的Controller的反射信息创建对应的实例
- 执行这个Controller实例的Init/Prepare等前置方法，然后直接注册的功能方法，最后执行Finish等后置方法

详细的代码分析可以参考（部分非关键逻辑已经省略）：

```go
// vendor/github.com/astaxie/beego/router.go
// Implement http.Handler interface.
func (p *ControllerRegister) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
	startTime := time.Now()
	var (
		runRouter    reflect.Type  // Controller的类型反射
		findRouter   bool          // 是否是注册过的路由
		runMethod    string        // 改请求对应的Controller实例的方法
		methodParams []*param.MethodParam
		routerInfo   *ControllerInfo
		isRunnable   bool
	)
	context := p.GetContext()

	context.Reset(rw, r)

	defer p.GiveBackContext(context)
	if BConfig.RecoverFunc != nil {
		defer BConfig.RecoverFunc(context)
	}

	context.Output.EnableGzip = BConfig.EnableGzip

	if BConfig.RunMode == DEV {
		context.Output.Header("Server", BConfig.ServerName)
	}

	var urlPath = r.URL.Path

	if !BConfig.RouterCaseSensitive {
		urlPath = strings.ToLower(urlPath)
	}

	// filter wrong http method
	if !HTTPMETHOD[r.Method] {
		exception("405", context)
		goto Admin
	}

	// filter for static file
	if len(p.filters[BeforeStatic]) > 0 && p.execFilter(context, urlPath, BeforeStatic) {
		goto Admin
	}

	serverStaticRouter(context)

	if context.ResponseWriter.Started {
		findRouter = true
		goto Admin
	}

	if r.Method != http.MethodGet && r.Method != http.MethodHead {
		if BConfig.CopyRequestBody && !context.Input.IsUpload() {
			context.Input.CopyBody(BConfig.MaxMemory)
		}
		context.Input.ParseFormOrMulitForm(BConfig.MaxMemory)
	}

	// session init
	if BConfig.WebConfig.Session.SessionOn {
		var err error
		context.Input.CruSession, err = GlobalSessions.SessionStart(rw, r)
		if err != nil {
			logs.Error(err)
			exception("503", context)
			goto Admin
		}
		defer func() {
			if context.Input.CruSession != nil {
				context.Input.CruSession.SessionRelease(rw)
			}
		}()
	}
	if len(p.filters[BeforeRouter]) > 0 && p.execFilter(context, urlPath, BeforeRouter) {
		goto Admin
	}
	// User can define RunController and RunMethod in filter
	if context.Input.RunController != nil && context.Input.RunMethod != "" {
		findRouter = true
		runMethod = context.Input.RunMethod
		runRouter = context.Input.RunController
	} else {
        // 关键逻辑，找到改请求对应的ControllerInfo，在这个过程中也会把":id"、":splat"等值写入到ctx的params中
		routerInfo, findRouter = p.FindRouter(context)
	}

	// if no matches to url, throw a not found exception
	if !findRouter {
		exception("404", context)
		goto Admin
	}
	if splat := context.Input.Param(":splat"); splat != "" {
		for k, v := range strings.Split(splat, "/") {
			context.Input.SetParam(strconv.Itoa(k), v)
		}
	}

	if routerInfo != nil {
		// store router pattern into context
		context.Input.SetData("RouterPattern", routerInfo.pattern)
	}

	// execute middleware filters
	if len(p.filters[BeforeExec]) > 0 && p.execFilter(context, urlPath, BeforeExec) {
		goto Admin
	}

	// check policies
	if p.execPolicy(context, urlPath) {
		goto Admin
	}

	if routerInfo != nil {
		if routerInfo.routerType == routerTypeRESTFul {
            // routerTypeRESTFul：AddMethod()的方式注册的路由
			if _, ok := routerInfo.methods[r.Method]; ok {
				isRunnable = true
				routerInfo.runFunction(context)
			} else {
				exception("405", context)
				goto Admin
			}
		} else if routerInfo.routerType == routerTypeHandler {
            // routerTypeHandler：Handler()的方式注册的路由
			isRunnable = true
			routerInfo.handler.ServeHTTP(context.ResponseWriter, context.Request)
		} else {
            // routerTypeBeego：Add()或者AddAuto()的方式注册的路由，前者就是2.1节中的方式
			runRouter = routerInfo.controllerType
			methodParams = routerInfo.methodParams
			method := r.Method
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodPut {
				method = http.MethodPut
			}
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodDelete {
				method = http.MethodDelete
			}
			if m, ok := routerInfo.methods[method]; ok {
				runMethod = m
			} else if m, ok = routerInfo.methods["*"]; ok {
				runMethod = m
			} else {
				runMethod = method
			}
		}
	}

	// also defined runRouter & runMethod from filter
	if !isRunnable {
		// Invoke the request handler
		var execController ControllerInterface
		if routerInfo != nil && routerInfo.initialize != nil {
			execController = routerInfo.initialize()
		} else {
            // 关键逻辑：实例化一个Controller，如1.2样例中的SampleController
			vc := reflect.New(runRouter)
			var ok bool
			execController, ok = vc.Interface().(ControllerInterface)
			if !ok {
				panic("controller is not ControllerInterface")
			}
		}

		// 调用实例的Init方法
		execController.Init(context, runRouter.Name(), runMethod, execController)

		// 调用实例的Prepare方法
		execController.Prepare()

		// 调用XSRF方法略 

        // URLMapping
		execController.URLMapping()

		if !context.ResponseWriter.Started {
			// exec main logic
			switch runMethod {
			case http.MethodGet:
				execController.Get()
			case http.MethodPost:
				execController.Post()
			case http.MethodDelete:
				execController.Delete()
			case http.MethodPut:
				execController.Put()
			case http.MethodHead:
				execController.Head()
			case http.MethodPatch:
				execController.Patch()
			case http.MethodOptions:
				execController.Options()
			case http.MethodTrace:
				execController.Trace()
			default:
				if !execController.HandlerFunc(runMethod) {
					vc := reflect.ValueOf(execController)
                    // 根据之前注册的Controller的函数名找到对应的方法
					method := vc.MethodByName(runMethod)
					in := param.ConvertParams(methodParams, method.Type(), context)
                    // 关键逻辑，调用注册的方法，如1.2中的Hello()
					out := method.Call(in)

					// For backward compatibility we only handle response if we had incoming methodParams
					if methodParams != nil {
						p.handleParamResponse(context, execController, out)
					}
				}
			}

			// render template
			if !context.ResponseWriter.Started && context.Output.Status == 0 {
				if BConfig.WebConfig.AutoRender {
					if err := execController.Render(); err != nil {
						logs.Error(err)
					}
				}
			}
		}

		// 调用实例的Finish方法
		execController.Finish()
	}

	// 执行中间件的filter函数

Admin:
	// 统计QPS数据，代码略
}
```



##### 2.2.3、`FindRouter`

2.2.2中有一步是查找是否存在这个URL对应的处理方法，对应的代码比较简单，主要就是调用`Tree`的Match方法。

```go
// vendor/github.com/astaxie/beego/router.go
// FindRouter Find Router info for URL
func (p *ControllerRegister) FindRouter(context *beecontext.Context) (routerInfo *ControllerInfo, isFind bool) {
	var urlPath = context.Input.URL()
	if !BConfig.RouterCaseSensitive {
		urlPath = strings.ToLower(urlPath)
	}
	httpMethod := context.Input.Method()
	if t, ok := p.routers[httpMethod]; ok {
        // 这里实际就去执行2.1.3.2中Tree的Match方法
		runObject := t.Match(urlPath, context)
		if r, ok := runObject.(*ControllerInfo); ok {
			return r, true
		}
	}
	return
}
```



### 3、总结

通过案例与`beego`源码的解析，可以看出，`beego`这个http框架的核心在于充分使用Go语言的反射机制来进行抽象，通过`Tree`来对注册的路由进行保存和匹配，同时提供一个基本的`Controller`类型提供给开发者进行内嵌，开发者只需要实现自己的处理方法然后在初始化时进行注册即可。



### 4、参考

[Golang的反射reflect深入理解和示例](https://juejin.cn/post/6844903559335526407)

[Go 语言反射的实现原理](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/)







