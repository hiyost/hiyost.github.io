---
title: 深入理解k8s中的list-watch机制
date: 2019-08-01 21:06:45
tags: list-watch, k8s, client-go
---



[TOC]



记得三年前刚转正的时候就说要把k8s里面的`list-watch`搞透彻然后给大家讲一下，结果这个坑一挖就是三年，现在终于能抽出点时间将这个坑填平，也希望以后多花点时间读读源码，打好基础。希望后续还可以把`etcd`、`docker`、`istio`等组件的代码也好好走读一遍，开拓视野，而不只是拉通扯皮搞业务。



言归正传，本文的目的是将list-watch的机制搞清楚，各位看官且往下看。



> 说明：本文使用的k8s代码为[1.13版本](<https://github.com/kubernetes/kubernetes/tree/release-1.13>)，其他版本代码可能会有少许差异。
>
> 名称解释：
>
> - 接口：
> - 对象：
> - 结构体：
> - 资源：
> - 元素：



# 0、啥是`list-watch`啊



在k8s中，`list-watch`本质上还是client端监听k8s资源变化并作出相应处理的生产者消费者框架。因此在分析这部分的代码之前就会有问题在脑海中：生产者是谁？消费者是谁？传递了啥？怎么传递的？



除此之外，往往生产者只有一个，消费者有多个，这种情况下怎么保证每个消费者都能收到消息？



好，带着这些问题，我们慢慢往下看。



## 0.1、`list`与`watch`





## 0.2、使用场景







## 0.3、代码目录

基本数据结构和算法的代码主要位于`staging/src/k8s.io/client-go/tools/cache`这个目录中，其中去掉UT测试的文件之后约3.6k代码，十分精巧：

```shell
.
├── BUILD
├── controller.go
├── delta_fifo.go
├── doc.go
├── expiration_cache_fakes.go
├── expiration_cache.go
├── fake_custom_store.go
├── fifo.go
├── heap.go
├── index.go
├── listers.go
├── listwatch.go
├── mutation_cache.go
├── mutation_detector.go
├── OWNERS
├── reflector.go
├── reflector_metrics.go
├── shared_informer.go
├── store.go
├── thread_safe_store.go
└── undelta_store.go
```



## 0.4、一个使用`list-watch`的简单例子







# 1、数据结构



## 1.1、基础数据结构`threadSafeMap`和`cache`



### 1.1.1、`threadSafeMap`

前面说到`list-watch`本质上是一个生产者消费者框架，最关键的是数据的传递，那么传递的数据的数据结构很重要，在`list-watch`中，最为底层的数据结构其实是一个线程安全的map：

代码位于`staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go`

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

可以看到这里的`threadSafeMap`的结构非常简单：

- `lock sync.RWMutex`是一个读写锁，用来保证线程安全
- `items map[string]interface{}`，这是一个`key`为`string`、`value`是任意类型的`map`，也是实际数据存放的最关键的数据结构，所有的操作最终都是跟这个`map`打交道，`key`是存储对象的唯一索引，`value`是存储的对象
- `indexers Indexers`是用于给`map`中的`value`数据做检索（也就是计算`value`的`key`）的函数，实际的数据结构是`map[string]IndexFunc`，注意可以有多个索引函数，且每个函数的返回值可以是多个。可以把`IndexFunc`理解为聚类函数。常用的`IndexFunc`有`MetaNamespaceIndexFunc`（返回对象的`namespace`）和`indexByPodNodeName`（返回Pod对象所在节点的名字）。
- `indices Indices`则是保存索引后的数据的`map[][]sets.String`，第一个key是`Indexers`中索引函数的名字，第二个key是这个索引函数返回值的索引，`sets.String`则是对应该索引值的所有`items`对象的key，这是一个已经按照索引分类好了的三维map。

> 注：关于`threadSafeMap`中的`indexers`和`indices`这两个元素的作用，后面会在`Indexer`中继续说明，这其实是`list-watch`之所以高效的一个重要原因。

#### 1.1.1.1、实现的接口`ThreadSafeStore`

不难看到`threadSafeMap`是一个私有的结构体，实际在使用时是通过它所实现的接口`ThreadSafeStore`来对里面的`map`进行数据操作：

```go
// ThreadSafeStore 本质上是threadSafeMap实现的所有接口，也是对其中数据的封装，防止外部直接使用map
type ThreadSafeStore interface {
    // 这部分是通过key对map进行增删改查等数据相关的基本操作，这些操作中会对indices表进行“日常”维护
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
    // Replace 的第二个入参是resourceVersion，但是函数中似乎并没有用到
	Replace(map[string]interface{}, string)
    // 这部分是一些高级的查询方法，主要是些同类索引的查询，后面的Indexer接口的实现依赖这部分功能
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// 给threadSafeMap增加Indexers函数，一定要在初始化（map中还没有数据）的时候做，否则直接报错
	AddIndexers(newIndexers Indexers) error
    // 暂时还未实现任何功能的函数，实际直接返回了nil
	Resync() error
}
```

可以看到这里定义了包括增删改查等所有对数据进行操作的函数，而前面所说的`threadSafeMap`均实现了这里面所有的接口，因此在实际使用的时候都是通过`ThreadSafeStore`来进行操作，这样做的目的也很容易理解：将数据隐藏，只暴露操作这些数据的函数。实际就是面向对象的理念。

#### 1.1.1.2、如何创建

在创建新的数据时，由于`threadSafeMap`是这个package里面的私有数据，虽然创建的是`threadSafeMap`结构体，但函数真正的返回值是`ThreadSafeStore`这个接口

```go
func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
	return &threadSafeMap{
		items:    map[string]interface{}{},
		indexers: indexers,
		indices:  indices,
	}
}
```

`NewThreadSafeStore`的被调方有2个，一个是创建`Store`的时候，另一个是创建`Indexer`的时候，后面会讲到。



这里有2个问题没有搞明白：

- `threadSafeMap`的定义里面`items map[string]interface{}`，但是在创建时`items:    map[string]interface{}{}`，后面的`{}`其实是对这个map的初始化
- `NewThreadSafeStore`函数中似乎并没有对`lock  sync.RWMutex`进行显式初始化，go语言的特性？



### 1.1.2、`cache`

了解了最底层的数据结构`threadSafeMap`和其实现的接口`ThreadSafeStore`之后，接下来看其更上一层的封装`cache`和其实现的接口`Store`和`Indexer`。注意`cache`既是这个结构体的名字，也是当前这个package的名字，可见其重要性。



可以看到`cache`相对于`threadSafeMap`而言最大的区别是多了一个`KeyFunc`（其定义是通过`obj`返回其对应的`key`字符串，这个`key`主要是给map做**唯一**索引用的）。

代码位于`staging/src/k8s.io/client-go/tools/cache/store.go`

```go
// cache定义了对出对象的key索引函数，并实现了ThreadSafeStore的所有方法
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	cacheStorage ThreadSafeStore
    // keyFunc用来确定对象的唯一性索引值，也就是计算map中的key值，最常用的keyFunc是MetaNamespaceKeyFunc，即取对象的“{namespace}/{name}”作为key
	keyFunc KeyFunc
}

type KeyFunc func(obj interface{}) (string, error)

// 用于确认cache实现了Store的所有接口，否则编译会出错（疑问：为啥Indexer不需要确认？）
var _ Store = &cache{}
```





#### 1.1.2.1、实现的接口`Store`

可以看到，实现了`Store`接口的`cache`结构体相比于实现了`ThreadSafeStore`接口的`threadSafeMap`结构体也只是多了一个`keyFunc`而已，并且`Store`中没有用到`indexers`函数，仅仅只是一个通过key值来做索引的map。常用的`keyFunc`是`MetaNamespaceKeyFunc`，即取对象的`{namespace}/{name}`作为key。

```go
// Store是一个通用的对象存储接口，并没有index的能力，因此常常用来做list-watch的队列载体
// 只不过cache和后面讲到的队列都实现了这个接口而已
type Store interface {
    // 这部分是对map进行增删改查等数据相关的基本操作，不需要做index
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)
	// Replace的第二个入参是resourceVersion，cache并没有用到，后面的队列中会用来
	Replace([]interface{}, string) error
    // Resync主要是后面讲到的队列用到的，cache中并没有使用到
	Resync() error
}
```

之所以先定义一套`Store`接口，是因为有很多地方其实并不需要使用索引功能，而只是需要一个线程安全的map而已（例如队列`Queue`）。

#### 1.1.2.2、实现的接口`Indexer`

**`cache`除了实现了`Store`的接口以外，还实现了`Indexers`的接口**，前者是`informer`使用的方式，后者是`reflector`的使用方式，注意这一点十分重要。`Indexer`本质上就是一个可以定义索引器的`Store`：

```go
// Indexer是一个带有索引功能的对象存储接口，通常用来作为list-watch的缓存器
type Indexer interface {
	Store
	// 获取通过indexName这个函数算出来的索引值和obj这个元素相同的所有元素（也就是找obj的同类）
	Index(indexName string, obj interface{}) ([]interface{}, error)
	// 获取通过indexName这个函数能算出来索引值包含indexKey值的所有元素在map中的key
	IndexKeys(indexName, indexKey string) ([]string, error)
	// 获取当前元素通过indexName这个函数能算出来的所有索引值（并非map的key）
	ListIndexFuncValues(indexName string) []string
    // 获取通过indexName这个函数算出来包含有indexKey值的所有元素（与IndexKeys()函数类似）
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	// 获取所有的indexers函数
	GetIndexers() Indexers
	// 增加Indexers函数，必须在初始化时（map中没有元素）添加，否则直接报错
	AddIndexers(newIndexers Indexers) error
}
```



`Indexer`可以说是最具有效率的一套存储器接口，原因在于它实现了根据索引来实时分类的功能，什么意思呢？当你定义了这个`Indexer`的`IndexFunc`，那么一旦有元素加入到这个`Store`中来，就会自动给这个元素分组（根据`IndexFunc`计算的key来进行分组）。这有什么好处呢？假如我需要根据`namespace`来进行索引（在k8s中这是最常见的索引方式），那么`IndexFunc`就是根据obj来返回其对应的`namespace`的值。设想一下如果没有自动索引的功能，我想要查询某个`namespace`下的所有对象，就需要遍历所有的对象并选出在这个`namespace`下的对应，需要做一次全量的遍历，而实现了实时索引功能的`indexer`就不通，由于我已经实现定义了`IndexFunc`，在每个对象增加进来的时候，我已经根据`namespace`来分类了（通过`threadSafeMap`中的`indices`来维护，`indices`实际是个` map[string][string]sets.String`），那么需要查询这个`namespace`下的所有对象时就只需要直接返回这个`set`就可以了，不需要遍历所有的对象。当前这个需要在每个对象发生变化的时候事先索引，存在一定的开销，但是对于查询操作带来的便捷是十分巨大的。



以上描述都是通过基本的接口和数据结构来实现的，对应的代码都在`staging/src/k8s.io/client-go/tools/cache/index.go`中。

首先是检索函数`IndexFunc`，它定义如下：

```go
// IndexFunc knows how to provide an indexed value for an object.
type IndexFunc func(obj interface{}) ([]string, error)
```

需要注意以下几点：

- `IndexFunc`的返回值是一个字符串的`slice`，而不是一个单独的字符串，这意味着`IndexFunc`可以是复合类型的索引。疑问：是否过度设计？
- `Indexer`的`IndexFunc`和`cache`的`keyFunc`不同，`keyFunc`计算的是对象的**唯一确定性**key（所以返回的是一个字符串而不是`slice`），不同对象经过`keyFunc`计算出来的key在这个`store`中是全局唯一的；而`IndexFunc`是一类对象的key，可能有多个对象都能通过这个`IndexFunc`计算出相同的key，这也是设计`IndexFunc`的初衷



为了实现这些逻辑，`indexer`中还设计了几个关键的数据结构（这几个实际上是`threadSafeMap`中的元素）

```go
// Index的key是IndexFunc计算出来的索引值，value是计算出相同索引值的元素的key
type Index map[string]sets.String
// Indexers的key是IndexFunc的名称，value是这个IndexFunc
type Indexers map[string]IndexFunc
// Indices的key是IndexFunc的名称，value是Index
type Indices map[string]Index
```

其中最需要注意的是`type Index map[string]sets.String`，这里面的`sets.String`存放的其实就是`Store`中`map`的key（也就是`keyFunc`返回的对象的**唯一确定性**key）。



#### 1.1.2.3、如何创建

由于`cache`是这个package中的私有数据，并不用来单独使用，而是通过其实现的两个接口`Store`和`Indexer`来使用：

```go
// NewStore返回的只是一个多了keyFunc的threadSafeStore，但是Indexers是空的
func NewStore(keyFunc KeyFunc) Store {
	return &cache{
		cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
		keyFunc:      keyFunc,
	}
}

// 可以看到Indexer只是比Store多了Indexers函数而已
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		keyFunc:      keyFunc,
	}
}
```



这里需要想办法举一个例子。



### 1.1.3、总结

本章节介绍了`list-watch`机制中最基础的数据结构`threadSafeMap`和`cache`，其本质是对一个`map[string]interface{}`进行增删改查，`cache`比`threadSafeMap`多了一个可以定义计算唯一索引方法（map的key值）的函数。为了便于对这个map中的对象进行分类，`threadSafeMap`中使用了`indexers`和`indices`这两个元素，用来维护根据不同维度来对所有对象进行分类的map。`cache`基于`threadSafeMap`中的函数实现了`Store`和`Indexer`两个interface，区别在于前者是不带分类功能（`indexers`为空），后者带分类功能。从interface上来讲，关系则稍显复杂，可以把`Store`当做最基本的操作interface，可拓展的方向很多，`Indexer`则是其中一种拓展，主要是增加了分类的功能，是`Informer`的基础。后面会讲到的`Queue`也是`Store`的一种拓展，增加了队列相关的出队列（`Pop`）功能。



这其中几点关键信息需要记住：

- 本章最重要的数据结构是`cache`这个结构体
- `cache`本质上是一个具有检索和自动分类功能map
- `cache`实现了`Store`和`Indexer`两个接口，后者是前者的拓展，比前者多的就是自动分类功能
- `Store`是`Reflector`的基础数据结构
- `Indexer`是`Informer`的基础数据结构



> 截止到目前，我们看到的主要就是跟`cache`相关的数据结构和接口，那么这些跟生产者消费者有什么关系呢？更确切地说，他们跟`list-watch`是什么关系呢？其实这个`cache`是`list-watch`的全量资源缓存，用来将提高查询的效率，降低client和server端的cpu消耗。当然这只是"然"，还需要搞懂"所以然"，我们带着这些疑问继续往下看。
>



## 1.2、基础数据结构`FIFO`和`DeltaFIFO`



在生产者消费者模型中，最重要的一个数据结构实际上是队列：

- 生产者将消息或者数据放入到队列中
- 消费者从队列中取出消息或者数据并对其进行处理



### 1.2.1、`FIFO`

由于go语言的官方库中并没有实现队列这个数据结构，k8s中自己实现了一套。`FIFO`是一个先入先出的队列，其中存放关键数据的元素是`items map[string]interface{}`和`queue []string`，`items`中存放的是所有的元素，`queue`中是通过map中的key值来存放的实际队列（通过slice实现，新增的元素append到最后，pop时取第一个），也就是说`FIFO`是通过map和slice这两个基础类型来配合实现的。

```go
// FIFO 的生产者是Reflector，消费者调用Pop()函数来获取队首的元素
// 当多个生产者向FIFO中增加元素时，FIFO会进行合并处理，只保留最新的元素
// 当消费者使用Pop()函数时获取的也是这个元素的最新版本
type FIFO struct {
	lock sync.RWMutex
    // cond是一个条件变量，消费者执行Pop()时使用它来wait，生产者将元素加入队列之后通过它来通知消费者
	cond sync.Cond
	// item存放队列中的实际元素
	items map[string]interface{}
    // queue中仅存放队列中元素的key值（注意queue和items中的数据必须是同步的）
	queue []string
	// 表示队列已经开始运作了（Delete/Add/Update/Replace被调用时都会将其设置为true）
	populated bool
    // 第一次执行Replace()（也就是第一次批量添加元素）时队列中元素的个数，执行Pop()时会递减
	initialPopulationCount int
	// keyFunc用于计算元素的唯一索引key值
	keyFunc KeyFunc
    // 用来设置这个队列的状态为关闭，并通知所有的消费者可以退出Pop()中的等待
	closed     bool
	closedLock sync.Mutex
}

var (
	_ = Queue(&FIFO{}) // FIFO is a Queue
)
```



#### 如何创建

创建一个新的`FIFO`只需要传入进行唯一索引值计算的`KeyFunc`就可以了

```go
// NewFIFO returns a Store which can be used to queue up items to
// process.
func NewFIFO(keyFunc KeyFunc) *FIFO {
	f := &FIFO{
		items:   map[string]interface{}{},
		queue:   []string{},
		keyFunc: keyFunc,
	}
	f.cond.L = &f.lock
	return f
}
```

实际上在k8s的大部分场景并没有使用到`FIFO`，仅scheduler中使用到了，而在list-watch中主要使用到了后面讲到的`DeltaFIFO`。



### 1.2.2、`DeltaFIFO`

在`FIFO`中，队列中的元素（实际是map中的value，而不是slice中的key）是一个无状态的对象`interface{}`，这对于watch事件而言是不够的，watch中需要知道这个对象是新增、删除还是更新，因此k8s又封装了一个叫做`DeltaFIFO`的队列，主要区别在于结构体中的元素`items map[string]interface{}`改成了`items map[string]Deltas`，你可以把他看成是消息或者事件。

```go
// DeltaFIFO的生产者是Reflector，消费者调用Pop()函数来获取队首的元素
// 如果有多个线程中调用，那么他们会获取到版本号稍微不同的元素
// 相较于FIFO，DeltaFIFO队列中可以处理删除的元素（Delete()函数也是往队列中添加元素）
type DeltaFIFO struct {
	lock sync.RWMutex
    // cond是一个条件变量，消费者执行Pop()时使用它来wait，生产者将元素加入队列之后通过它来通知消费者
	cond sync.Cond
	// DeltaFIFO最大的区别在于value是一个Deltas，包含了事件（Added/Updated/Deleted/Sync）信息
	items map[string]Deltas
    // queue中仅存放队列中元素的key值（注意queue和items中的数据必须是同步的，Resync()函数的工作）
	queue []string
    // 表示队列已经开始运作了（Delete/Add/Update/Replace被调用时都会将其设置为true）
	populated bool
    // 第一次执行Replace()（也就是第一次批量添加元素）时队列中元素的个数，执行Pop()时会递减
	initialPopulationCount int
    // keyFunc用于计算元素的唯一索引key值（留一个小悬念：这个跟cache中的keyFunc是否可以不同？）
	keyFunc KeyFunc
	// 这也是跟FIFO不同的一个地方，实际上就是已知可以操作的所有对象的集合，在Replace/Resync/Delete时需要用来判断该元素是否需要入队"Deleted"的事件，这其实是informer中维护的一个Indexer，包含了全量的缓存，DeltaFIFO中只能对其进行GetByKey和ListKeys，无法进行增删改查
	knownObjects KeyListerGetter
	closed     bool
	closedLock sync.Mutex
}

var (
	_ = Queue(&DeltaFIFO{}) // DeltaFIFO is a Queue
)
```

从结构体中可以看到`DeltaFIFO`与`FIFO`的两处区别：

- `item`中的value从`interface{}`变成了`Deltas`，`Deltas`是一个带有事件类型`DeltaType`的`interface{}`。

```go
// DeltaType is the type of a change (addition, deletion, etc)
type DeltaType string

const (
	Added   DeltaType = "Added"
	Updated DeltaType = "Updated"
	Deleted DeltaType = "Deleted"
	// 新的list-watch周期产生的replace类型（list-watch期间可能会失败，触发重新list-watch）
    // 一般在执行Replace或者Resync操作的时候会产生
	Sync DeltaType = "Sync"
)
// 包含了变化类型的队列元素
type Delta struct {
	Type   DeltaType
	Object interface{}
}
// 注意Deltas是一个slice，也是这个资源产生的所有变化，Pop时出队的也是一整个slice
// 最新的变化在slice末尾
type Deltas []Delta
```

- 多了一个`knownObjects`字段，其类型是`KeyListerGetter`这个接口，实际创建`DeltaFIFO`时这里传进来的是一个`cache`，由于`KeyListerGetter`只有`GetByKey`和`ListKeys`，因此`knownObjects`只具有读权限

```go
// A KeyListerGetter is anything that knows how to list its keys and look up by key.
type KeyListerGetter interface {
	KeyLister
	KeyGetter
}

// A KeyLister is anything that knows how to list its keys.
type KeyLister interface {
	ListKeys() []string
}

// A KeyGetter is anything that knows how to get the value stored under a given key.
type KeyGetter interface {
	GetByKey(key string) (interface{}, bool, error)
}
```





#### 如何创建



创建一个`DeltaFIFO`除了要传入进行唯一索引值计算的`KeyFunc`以外，还需要传入实现了`KeyListerGetter`接口的对象。从前文的解析中可以发现，`Store`和`Indexer`中均实现了`KeyListerGetter`，由此不难想到这里传入的要么就是`Store`要么就是`Indexer`了。于是队列（`DeltaFIFO`）就和缓存（`cache`）链接了起来，这也是list-watch机制中至关重要的一个联系。

```go
func NewDeltaFIFO(keyFunc KeyFunc, knownObjects KeyListerGetter) *DeltaFIFO {
	f := &DeltaFIFO{
		items:        map[string]Deltas{},
		queue:        []string{},
		keyFunc:      keyFunc,
		knownObjects: knownObjects,
	}
	f.cond.L = &f.lock
	return f
}
```



### 1.2.3、实现的接口`Queue`

好，当前这一小节到目前为止都是在说队列的数据结构，其实外部传递的还是`FIFO`和`DeltaFIFO`实现的`Queue`这个接口：

```go
// Queue是Store的一个拓展，具有队列的Pop功能
type Queue interface {
	Store
    // 当队列中没有元素时执行Pop会阻塞，直到有生产者往里面增加元素才会处理
	// 返回Pop出来的元素以及经过PopProcessFunc函数处理的结果，若PopProcessFunc处理失败会重新加入队列
	// type PopProcessFunc func(interface{}) error
    // 注意PopProcessFunc十分关键，这是我们使用list-watch真正能传进去的函数
	Pop(PopProcessFunc) (interface{}, error)
	// 当这个元素在队列中不存在时才会加入到队列中（注意跟Add的区别：Add会直接加入或刷新到队列中）
	AddIfNotPresent(interface{}) error
	// 判断第一批加进去的元素是否都已经处理完毕
	HasSynced() bool
	// 关闭这个队列，同时也会让对应的消费者关闭等待
	Close()
}
```

可以看到，辛辛苦苦讲一节，最终还是回到了原点：`Queue`其实是增加了队列功能的`Store`，相比而言只多了4个接口而已。关于`Store`接口可以看看上一节。另外还有一个关键的信息是，无论是`FIFO`还是`DeltaFIFO`，除了队列的操作以外，同`cache`一样也实现了`Store`接口。



### 1.2.4、和`cache`/`FIFO`的比较



#### 1.2.4.1、结构体成员的区别

这里先对比一下`cache`结构体：

- cache中有`indexers` 和`indices`来实现同类型资源的分类功能，FIFO/DeltaFIFO中没有，也无需处理
- FIFO/DeltaFIFO中同样需要有map来保存所有的原始数据
- FIFO/DeltaFIFO中多了一个通过slice实现的queue队列，这个队列中只保存需要处理的元素的key值，以减小队列的大小
- FIFO中多了一个条件变量`sync.Cond`，用来做消息同步（Pop时如果队列为空则等待有新元素加入的消息）
- DeltaFIFO相较于FIFO而言map中的值（`Deltas`）不一样，可以处理元素的操作类型，甚至可以处理元素的删除事件，并且当执行Pop出队时返回的是一个slice，
- DeltaFIFO相较于FIFO多了一个knownObjects，这个在list-watch的时候十分重要，一旦某个周期中的list-watch失败了，会触发新的list-watch周期，在这个周期的开始阶段，会list一把最新的资源，并更新队列中的所有数据



| cache                        | FIFO                         | DeltaFIFO                    |
| ---------------------------- | ---------------------------- | ---------------------------- |
| lock  sync.RWMutex           | lock sync.RWMutex            | lock sync.RWMutex            |
| items map[string]interface{} | items map[string]interface{} | items map[string]**Deltas**  |
| keyFunc KeyFunc              | keyFunc KeyFunc              | keyFunc KeyFunc              |
| indexers Indexers            | cond sync.Cond               | cond sync.Cond               |
| indices Indices              | queue []string               | queue []string               |
|                              | populated bool               | populated bool               |
|                              | initialPopulationCount int   | initialPopulationCount int   |
|                              | closed bool                  | closed bool                  |
|                              | closedLock sync.Mutex        | closedLock sync.Mutex        |
|                              |                              | knownObjects KeyListerGetter |



#### 1.2.4.2、接口实现的区别

我们对比一下目前已经讲过的几个接口：

| 接口 | ThreadSafeStore       | Store                 | Indexer        | Queue          |
| ---- | --------------------- | --------------------- | -------------- | -------------- |
|      | ThreadSafeStore的接口 | 继承并增加了`KeyFunc` | 实现了`Store`  | 实现`Store`    |
|      | NA                    | NA                    | 增加了索引功能 | 增加了队列功能 |
| 对象 | threadSafeMap         | cache/FIFO/DeltaFIFO  | cache          | FIFO/DeltaFIFO |

可以看到，实现了`Indexer`和`Queue`接口的对象其实也实现了`Store`接口，因此`FIFO`和`DeltaFIFO`也都实现了`Store`的接口。但是与`cache`不同的是，`FIFO`和`DeltaFIFO`所实现的`Store`中的`Resync()`和`Replace()`完全不同，可以看到`Resync()`其实是专门为队列设计的：

| Store接口   | `cache`            | `FIFO`           | `DeltaFIFO`                  |
| ----------- | ------------------ | ---------------- | ---------------------------- |
| `Replace()` | 重建items和indices | 更新items和queue | 更新items和queue（该删的删） |
| `Resync()`  | 无具体实现         | 同步items与queue | 同步items与queue             |

这里其实需要重点讲一下`DeltaFIFO`的`Replace()`和`Resync()`，建议大家去看看实际的代码实现：

- `Replace(list []interface{})`：将新的list全部以"Sync"事件加入到队列中（即使这个元素不在当前队列中），然后将新的list中已经没有的元素增加其"Deleted"事件（这个过程比较有意思，如果`DeltaFIFO`的`knownObjects`不为空就遍历`knownObjects`，否则遍历`DeltaFIFO`的`items`，list中没有就"Deleted"）
- `Resync()`：如果`DeltaFIFO`的`knownObjects`为空就直接返回，如果不为空则将`knownObjects`中的所有元素以"Sync"事件加入到队列中（估计是因为有这个机制，所以`Replace`中优先使用`knownObjects`来遍历）



#### 1.2.4.3、功能的区别



通过结构体和接口实现的区别不难窥见：

- `cache`用来存放最新的全量数据（无论是否变化）
- `FIFO`/`DeltaFIFO`用来存放变化（增/删/改/同步）的数据

也就是说，`cache`中存放的是list出来的所有数据（很多数据可能是一直没有修改），`FIFO`/`DeltaFIFO`中存放的是watch到的变化（增删改）并且是需要去做对应处理的数据，这一点对于理解为什么要同时有`cache`/`FIFO`/`DeltaFIFO`这几个结构体、`Store`/`Indexer`/`Queue`这几个接口十分重要，后面讲到的关键数据结构`Reflector`和`Informer`也依赖于前面的这些基本的数据结构和接口。这时候你可能已经猜到这几个基本数据结构在整个list-watch中的角色了，我们接着往下看。





### 1.2.5、总结



这一章节我们讲到的是生产者消费者中最关键的数据结构**队列**，包含`FIFO`和`DeltaFIFO`，其本质都是一个`slice`，两者在结构上稍微有些差别：

- `DeltaFIFO`比`FIFO`多了元素的操作类型，因此可以根据元素的增删改作出不同的处理
- `DeltaFIFO`会保存同一个元素的所有操作历史（相同操作会去重），FIFO只会保留元素最新的状态
- `DeltaFIFO`的`Pop`函数中传入的`PopProcessFunc`可以对元素的增删改作出不同的处理，FIFO则只能处理最新的

除了`FIFO`和`DeltaFIFO`以外，我们还讲到了这两个对象实现了的接口`Queue`，本质上`Queue`是一个实现了队列操作的`Store`。



> 了解完队列相关的数据结构，你是不是对后面可能要讲到的内容有一定的预期了呢？
>
> 1.1节讲到的cache用来做全量资源的缓存，1.2节讲到的队列用来处理增删改的事件，这些都是list-watch的生产者消费者框架中重要的数据结构，那么一定有其他的数据结构和算法来把他们组织起来，从之前了解到的list-watch的client端画像来看，这个client一定有段逻辑来维护cache这个全量缓存并通过队列来把生产者与消费者组织起来。
>
> 我们不妨大胆做一个猜测：可能是先到server端全量list一把，然后存到cache中，剩下的事情就是从server端不断watch新的变化并加入到队列中，这个队列的消费者那里除了将变化更新到cache中之外，还会对这个变化做出相应的处理。



## 1.3、关键数据结构`Informer`和`Reflector`





1.1和1.2章节讲到的都是生产者消费者模型中最基本的数据结构，接下来我们要讨论的是将生产者消费者模型组织起来的关键数据结构`informer`和`reflector`，这也是在k8s的client中直接使用的数据结构，其中会使用到我们前面讲到的基础数据结构。





> 这里有必要提前说明一下informer、reflector、Indexer这几者的区别：[参考资料](<https://www.huweihuang.com/kubernetes-notes/code-analysis/kube-controller-manager/sharedIndexInformer.html>)
>
> - `Reflector`：reflector用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的DeltaFIFO队列中。
> - `Informer`：informer从DeltaFIFO队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。
> - `Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合。
>
> 也就是说，reflector是真正的生产者，informer则是消费者，由此可以推测reflector中一定有队列，informer中一定有逻辑来调用这个队列的Pop函数来进行处理。





### 1.3.1、`Reflector`



我们先来看`Reflector`的结构体，之前已经说到`Reflector`是生产者，那么它的数据结构是什么样的呢？

```go
// Reflector仅监听一种k8s资源的变化并放置到存储器（队列）中
type Reflector struct {
	// reflector的名称，默认情况下是以调用NewReflector函数的代码file:line为名称
	name string
	// metrics tracks basic metric information about the reflector
	metrics *reflectorMetrics
	// 监听的资源类型，之所以这个结构体称为Reflector，主要就是能反射所有的资源类型，只要你定义好
    // 需要注意的是，这里的资源类型
	expectedType reflect.Type
	// 存储器（本质上是DeltaFIFO队列），也是生产者和消费者之间的桥梁
	store Store
	// 真正的生产者函数，其本质是调用kube-apiserver的接口来list-watch这个资源
	listerWatcher ListerWatcher
    // list-watch执行失败后重新list-watch的等待时间，默认为1秒
	period       time.Duration 
    // 在watch的过程中定期进行同步队列的周期（并非重新执行list操作的周期）
	resyncPeriod time.Duration 
    // 判断是否需要重新同步，watch的过程单独的协程中调用
	ShouldResync func() bool 
	// clock allows tests to manipulate time
	clock clock.Clock
	// 最后一次同步时从kube-apiserver获取到的该资源的版本号，跟etcd中可能不一致
    // 无论是初次list还是后续不断的watch过程都会更新这个版本号
	lastSyncResourceVersion string
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
	// 初次list时进行分页的页大小，主要用来提升list的性能
	WatchListPageSize int64
}
```

以上是`Reflector`的所有成员，从中不难看出这些成员之间的配合关系，在此简单梳理一下`Reflector`实际的执行逻辑（即`Reflector`的`Run()`函数逻辑，代码位于staging/src/k8s.io/client-go/tools/cache/reflector.go）：

- 首先list一把当前这个资源类型的所有数据，然后调用`DeltaFIFO`的`Replace()`将list到的新数据同步到队列中，并将当前的版本号记录到`lastSyncResourceVersion`中
- 起一个协程（收到stop信号或resync报错时退出），以`resyncPeriod`为周期，每次都通过`ShouldResync`函数判断（nil时默认执行）是否需要重新同步，是则将`DeltaFIFO`中的`knownObjects`（不为nil的情况下）遍历，并将`DeltaFIFO`队列中不存在的元素加入到队列中（`DeltaType`为`Sync`）
- 起一个死循环（收到stop信号时退出），watch对应类型的资源，一旦有新事件产生则调用`DeltaFIFO`的增删改接口加入到队列中去

> 值得注意的是，list只在watch之前执行一次，在watch的过程中并不会重新list，而只是会定期“整理”（`Resync()`，与`DeltaFIFO`中的`knownObjects`对比）队列中的元素。只有当watch报错时才会重新触发list-watch。

由此可以看出，这个过程中除了watch到了应该关注的增删改事件之外，还产生了很多`DeltaType`为`Sync`的事件，为什么需要产生这中类型的事件？这些事件是如何处理的呢？另外`Reflector`究竟是在被谁使用？我们且带着疑问继续往下看~



`Reflector`的`Run()`函数逻辑（可以跳过）

代码位于





#### 如何创建



```go
// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
	r := &Reflector{
		name:          name,
		listerWatcher: lw,
		store:         store,
		expectedType:  reflect.TypeOf(expectedType),
		period:        time.Second,
		resyncPeriod:  resyncPeriod,
		clock:         &clock.RealClock{},
	}
	return r
}
```



### 1.3.2、`Controller`

前面讲到了`Reflector`是生产者，`Informer`是消费者，但是中间需要有一个桥梁来将两者关联起来，而这个角色就是`Controller`来扮演的，它会将生产者和消费者都运行起来。

```go
// Controller is a generic controller framework.
type controller struct {
	config         Config // 主要是里面的配置文件
	reflector      *Reflector // controller中的Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}
// Controller中最重要的是Run函数
type Controller interface {
	Run(stopCh <-chan struct{})  // controller实际执行的函数
	HasSynced() bool // 队列中是否已经同步过了（没有sync或者第一批加进去的元素）
	LastSyncResourceVersion() string // 返回资源最近一次的版本号
}
// Config contains all the settings for a Controller.
type Config struct {
	// 这个controller中的队列，给reflector用的
	Queue
	// 给kube-apiserver发送list-watch请求的函数
	ListerWatcher
    // 队列中的元素Pop()后对其进行处理的函数，Pop(PopProcessFunc) (interface{}, error)
	Process ProcessFunc
	// 要处理的对象的类型（也就是说一个controller只能处理一种类型的资源）
	ObjectType runtime.Object
	// Reprocess everything at least this often.
	// Note that if it takes longer for you to clear the queue than this
	// period, you will end up processing items in the order determined
	// by FIFO.Replace(). Currently, this is random. If this is a
	// problem, we can change that replacement policy to append new
	// things to the end of the queue instead of replacing the entire
	// queue.
	FullResyncPeriod time.Duration
	// 其实就是传给reflector的ShouldResync函数
	ShouldResync ShouldResyncFunc
	// 是则每次处理失败将元素重新入队
	RetryOnError bool
}
```

`Controller`中的运行逻辑比较简单，这里我们直接用代码说明一下：

```go
// Run中创建一个reflector并起一个协程将其Run起来，然后不停执行processLoop用来Pop队列中的元素并处理
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
        // 如果收到了stopCh的信号，则将队列关闭
		<-stopCh
		c.config.Queue.Close()
	}()
    // 使用config中的参数先创建一个reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()
	// 新起一个协程来执行reflector的Run函数（通过调用kube-apiserver的接口将元素加入到队列中）
    // 实际是生产者
	wg.StartWithChannel(stopCh, r.Run)
    // 不断执行processLoop函数来对队列中的元素进行Pop和处理
    // 实际是消费者
	wait.Until(c.processLoop, time.Second, stopCh)
}
// processLoop是处理队列中元素的消费者
func (c *controller) processLoop() {
	for {
        // 将队列中的元素Pop出来并使用c.config.Process中的函数进行处理
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == FIFOClosedError {
				return
			}
			if c.config.RetryOnError {
				// 如果设置了RetryOnError则重新入队
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

可以看到从`reflector`到`controller`都是生产者这边对队列的入队处理以及消费者对出队元素的处理，并没有看到全量缓存的踪迹，其实这个是要留给`Informer`来做的，另外对队列中Pop出来的元素进行处理也是config中的Process函数的事情，这个处理函数也是`Informer`中定义的。



### 1.3.3、`Informer`

我们先看一下`Informer`的结构体

```go
type sharedIndexInformer struct {
	indexer    Indexer  // 本质上就是一个cache，是本地的全量数据缓存
	controller Controller  // 前文提到的controller，里面包含了Reflector
	processor             *sharedProcessor // 针对各个事件的处理函数，一个informer中可以有多个
	cacheMutationDetector CacheMutationDetector // 突变检查器，当发现前后数据突变时直接panic，默认情况下并未开启
	listerWatcher ListerWatcher  // 生产者，client发送http请求list-watch apiserver的client
	objectType    runtime.Object  // 处理的资源类型，注意一个informer也只能处理一种k8s资源类型
	// resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
	// shouldResync to check if any of our listeners need a resync.
	resyncCheckPeriod time.Duration
	// EventHandler中默认的同步周期，添加EventHandler时如果没有指定就使用这个
	defaultEventHandlerResyncPeriod time.Duration
	// clock主要是用来测试的
	clock clock.Clock
    // 用于标示这个informer是否已经开始或者已经停止，processor处理的时候会做一些调整
	started, stopped bool
	startedLock      sync.Mutex

	// blockDeltas gives a way to stop all event distribution so that a late event handler
	// can safely join the shared informer.
	blockDeltas sync.Mutex
}
```

interface





```go
// SharedInformer has a shared data cache and is capable of distributing notifications for changes
// to the cache to multiple listeners who registered via AddEventHandler. If you use this, there is
// one behavior change compared to a standard Informer.  When you receive a notification, the cache
// will be AT LEAST as fresh as the notification, but it MAY be more fresh.  You should NOT depend
// on the contents of the cache exactly matching the notification you've received in handler
// functions.  If there was a create, followed by a delete, the cache may NOT have your item.  This
// has advantages over the broadcaster since it allows us to share a common cache across many
// controllers. Extending the broadcaster would have required us keep duplicate caches for each
// watch.
type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	AddEventHandler(handler ResourceEventHandler)
	// AddEventHandlerWithResyncPeriod adds an event handler to the shared informer using the
	// specified resync period.  Events to a single handler are delivered sequentially, but there is
	// no coordination between different handlers.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore获取缓存接口（cache的Store接口）
	GetStore() Store
	// GetController gives back a synthetic interface that "votes" to start the informer
	GetController() Controller
	// Run starts the shared informer, which will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	// HasSynced returns true if the shared informer's store has synced.
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string
}

type SharedIndexInformer interface {
	SharedInformer
	// AddIndexers add indexers to the informer before it starts.
	AddIndexers(indexers Indexers) error
	GetIndexer() Indexer
}
```



```go

func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
    // 创建一个DeltaFIFO，以<namespace>/<name>作为索引的key，将indexer作为knownObjects进行同步
    // 注意到s.indexer到了DeltaFIFO那边之后只是作为全量的参考，用于DeltaFIFO做Replace/Resync
    // 实际s.indexer是informer自己在维护，DeltaFIFO只做ListKeys和GetByKey
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)
    // 创建controller中用到的Config
	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,
        // 这实际是传给DeltaFIFO的Pop()的处理函数，也就是实际处理队列元素的函数
        // HandleDeltas中维护缓存（Indexer）中的数据并把消息（实际也是这个元素）发送给处理函数
		Process: s.HandleDeltas,
	}
    // 那么问题来了，这里为什么要写成匿名函数呢？
	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
        // 通过前面生成的cfg创建informer中的controller
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// 这里主要是让主进程要结束的时候通知下面两个协程也退出并且等待这两个协程退出之后再退出主进程
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
    // 注意processor是通过AddEventHandler添加进来的，这里是执行这些Handler
    // 前面提到的s.HandleDeltas里面把要处理的元素放在chanel里面，processor执行的时候等待这个chanel
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
    // 让controller运行起来，这个Run函数中会创建reflector并起一个协程来进行list-watch，这个协程中会将watch到的变化加入到队列中，主进程中不停进行Pop操作并调用s.HandleDeltas对出队的元素进行处理
	s.controller.Run(stopCh)
}
```



### 1.3.4、总结













## 1.4、各个k8s资源的informer实现











# 2、关键运行流程





NewSharedInformerFactory









1. Informer 在初始化时，Reflector 会先 List API 获得所有的 Pod
2. Reflect 拿到全部 Pod 后，会将全部 Pod 放到 Store 中
3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据
4. Informer 初始化完成之后，Reflector 开始 Watch Pod，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件
5. Reflector 将 pod_1 被删除 的这个事件发送到 DeltaFIFO
6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1
7. DeltaFIFO 再 Pop 这个事件到 Controller 中
8. Controller 收到这个事件，会触发 Processor 的回调函数
9. LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中



![img](http://o6sfmikvw.bkt.clouddn.com/listwatch.png)

![img](https://res.cloudinary.com/dqxtn0ick/image/upload/v1555479782/article/code-analysis/informer/client-go-controller-interaction.jpg)





# 3、如何写一个自定义的controller







# 4、List-Watch中的设计理念







## 4.1、合理利用计算和存储资源，功夫下在每分每秒

`threadSafeMap`中的`indices`这个map的维护是深得这一理念的真传：

- **增删改**的时候维护好分类好的芸芸众生（功夫下在平时，多消耗一点计算和存储资源）
- **查**的时候直接按照分类返回同类（响应时间）

这个跟微信达人们使用微信分组是多么像啊：

- 平时管理好友时做好分组
- 发朋友圈时直接选择可见的分组



## 4.2、封装，接着封装





几个断言：

```go
func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
	return &threadSafeMap{
		items:    map[string]interface{}{},
		indexers: indexers,
		indices:  indices,
	}
}
```



```go
// 用于确认cache实现了Store的所有接口，否则编译会出错
var _ Store = &cache{}
```



```go
// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
	return &cache{
		cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
		keyFunc:      keyFunc,
	}
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		keyFunc:      keyFunc,
	}
}
```



```go
var (
	_ = Queue(&FIFO{}) // FIFO is a Queue
)
```



```go
var (
	_ = Queue(&DeltaFIFO{}) // DeltaFIFO is a Queue
)
```

