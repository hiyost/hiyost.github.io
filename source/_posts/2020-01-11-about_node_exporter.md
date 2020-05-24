---
title: 每周学习一个组件系列之node_exporter
date: 2020-01-11 10:42:14
tags: node_exporter
---

### 1、项目概况

项目地址：https://github.com/prometheus/node_exporter

`node_exporter`本质是`prometheus`项目衍生出来的[众多exporter](https://prometheus.io/docs/instrumenting/exporters/#exporters-and-integrations)中的一个，主要用于收集*NIX内核的节点上硬件和操作系统的各种数据，并暴露`/metrics`接口以供其他组件（如prometheus）采集。该组件收集的监控指标可以直接参考github代码库的`README.md`，在此就不多做赘述。

<!-- more -->

### 2、部署方法

#### 2.1、二进制部署

```shell
wget https://github.com/prometheus/node_exporter/releases/download/v0.14.0/node_exporter-0.14.0.linux-amd64.tar.gz
tar -xvzf node_exporter-0.14.0.linux-amd64.tar.gz
cd node_exporter-0.14.0.linux-amd64
./node_exporter
```

#### 2.2、通过容器部署

```shell
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  --net="host" \
  quay.io/prometheus/node-exporter \
    -collector.procfs /host/proc \
    -collector.sysfs /host/sys \
    -collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

#### 2.3、通过k8s部署

node-exporter.yaml文件内容如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    app: node-exporter
    name: node-exporter
  name: node-exporter
spec:
  clusterIP: None
  ports:
  - name: scrape
    port: 9100
    protocol: TCP
  selector:
    app: node-exporter
  type: ClusterIP
----
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  template:
    metadata:
      labels:
        app: node-exporter
      name: node-exporter
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/tryk8s/node-exporter:latest
        name: node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: scrape
      hostNetwork: true
      hostPID: true
```

然后通过kubectl创建deploy和对应的service即可

```shell
kubectl create -f node-exporter.yaml
```

### 3、启动参数说明

```shell
/bin/node_exporter -h
      --web.listen-address=":9100"  # 监听的端口，默认是9100
      --web.telemetry-path="/metrics"  # metrics的路径，默认为/metrics
      --web.disable-exporter-metrics  # 是否禁用go、prome默认的metrics
      --web.max-requests=40     # 最大并行请求数，默认40，设置为0时不限制
      --log.level="info"        # 日志等级: [debug, info, warn, error, fatal]
      --log.format="logger:stderr"  # 设置日志打印target和格式. 例子: "logger:syslog?appname=bob&local=7" or "logger:stdout?json=true"
      --version                 # 版本号
      --collector.{metric-name} # 各个metric对应的参数
```

### 4、代码分析

`node_exporter`的main函数在根目录下的`node_exporter.go`中，各个metrics参数的实现代码都存在于`./collector`目录中，下面我们从main函数开始逐一看一下这个组件是怎么运行的：

```go
func main() {
    // 前面是各种启动参数和日志的初始化处理，省略

    // /metrics路径注册到http server中，这里的newHandler是关键
	http.Handle(*metricsPath, newHandler(!*disableExporterMetrics, *maxRequests, logger))
    // /根路径注册到http server中，这里直接返回html
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`<html>
			<head><title>Node Exporter</title></head>
			<body>
			<h1>Node Exporter</h1>
			<p><a href="` + *metricsPath + `">Metrics</a></p>
			</body>
			</html>`))
	})

    // 启动这个http server
	level.Info(logger).Log("msg", "Listening on", "address", *listenAddress)
	server := &http.Server{Addr: *listenAddress}
	if err := https.Listen(server, *configFile); err != nil {
		level.Error(logger).Log("err", err)
		os.Exit(1)
	}
}
```

对于`node_exporter`而言，最重要的就是`/metrics`路径，那么这个路径的handler就是最为关键的实现了，上一段代码中的`newHandler()`函数返回了这个实现：

```go
// handler 中集合了所有的metrics参数的Handler，实际收到请求时只会使用启动参数中指定开启的metrics 
type handler struct {
    // unfilteredHandler就是Handler的集合，可见这是最重要的成员
	unfilteredHandler http.Handler
	// node_exporter本身的metrics
	exporterMetricsRegistry *prometheus.Registry
    // 是否暴露gc、进程相关的metrics，可在启动参数中设定
	includeExporterMetrics  bool
    // 限制的最大并行请求数
	maxRequests             int
	logger                  log.Logger
}

func newHandler(includeExporterMetrics bool, maxRequests int, logger log.Logger) *handler {
    // 初始化handler
	h := &handler{
		exporterMetricsRegistry: prometheus.NewRegistry(),
		includeExporterMetrics:  includeExporterMetrics,
		maxRequests:             maxRequests,
		logger:                  logger,
	}
	if h.includeExporterMetrics {
		h.exporterMetricsRegistry.MustRegister(
			prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{}),
			prometheus.NewGoCollector(),
		)
	}
    // 初始化Handler，将所有metric的Handler注册进来
	if innerHandler, err := h.innerHandler(); err != nil {
		panic(fmt.Sprintf("Couldn't create metrics handler: %s", err))
	} else {
		h.unfilteredHandler = innerHandler
	}
	return h
}

// ServeHTTP实现了http.Handler. http server起来之后调用的就是这个函数
// 这里主要根据请求中的collect过滤了一把
func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	filters := r.URL.Query()["collect[]"]
	level.Debug(h.logger).Log("msg", "collect query:", "filters", filters)

	if len(filters) == 0 {
		// No filters, use the prepared unfiltered handler.
		h.unfilteredHandler.ServeHTTP(w, r)
		return
	}
	// To serve filtered metrics, we create a filtering handler on the fly.
	filteredHandler, err := h.innerHandler(filters...)
	if err != nil {
		level.Warn(h.logger).Log("msg", "Couldn't create filtered metrics handler:", "err", err)
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte(fmt.Sprintf("Couldn't create filtered metrics handler: %s", err)))
		return
	}
	filteredHandler.ServeHTTP(w, r)
}
```

我们可以看到`node_exporter`中定义了一个`handler`的结构体类型，这个类型实现了http.Handler接口，`newHandler()`函数返回的就是这个类型。无论是在`newHandler`这个函数还是其实现的`ServeHTTP`中，均调用了`innerHandler`来初始化实际运行的handler。接下来我们看一下这个`innerHandler`：

```go
// innerHandler用来创建实现了promethues接口的handler
func (h *handler) innerHandler(filters ...string) (http.Handler, error) {
    // 关键步骤1：创建Collector集合，注意这是个工厂模式，也是各个metrics各自实现的关键，后文继续分析
	nc, err := collector.NewNodeCollector(h.logger, filters...)
	if err != nil {
		return nil, fmt.Errorf("couldn't create collector: %s", err)
	}
	// 打印创建日志，只会在初始化h.unfilteredHandler的执行，因为ServeHTTP函数中如果filters为空时不会进入到当前这个函数中来
	if len(filters) == 0 {
		level.Info(h.logger).Log("msg", "Enabled collectors")
		collectors := []string{}
		for n := range nc.Collectors {
			collectors = append(collectors, n)
		}
		sort.Strings(collectors)
		for _, c := range collectors {
			level.Info(h.logger).Log("collector", c)
		}
	}

	r := prometheus.NewRegistry()
	r.MustRegister(version.NewCollector("node_exporter"))
    // 关键步骤2：将前面所有的Collector注册到prometheus的Registry中，注意nc需要实现prometheus的Collector接口，即Describe(ch chan<- *prometheus.Desc)和Collect(ch chan<- prometheus.Metric)
	if err := r.Register(nc); err != nil {
		return nil, fmt.Errorf("couldn't register node collector: %s", err)
	}
    // 以下是prom接口的通用注册流程
	handler := promhttp.HandlerFor(
		prometheus.Gatherers{h.exporterMetricsRegistry, r},
		promhttp.HandlerOpts{
			ErrorHandling:       promhttp.ContinueOnError,
			MaxRequestsInFlight: h.maxRequests,
			Registry:            h.exporterMetricsRegistry,
		},
	)
	if h.includeExporterMetrics {
		// Note that we have to use h.exporterMetricsRegistry here to
		// use the same promhttp metrics for all expositions.
		handler = promhttp.InstrumentMetricHandler(
			h.exporterMetricsRegistry, handler,
		)
	}
	return handler, nil
}
```

通过上面的代码中也可以看出来，`innerHandler`中最关键的两个步骤就是先通过`NewNodeCollector`创建了一个`Collector`集合，然后使用prometheus库的`Registry`将该`Collector`注册了进去。好，这里我们注意到了两个细节：

- `NewNodeCollector`创建了一个`Collector`集合（实际是node_exporter中自己定义的`NodeCollector`类型）想要注册到`Registry`中，就必须实现prometheus的`Collector`接口
- 请求中的filters不为空时，每个请求过来都会在`innerHandler`函数中创建`Collector`，面对请求中filters不为空占大多数的情况是效率比较低的，这里似乎可以继续优化

前面已经说到了两个关键步骤，接下来我们就一起看一下这两个关键步骤分别做了什么：

1、`NewNodeCollector`创建了一个`Collector`集合

在node_exporter的`./collector`目录中，定义了一个`NodeCollector`的结构体类型，`NewNodeCollector`返回的就是这个类型的实例，当然这个类型实现了prometheus的`Collector`接口

```go
// NodeCollector implements the prometheus.Collector interface.
type NodeCollector struct {
    // 以名字为索引的Collector表，仅保存需要的Collector
	Collectors map[string]Collector
	logger     log.Logger
}

// NewNodeCollector creates a new NodeCollector.
func NewNodeCollector(logger log.Logger, filters ...string) (*NodeCollector, error) {
	f := make(map[string]bool)
	for _, filter := range filters {
		enabled, exist := collectorState[filter]
		if !exist {
			return nil, fmt.Errorf("missing collector: %s", filter)
		}
		if !*enabled {
			return nil, fmt.Errorf("disabled collector: %s", filter)
		}
		f[filter] = true
	}
    // 根据filters和默认开启情况来将各个Collector集合到NodeCollector中
	collectors := make(map[string]Collector)
	for key, enabled := range collectorState {
		if *enabled {
            // 注意到这里由一个factories，如果要将实现的某一个Collector使用起来，就需要将其注册到这个factories中
			collector, err := factories[key](log.With(logger, "collector", key))
			if err != nil {
				return nil, err
			}
			if len(f) == 0 || f[key] {
				collectors[key] = collector
			}
		}
	}
	return &NodeCollector{Collectors: collectors, logger: logger}, nil
}

// Describe implements the prometheus.Collector interface.
func (n NodeCollector) Describe(ch chan<- *prometheus.Desc) {
	ch <- scrapeDurationDesc
	ch <- scrapeSuccessDesc
}

// Collect implements the prometheus.Collector interface.
// Collect是http server收到请求之后实际执行的函数，这里面其实就是把所有注册的Collector都执行一遍
func (n NodeCollector) Collect(ch chan<- prometheus.Metric) {
	wg := sync.WaitGroup{}
	wg.Add(len(n.Collectors))
	for name, c := range n.Collectors {
		go func(name string, c Collector) {
			execute(name, c, ch, n.logger)
			wg.Done()
		}(name, c)
	}
	wg.Wait()
}

func execute(name string, c Collector, ch chan<- prometheus.Metric, logger log.Logger) {
	begin := time.Now()
    // 执行这个Collector的Update函数，将从节点上获取的数据写入到channel中去
	err := c.Update(ch)
	// 中间部分略去
	ch <- prometheus.MustNewConstMetric(scrapeDurationDesc, prometheus.GaugeValue, duration.Seconds(), name)
	ch <- prometheus.MustNewConstMetric(scrapeSuccessDesc, prometheus.GaugeValue, success, name)
}
```

NodeCollector结构体类型中最重要的成员是`Collector`，这是抽象出来的一个interface类型。

```go
// Collector is the interface a collector has to implement.
type Collector interface {
	// Get new metrics and expose them via prometheus registry.
	Update(ch chan<- prometheus.Metric) error
}
```

由此也可以看得比较清楚了，对于每一个指标而言，只需要实现`Collector`这个interface就可以了，`NodeCollector`在其`Collect`函数中会对其进行调用。同时在collector包中写好了`registerCollector`函数，每一个指标只需要实现`Collector`然后通过这个函数将其注册到`factories`中即可。

```go
const (
	defaultEnabled  = true
	defaultDisabled = false
)

var (
    // Collector工厂，通过registerCollector来注册
	factories      = make(map[string]func(logger log.Logger) (Collector, error))
    // 
	collectorState = make(map[string]*bool)
)

func registerCollector(collector string, isDefaultEnabled bool, factory func(logger log.Logger) (Collector, error)) {
	var helpDefaultState string
	if isDefaultEnabled {
		helpDefaultState = "enabled"
	} else {
		helpDefaultState = "disabled"
	}

	flagName := fmt.Sprintf("collector.%s", collector)
	flagHelp := fmt.Sprintf("Enable the %s collector (default: %s).", collector, helpDefaultState)
	defaultValue := fmt.Sprintf("%v", isDefaultEnabled)

	flag := kingpin.Flag(flagName, flagHelp).Default(defaultValue).Bool()
	collectorState[collector] = flag

	factories[collector] = factory
}
```

这里可以举一个conntrack的栗子

```go
type conntrackCollector struct {
	current *prometheus.Desc
	limit   *prometheus.Desc
	logger  log.Logger
}
// 初始化时将conntrack这个Collector注册到工厂中，并且默认是开启的，注意所有的监控指标都是在初始化的时候直接指定默认是否开启
func init() {
	registerCollector("conntrack", defaultEnabled, NewConntrackCollector)
}

// NewConntrackCollector returns a new Collector exposing conntrack stats.
func NewConntrackCollector(logger log.Logger) (Collector, error) {
	return &conntrackCollector{
		current: prometheus.NewDesc(
			prometheus.BuildFQName(namespace, "", "nf_conntrack_entries"),
			"Number of currently allocated flow entries for connection tracking.",
			nil, nil,
		),
		limit: prometheus.NewDesc(
			prometheus.BuildFQName(namespace, "", "nf_conntrack_entries_limit"),
			"Maximum size of connection tracking table.",
			nil, nil,
		),
		logger: logger,
	}, nil
}
// Update实现了Collector接口，从操作系统中读取conntrack的当前值然后发送到channel中
func (c *conntrackCollector) Update(ch chan<- prometheus.Metric) error {
	value, err := readUintFromFile(procFilePath("sys/net/netfilter/nf_conntrack_count"))
	if err != nil {
		// Conntrack probably not loaded into the kernel.
		return nil
	}
	ch <- prometheus.MustNewConstMetric(
		c.current, prometheus.GaugeValue, float64(value))

	value, err = readUintFromFile(procFilePath("sys/net/netfilter/nf_conntrack_max"))
	if err != nil {
		return nil
	}
	ch <- prometheus.MustNewConstMetric(
		c.limit, prometheus.GaugeValue, float64(value))

	return nil
}
```

