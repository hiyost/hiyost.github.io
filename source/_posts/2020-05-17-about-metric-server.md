---
title: 每周学习一个组件系列之metric-server
date: 2020-05-17 15:15:51
tags: metric-server
---

### 1、项目概况

项目地址：https://github.com/kubernetes-sigs/metrics-server

在`k8s`集群中，如果你想要去做弹性伸缩，或者想要使用`kubectl top`命令，那么`metric-server`是你绕不开的组件。`metric-server`主要用来通过`aggregate api`向其它组件提供集群中的`pod`和`node`的`cpu`和`memory`的监控指标，弹性伸缩中的`podautoscaler`就是通过调用这个接口来查看pod的当前资源使用量来进行pod的扩缩容的。

需要注意的是：

- `metric-server`提供的是实时的指标（实际是最近一次采集的数据，保存在内存中），并没有数据库来存储
- 这些数据指标并非由`metric-server`本身采集，而是由每个节点上的`cadvisor`采集，`metric-server`只是发请求给`cadvisor`并将`metric`格式的数据转换成`aggregate api`
- 由于需要通过`aggregate api`来提供接口，需要集群中的`kube-apiserver`开启该功能（开启方法可以参考官方社区的文档）

<!-- more -->

### 2、部署方法

`metric-server`最佳的安装方法是通过`deployment`：

```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

该`yaml`中主要的`deployment`参数如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
        ports:
        - name: main-port
          containerPort: 4443
          protocol: TCP
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: "amd64"
```

其中还有一个值得注意的资源是一个`APIService`，这个资源主要就是将`metrics-server`注册到`aggregate api`中。

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

### 3、启动参数

| 参数名称                               | 参数解释                                            |
| -------------------------------------- | --------------------------------------------------- |
| metric-resolution                      | 周期性调用接口获取metric原始数据的时间间隔，默认60s |
| kubelet-insecure-tls                   | 访问kubelet时不对其证书进行ca校验，仅测试时使用     |
| kubelet-port                           | 调用节点上的kubelet获取metric的端口，默认10250端口  |
| kubeconfig                             | 调用kube-apiserver和kubelet使用的kubeconfig文件路径 |
| kubelet-preferred-address-types        | 调用kubelet使用的ip地址优先级                       |
| kubelet-certificate-authority          | 访问kubelet使用的ca证书                             |
| deprecated-kubelet-completely-insecure | 使用非安全方式访问kubelet（即将废弃）               |

### 4、代码分析

在开始走读`metrics-server`的代码之前，我们先来根据其功能来猜测一下它的代码逻辑。我们知道，通过节点上的`cadvisor`接口获取到的数据一般是这样的，包含的信息太多：

```
[root@node1 ~]# curl -k https://172.17.8.101:10250/stats/summary?only_cpu_and_memory=true
{
  "node": {
    "nodeName": "node1",
    "systemContainers": [
      {
        "name": "kubelet",
        "startTime": "2020-05-24T12:54:13Z",
        "cpu": {
          "time": "2020-05-24T14:12:31Z",
          "usageNanoCores": 20686133,
          "usageCoreNanoSeconds": 156089526198
        },
        "memory": {
          "time": "2020-05-24T14:12:31Z",
          "usageBytes": 170590208,
          "workingSetBytes": 122531840,
          "rssBytes": 66949120,
          "pageFaults": 763727,
          "majorPageFaults": 85
        },
        "userDefinedMetrics": null
      },
      {
        "name": "runtime",
        "startTime": "2020-05-24T12:54:13Z",
        "cpu": {
          "time": "2020-05-24T14:12:31Z",
          "usageNanoCores": 20686133,
          "usageCoreNanoSeconds": 156089526198
        },
        "memory": {
          "time": "2020-05-24T14:12:31Z",
          "usageBytes": 170590208,
          "workingSetBytes": 122531840,
          "rssBytes": 66949120,
          "pageFaults": 763727,
          "majorPageFaults": 85
        },
        "userDefinedMetrics": null
      },
      {
        "name": "pods",
        "startTime": "2020-05-24T12:54:13Z",
        "cpu": {
          "time": "2020-05-24T14:12:39Z",
          "usageNanoCores": 0,
          "usageCoreNanoSeconds": 42207538504
        },
        "memory": {
          "time": "2020-05-24T14:12:39Z",
          "availableBytes": 1910824960,
          "usageBytes": 33480704,
          "workingSetBytes": 16498688,
          "rssBytes": 36864,
          "pageFaults": 0,
          "majorPageFaults": 0
        },
        "userDefinedMetrics": null
      }
    ],
    "startTime": "2020-05-24T12:52:24Z",
    "cpu": {
      "time": "2020-05-24T14:12:39Z",
      "usageNanoCores": 888521168,
      "usageCoreNanoSeconds": 776524490477
    },
    "memory": {
      "time": "2020-05-24T14:12:39Z",
      "availableBytes": 891166720,
      "usageBytes": 1627074560,
      "workingSetBytes": 1036156928,
      "rssBytes": 359944192,
      "pageFaults": 1850284,
      "majorPageFaults": 1987
    }
  },
  "pods": [
    {
      "podRef": {
        "name": "metrics-server-7668599459-2jxq5",
        "namespace": "kube-system",
        "uid": "f5af876f-03de-43e5-902b-79bece68c508"
      },
      "startTime": "2020-05-24T13:27:42Z",
      "containers": null,
      "cpu": {
        "time": "2020-05-24T14:12:36Z",
        "usageNanoCores": 0,
        "usageCoreNanoSeconds": 6297886
      },
      "memory": {
        "time": "2020-05-24T14:12:36Z",
        "usageBytes": 434176,
        "workingSetBytes": 249856,
        "rssBytes": 36864,
        "pageFaults": 0,
        "majorPageFaults": 0
      }
    }
  ]
}
```

而我们通过`metrics-server`获得的数据则是这样：

```
[root@node1 ~]# curl http://172.17.8.101:8080/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/metrics-server-7668599459-2jxq5
{
  "kind": "PodMetrics",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "name": "metrics-server-7668599459-2jxq5",
    "namespace": "kube-system",
    "selfLink": "/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/metrics-server-7668599459-2jxq5",
    "creationTimestamp": "2020-05-24T13:27:42Z"
  },
  "timeStamp": "2020-05-24T13:27:42Z",
  "window": "30s",
  "containers": [
    {
      "name": "metrics-server",
      "usage": {
        "cpu": "0",
        "memory": "424Ki"
      }
    }
  ]
}
```

也就是说，本质上`metrics-server`相当于做了一次数据的转换，把`cadvisor`格式的数据转换成了`k8s`的`api`的`json`格式。由此我们不难猜测，`metrics-server`的代码中必然存在这种先从metric中获取接口中的所有信息，再解析出其中的数据的过程。除此之外，我们可能也会有一个疑惑，那就是：我们给`metric-server`发送请求时，`metric-server`是马上向`cadvisor`发送请求然后解析请求中的数据再返回回来，还是`metrics-server`中已经定期从中`cadvisor`获取好数据了（可能缓存在内存中），当请求发过来时直接返回缓存中的数据。我们可以带着这个疑问直接去看源码。

#### 4.1、启动程序

`metric-server`的启动流程使用的也是`github.com/spf13/cobra`框架，对这个库感兴趣的可以去[`github`](<https://github.com/spf13/cobra>)上了解一下，该框架实际执行的是`MetricsServerOptions`实现的Run函数

```go
// cmd/metrics-server/app/start.go
func (o MetricsServerOptions) Run(stopCh <-chan struct{}) error {
	// 1、生成metric-server自己的server端配置
	config, err := o.Config()
	if err != nil {
		return err
	}
	config.GenericConfig.EnableMetrics = true

	// 2、生成metric-server自己的client端配置
    // 包含对kube-apiserver的client（获取集群node信息）和对cadvisor的client（获取原始监控数据）
	var clientConfig *rest.Config
	if len(o.Kubeconfig) > 0 {
		loadingRules := &clientcmd.ClientConfigLoadingRules{ExplicitPath: o.Kubeconfig}
		loader := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(loadingRules, &clientcmd.ConfigOverrides{})
		clientConfig, err = loader.ClientConfig()
	} else {
		clientConfig, err = rest.InClusterConfig()
	}
	if err != nil {
		return fmt.Errorf("unable to construct lister client config: %v", err)
	}
	// Use protobufs for communication with apiserver
	clientConfig.ContentType = "application/vnd.kubernetes.protobuf"

	// 2.1、通过刚才的client配置参数创建kube-apiserver的client
	kubeClient, err := kubernetes.NewForConfig(clientConfig)
	if err != nil {
		return fmt.Errorf("unable to construct lister client: %v", err)
	}
	// 根据client创建对应的informer，到这里与kuber-apiserver通信的部分就设置好了
	informerFactory := informers.NewSharedInformerFactory(kubeClient, 0)

	// 2.2、这里开始创建与节点上的metric接口相关的client
	kubeletRestCfg := rest.CopyConfig(clientConfig)
	if len(o.KubeletCAFile) > 0 {
		kubeletRestCfg.TLSClientConfig.CAFile = o.KubeletCAFile
		kubeletRestCfg.TLSClientConfig.CAData = nil
	}
	kubeletConfig := summary.GetKubeletConfig(kubeletRestCfg, o.KubeletPort, o.InsecureKubeletTLS, o.DeprecatedCompletelyInsecureKubelet)
	kubeletClient, err := summary.KubeletClientFor(kubeletConfig)
	if err != nil {
		return fmt.Errorf("unable to construct a client to connect to the kubelets: %v", err)
	}

	// 设置访问node的ip的优先级（node中保存有各种address，包括InternalIP、ExternalIP等）
	addrPriority := make([]corev1.NodeAddressType, len(o.KubeletPreferredAddressTypes))
	for i, addrType := range o.KubeletPreferredAddressTypes {
		addrPriority[i] = corev1.NodeAddressType(addrType)
	}
	addrResolver := summary.NewPriorityNodeAddressResolver(addrPriority)
    // sourceProvider是将前面的两个client合并到一起从cadvisor抓取数据
    // informer负责获取集群中的节点相关的信息，kubeletClient则调用这些节点上的cadvisor接口
    // 注意这里只传入了NodeLister，也就是是说只需要list node相关的信息就可以了
	sourceProvider := summary.NewSummaryProvider(informerFactory.Core().V1().Nodes().Lister(), kubeletClient, addrResolver)
	scrapeTimeout := time.Duration(float64(o.MetricResolution) * 0.90) // scrape timeout is 90% of the scrape interval
    // 将抓取时间间隔放到本server的metric接口中，并创建一个sourceManager
	sources.RegisterDurationMetrics(scrapeTimeout)
	sourceManager := sources.NewSourceManager(sourceProvider, scrapeTimeout)

	// 3、创建metricSink用来保存获取并解析出来的监控数据（仅保存在内存中）
    //    需要注意的是，这里的metricSink和metricsProvider是同一个sinkMetricsProvider实例
	metricSink, metricsProvider := sink.NewSinkProvider()

	// 4、创建一个Manager用来将前面的sourceProvider和metricSink管理起来，前者抓数据，后者存数据
	manager.RegisterDurationMetrics(o.MetricResolution)
	mgr := manager.NewManager(sourceManager, metricSink, o.MetricResolution)

	// 1.1、将刚才的metricSink传入到server的配置中去，这样http server直接从metricSink中获取数据，然后直接返回给client就可以了，不需要再去调用cadvisor查metric数据
	config.ProviderConfig.Node = metricsProvider
	config.ProviderConfig.Pod = metricsProvider

	// 1.2、通过config给将要启动的server做一些初始化的动作，同时也将informerFactory传进去
	server, err := config.Complete(informerFactory).New()
	if err != nil {
		return err
	}

	// add health checks
	server.AddHealthzChecks(healthz.NamedCheck("healthz", mgr.CheckHealth))

	// 5、将刚才的manager运行起来（调用cadvisor的接口获取并解析数据，然后存到metricSink中）
	mgr.RunUntil(stopCh)
    // 1.3、根据1中的配置将metric-server启动起来
	return server.GenericAPIServer.PrepareRun().Run(stopCh)
}
```

从这段代码中可以看出来，数据的抓取和缓存与`server`是两个不同的处理流程，他们之间通过共享内存来配合，数据定期抓取完之后缓存到`metricSink`（其实也就是`metricsProvider`）中，而`server`收到请求时从`metricSink`中读取数据并返回给`client`。这个过程也正好回答了我们之前的问题，`metrics-server`中已经定期从中`cadvisor`获取好数据了，当请求发过来时直接返回缓存中的数据。

#### 4.2、数据抓取与缓存

4.1章节代码注释中的2/3/4/5小节就是`metric-server`的数据抓取流程的启动过程，我们暂且称之为`manager`，我们看到这其中主要是起了两个`client`，一个是`kube-apiserver`的`client`用来获取集群中`node`资源，另一个client则是调用节点上`cadvisor`的接口获取节点和`pod`的`cpu`和`memory`监控数据，同时也创建了一个`metricSink`用来保存获取的监控数据。话不多说，我们来看一下这个`manager`是如何运转的。

我们将4.1章节代码注释中的4和5连在一起就是`manager`的启动过程，创建一个`manager`然后运行起来。

```go
	manager.RegisterDurationMetrics(o.MetricResolution)
	mgr := manager.NewManager(sourceManager, metricSink, o.MetricResolution)

	mgr.RunUntil(stopCh)
```

上面的启动过程实际如下，创建好的`manager`运行是其实就是周期性地执行其`Collect`函数。

```go
// pkg/manager/manager.go
func NewManager(metricSrc sources.MetricSource, metricSink sink.MetricSink, resolution time.Duration) *Manager {
	manager := Manager{
		source:     metricSrc, // 抓取metric的interface，需要实现Collect
		sink:       metricSink, // 保存抓取数据的接收器，需要实现Receive
		resolution: resolution, // 抓取metric的时间间隔
	}

	return &manager
}

func (rm *Manager) RunUntil(stopCh <-chan struct{}) {
	go func() {
        // 创建一个周期性的定时器
		ticker := time.NewTicker(rm.resolution)
		defer ticker.Stop()
		rm.Collect(time.Now())

		for {
			select {
            // 周期性执行Collect
			case startTime := <-ticker.C:
				rm.Collect(startTime) // 实际周期性执行的是这里的Collect函数
			case <-stopCh:
				return
			}
		}
	}()
}
```

接下来看`Collect`函数中的逻辑就很简介明了了，主要做了两件事件：

- 调用sourceManager实现的Collect函数获取metric数据
- 将获取到的原始metric数据解析成pod和node的数值并保存metricSink中去

```go
func (rm *Manager) Collect(startTime time.Time) {
	rm.healthMu.Lock()
	rm.lastTickStart = startTime
	rm.healthMu.Unlock()

	healthyTick := true

    // 给发request的context中设置超时时间为manager的检查周期
	ctx, cancelTimeout := context.WithTimeout(context.Background(), rm.resolution)
	defer cancelTimeout()

	klog.V(6).Infof("Beginning cycle, collecting metrics...")
    // 1、执行sourceManager实现的Collect函数获取metric数据
	data, collectErr := rm.source.Collect(ctx)
    
    // 省略异常处理的逻辑

	klog.V(6).Infof("...Storing metrics...")
    // 2、将获取到的原始metric数据保存到metricSink中去
	recvErr := rm.sink.Receive(data)
    
    // 省略异常处理的逻辑

    // 将实际的collect处理时间放到自己的metric接口中
	collectTime := time.Since(startTime)
	tickDuration.Observe(float64(collectTime) / float64(time.Second))
	klog.V(6).Infof("...Cycle complete")

	rm.healthMu.Lock()
	rm.lastOk = healthyTick
	rm.healthMu.Unlock()
}
```

##### 4.2.1、获取`metric`数据（`rm.source.Collect(ctx)`）

获取`metric`数据本质上就是调接口获取第4章节开头说的`/metric`格式的数据，而这个接口本质上就是k8s集群中节点上的`cadvisor`（实际由`kubelet`暴露），因此这部分的逻辑就是围绕这个思路展开。

- 首先需要知道这个集群中有哪些节点，并获取这些节点上获取`metric`的`ip`和端口
- 分别调用这些节点上的`metric`接口并解析其中`node`和`pod`的`cpu`和`memory`数值

```go
// pkg/sources/manager.go
func (m *sourceManager) Collect(baseCtx context.Context) (*MetricsBatch, error) {
    // 1、获取需要抓取数据的所有源头，即集群中节点上的cadvisor接口
	sources, err := m.srcProv.GetMetricSources()
	var errs []error
	if err != nil {
		errs = append(errs, err)
	}
	klog.V(1).Infof("Scraping metrics from %v sources", len(sources))
    // 创建接受数据和错误的channel
	responseChannel := make(chan *MetricsBatch, len(sources))
	errChannel := make(chan error, len(sources))
	defer close(responseChannel)
	defer close(errChannel)
	startTime := time.Now()
	delayMs := delayPerSourceMs * len(sources)
	if delayMs > maxDelayMs {
		delayMs = maxDelayMs
	}
	for _, source := range sources {
        // 2、分别起一个协程去调每个source的接口抓取数据，并写入到channel中
		go func(source MetricSource) {
            // 每个协程中随机sleep一段时间，防止几个协程同时发请求造成网络拥塞
			sleepDuration := time.Duration(rand.Intn(delayMs)) * time.Millisecond
			time.Sleep(sleepDuration)
			// 超时时间减去刚才sleep的时间
			ctx, cancelTimeout := context.WithTimeout(baseCtx, m.scrapeTimeout-sleepDuration)
			defer cancelTimeout()
			klog.V(2).Infof("Querying source: %s", source)
            // 抓取数据
			metrics, err := scrapeWithMetrics(ctx, source)
			if err != nil {
				err = fmt.Errorf("unable to fully scrape metrics from source %s: %v", source.Name(), err)
			}
			responseChannel <- metrics
			errChannel <- err
		}(source)
	}
	res := &MetricsBatch{}
	for range sources {
        // 将抓取的数据分成node和pod的保存下来并返回
		err := <-errChannel
		srcBatch := <-responseChannel
		if err != nil {
			errs = append(errs, err)
		}
		if srcBatch == nil {
			continue
		}

		res.Nodes = append(res.Nodes, srcBatch.Nodes...)
		res.Pods = append(res.Pods, srcBatch.Pods...)
	}

	klog.V(1).Infof("ScrapeMetrics: time: %s, nodes: %v, pods: %v", time.Since(startTime), len(res.Nodes), len(res.Pods))
	return res, utilerrors.NewAggregate(errs)
}
```

可以看到上述两个关键点的细节都封装起来了，我们一个一个来看：

1. ###### 获取源头`m.srcProv.GetMetricSources()`
```go
// pkg/sources/summary/summary.go
func (p *summaryProvider) GetMetricSources() ([]sources.MetricSource, error) {
	sources := []sources.MetricSource{}
    // 调用k8s接口List所有节点的信息
	nodes, err := p.nodeLister.List(labels.Everything())
	if err != nil {
		return nil, fmt.Errorf("unable to list nodes: %v", err)
	}

	var errs []error
	for _, node := range nodes {
        // 从节点的结构体中获取这个节点的IP和name，并和kubeletClient一起组装到source中去
        // 注意节点的IP时根据启动参数kubelet-preferred-address-types优先级来获取的
		info, err := p.getNodeInfo(node)
		if err != nil {
			errs = append(errs, fmt.Errorf("unable to extract connection information for node %q: %v", node.Name, err))
			continue
		}
        // 注意所有source的kubeletClient是共用的，端口都是一样，区别是IP不同
		sources = append(sources, NewSummaryMetricsSource(info, p.kubeletClient))
	}
	return sources, utilerrors.NewAggregate(errs)
}

func NewSummaryMetricsSource(node NodeInfo, client KubeletInterface) sources.MetricSource {
	return &summaryMetricsSource{
		node:          node,
		kubeletClient: client,
	}
}
```
2. ###### 抓取数据并解析数据`scrapeWithMetrics(ctx, source)`

```go
// pkg/sources/manager.go
func scrapeWithMetrics(ctx context.Context, s MetricSource) (*MetricsBatch, error) {
	sourceName := s.Name()
	startTime := time.Now()
	defer lastScrapeTimestamp.
		WithLabelValues(sourceName).
		Set(float64(time.Now().Unix()))
	defer scraperDuration.
		WithLabelValues(sourceName).
		Observe(float64(time.Since(startTime)) / float64(time.Second))
    // 实际调用MetricSource的Collect接口
	return s.Collect(ctx)
}
```

这里的`s.Collect(ctx)`实际上就是步骤1中`NewSummaryMetricsSource`创建出来的`MetricSource`的`Collect`

```go
// pkg/sources/summary/summary.go
func (src *summaryMetricsSource) Collect(ctx context.Context) (*sources.MetricsBatch, error) {
    // 关键逻辑，通过当前source的kubeletClient获取节点上的metric数据，并解析到summary中
    // 这里之所以把这段逻辑协程闭包是为了在GetSummary执行完之后就马上执行defer中的逻辑
	summary, err := func() (*stats.Summary, error) {
		startTime := time.Now()
		defer summaryRequestLatency.WithLabelValues(src.node.Name).Observe(float64(time.Since(startTime)) / float64(time.Second))  // 马上将执行时间写入自己的metric中
		return src.kubeletClient.GetSummary(ctx, src.node.ConnectAddress)
	}()
    // 部分省略
	res := &sources.MetricsBatch{
		Nodes: make([]sources.NodeMetricsPoint, 1),
		Pods:  make([]sources.PodMetricsPoint, len(summary.Pods)),
	}
    
    // 省略部分解析summary中node和pod数据的逻辑

	return res, utilerrors.NewAggregate(errs)
}
```

`Collect`的逻辑本质上是执行了`GetSummary`，这里起了一个`http client`，然后给`https://{node ip}:{port}/stats/summary?only_cpu_and_memory=true`发了一个`GET`请求并获取了返回的`body`体。

```go
// pkg/sources/summary/client.go
func (kc *kubeletClient) GetSummary(ctx context.Context, host string) (*stats.Summary, error) {
	scheme := "https"
	if kc.deprecatedNoTLS {
		scheme = "http"
	}
	url := url.URL{
		Scheme:   scheme,
		Host:     net.JoinHostPort(host, strconv.Itoa(kc.port)),
		Path:     "/stats/summary",
		RawQuery: "only_cpu_and_memory=true",
	}

	req, err := http.NewRequest("GET", url.String(), nil)
	if err != nil {
		return nil, err
	}
	summary := &stats.Summary{}
	client := kc.client
	if client == nil {
		client = http.DefaultClient
	}
    // 执行req并将response中的body体保存到summary中
	err = kc.makeRequestAndGetValue(client, req.WithContext(ctx), summary)
	return summary, err
}
```



3. ###### 总结

通过4.2.1中的代码分析我们印证之前的过程，获取`metric`数据本质上就是调用`cadvisor`的接口来获取数据而已。这里我们有必要来看一下之前我们忽略掉的关键数据结构`MetricsBatch`，这是最终解析出来的包含了`node`和`pod`的`cpu`和`memory`信息的对象，也是传递给`metricSink`的数据。

```go
// pkg/sources/interfaces.go
// MetricsBatch是所有node和所有pod的metric数据
type MetricsBatch struct {
	Nodes []NodeMetricsPoint
	Pods  []PodMetricsPoint
}
// NodeMetricsPoint包含这个节点的metric数据
type NodeMetricsPoint struct {
	Name string
	MetricsPoint
}
// PodMetricsPoint包含了这个pod所有容器的metric数据
type PodMetricsPoint struct {
	Name      string
	Namespace string
	Containers []ContainerMetricsPoint
}
// ContainerMetricsPoint包含这个容器的metric数据
type ContainerMetricsPoint struct {
	Name string
	MetricsPoint
}
// MetricsPoint是node和容器metric数据的基本单位，其中包含了cpu和memory度量
type MetricsPoint struct {
	Timestamp time.Time
	// CpuUsage is the CPU usage rate, in cores
	CpuUsage resource.Quantity
	// MemoryUsage is the working set size, in bytes.
	MemoryUsage resource.Quantity
}
```


##### 4.2.2、保存`metric`数据（`rm.sink.Receive(data)`）

4.2.1中已经调用集群中所有节点的`cadvisor`接口并获取了所有节点和`pod`的`metric`数据，接下来就是保存到缓存中了。前面已经将获取到的数据存在了`data`（本质上就是4.2.1总结时说到的`MetricsBatch`）变量中，接下来就时对这个变量进行处理了。

```go
// pkg/provider/sink/sinkprov.go
// sinkMetricsProvider既实现了provider.MetricsProvider（提供server时使用）也实现了sink.MetricSink（抓取后缓存数据用）
// sinkMetricsProvider本质上就是两个map，一个保存node，一个保存pod
type sinkMetricsProvider struct {
	mu    sync.RWMutex
	nodes map[string]sources.NodeMetricsPoint
	pods  map[apitypes.NamespacedName]sources.PodMetricsPoint
}
// metric-server收到node的GET请求调用这个函数来获取metric数据
func (p *sinkMetricsProvider) GetNodeMetrics(nodes ...string) ([]provider.TimeInfo, []corev1.ResourceList, error) {
    // 代码逻辑略
}
// metric-server收到pod的GET请求调用这个函数来获取metric数据
func (p *sinkMetricsProvider) GetContainerMetrics(pods ...apitypes.NamespacedName) ([]provider.TimeInfo, [][]metrics.ContainerMetrics, error) {
    // 代码逻辑略，4.3节的metric-server再讲
}

func (p *sinkMetricsProvider) Receive(batch *sources.MetricsBatch) error {
    // 创建一个新的node的map，并将数据去重写入
	newNodes := make(map[string]sources.NodeMetricsPoint, len(batch.Nodes))
	for _, nodePoint := range batch.Nodes {
		if _, exists := newNodes[nodePoint.Name]; exists {
			klog.Errorf("duplicate node %s received", nodePoint.Name)
			continue
		}
		newNodes[nodePoint.Name] = nodePoint
	}
    // 创建一个新的pod的map，并将数据去重写入
	newPods := make(map[apitypes.NamespacedName]sources.PodMetricsPoint, len(batch.Pods))
	for _, podPoint := range batch.Pods {
		podIdent := apitypes.NamespacedName{Name: podPoint.Name, Namespace: podPoint.Namespace}
		if _, exists := newPods[podIdent]; exists {
			klog.Errorf("duplicate pod %s received", podIdent)
			continue
		}
		newPods[podIdent] = podPoint
	}

	p.mu.Lock()
	defer p.mu.Unlock()
    // 将刚创建的新map赋值给provider，注意之前的旧map就直接回收了
	p.nodes = newNodes
	p.pods = newPods

	return nil
}
```

可以看到`sinkMetricsProvider`数据的结构体本质上就是两个`map`而已，而保存的逻辑也非常简单，直接创建两个新map并赋值过去就可以，并不需要处理之前的旧数据，简单粗暴。

#### 4.3、`metric-server`

在4.1节的启动程序中已经说明了`metric-server`的启动过程，经过4.2.2的代码分析之后，我们可以猜测`metric-server`本质上就是收到请求之后到`sinkMetricsProvider`的两个`map`中读取数据并返回而已。

我们回到启动程序中，这里包含了两个步骤，一个是创建好一个`server`（本质上是`k8s.io/apiserver`库的一个`GenericAPIServer`），另一个则是直接将这个`server`运行起来。

```go
	// 关键逻辑：通过配置文件创建好对应的GenericAPIServer
	server, err := config.Complete(informerFactory).New()
	// 将GenericAPIServer运行起来
	return server.GenericAPIServer.PrepareRun().Run(stopCh)
```

由于`k8s.io/apiserver`库的原理比较复杂，暂且不表，我们只讲创建`GenericAPIServer`的创建流程并说明`server`是如何使用的。对于`metric-server`而言，需要先创建一个`GenericAPIServer`，然后将`metric-server`自己的`API`与对应的处理`handler`注册进来即可。

```go
// pkg/apiserver/config.go
type MetricsServer struct {
	*genericapiserver.GenericAPIServer
}
// New returns a new instance of MetricsServer from the given config.
func (c completedConfig) New() (*MetricsServer, error) {
    // 创建一个名为"metrics-server"的genericServer
	genericServer, err := c.CompletedConfig.New("metrics-server", genericapiserver.NewEmptyDelegate()) // completion is done in Complete, no need for a second time
	if err != nil {
		return nil, err
	}
    // 关键逻辑：将metric-server的处理实体注册到genericServer中，下文继续讲解
	if err := generic.InstallStorage(c.ProviderConfig, c.SharedInformerFactory.Core().V1(), genericServer); err != nil {
		return nil, err
	}

	return &MetricsServer{
		GenericAPIServer: genericServer,
	}, nil
}

```

在注册`metric-server`自己的`API`时，需要先创建一个`APIGroup`（即`metrics.k8s.io`），然后将这个`Group`下面的各个资源（例如这里的`"nodes"`和`"pods"`）的`Storage`注册到`VersionedResourcesStorageMap`中，这里面最关键的就是每个资源的`Storage`需要实现处理请求的rest接口。

```go
// pkg/apiserver/generic/storage.go
// InstallStorage构造一个metrics.k8s.io的apiGroup并注册到genericServer中
func InstallStorage(providers *ProviderConfig, informers coreinf.Interface, server *genericapiserver.GenericAPIServer) error {
    // 创建一个APIGroup
	info := BuildStorage(providers, informers)
    // 将这个APIGroup注册到GenericAPIServer中
	return server.InstallAPIGroup(&info)
}

// BuildStorage构造一个名为"metrics.k8s.io"的apiGroup
func BuildStorage(providers *ProviderConfig, informers coreinf.Interface) genericapiserver.APIGroupInfo {
    // 创建一个名为"metrics.k8s.io"的apiGroup
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(metrics.GroupName, Scheme, metav1.ParameterCodec, Codecs)
    // 创建node的storage，这个storage跟pod的类似，下文以pod为例
	nodemetricsStorage := nodemetricsstorage.NewStorage(metrics.Resource("nodemetrics"), providers.Node, informers.Nodes().Lister())
    // 创建pod的storage，这个storage中实现了收到请求之后的处理逻辑，下文继续讲解
	podmetricsStorage := podmetricsstorage.NewStorage(metrics.Resource("podmetrics"), providers.Pod, informers.Pods().Lister())
    // 将node和pod的Storage存放到map中
	metricsServerResources := map[string]rest.Storage{
		"nodes": nodemetricsStorage,
		"pods":  podmetricsStorage,
	}
    // 在"metrics.k8s.io"这个apiGroup中注册"v1beta1"这个version的Storage处理map
	apiGroupInfo.VersionedResourcesStorageMap[v1beta1.SchemeGroupVersion.Version] = metricsServerResources
	return apiGroupInfo
}
```

这里以`pod`的`MetricStorage`的`Getter`接口为例，这里实现的`Get`函数就是当`metric-server`收到获取某个`pod`的`metric`的请求时处理该请求的Handler。

```go
// pkg/storage/podmetrics/reststorage.go
type MetricStorage struct {
	groupResource schema.GroupResource
	prov          provider.PodMetricsProvider
	podLister     v1listers.PodLister
}
// 用来检查MetricStorage已经实现了rest的这4个接口
var _ rest.KindProvider = &MetricStorage{}
var _ rest.Storage = &MetricStorage{}
var _ rest.Getter = &MetricStorage{}
var _ rest.Lister = &MetricStorage{}

func NewStorage(groupResource schema.GroupResource, prov provider.PodMetricsProvider, podLister v1listers.PodLister) *MetricStorage {
	return &MetricStorage{
		groupResource: groupResource,
		prov:          prov,
		podLister:     podLister,
	}
}

// 实现rest.Getter接口
func (m *MetricStorage) Get(ctx context.Context, name string, opts *metav1.GetOptions) (runtime.Object, error) {
	namespace := genericapirequest.NamespaceValue(ctx)
    // 从k8s的client缓存中获取pod信息
	pod, err := m.podLister.Pods(namespace).Get(name)
    
    // 省略异常处理的逻辑
    
    // 从之前的sinkMetric缓存中获取这个pod的
	podMetrics, err := m.getPodMetrics(pod)
    
    // 省略异常处理的逻辑
    
	return &podMetrics[0], nil
}

func (m *MetricStorage) getPodMetrics(pods ...*v1.Pod) ([]metrics.PodMetrics, error) {
	namespacedNames := make([]apitypes.NamespacedName, len(pods))
	for i, pod := range pods {
		namespacedNames[i] = apitypes.NamespacedName{
			Name:      pod.Name,
			Namespace: pod.Namespace,
		}
	}
    // 从缓存中获取pod的metric数据，这其实就是调用的4.2.2节中的GetContainerMetrics
	timestamps, containerMetrics, err := m.prov.GetContainerMetrics(namespacedNames...)
	if err != nil {
		return nil, err
	}
	res := make([]metrics.PodMetrics, 0, len(pods))
	for i, pod := range pods {
        // 省略pod状态不为Running和metric数据为空的continue逻辑

        // 创建返回体的内容
		res = append(res, metrics.PodMetrics{
			ObjectMeta: metav1.ObjectMeta{
				Name:              pod.Name,
				Namespace:         pod.Namespace,
				CreationTimestamp: metav1.NewTime(time.Now()),
			},
			Timestamp:  metav1.NewTime(timestamps[i].Timestamp),
			Window:     metav1.Duration{Duration: timestamps[i].Window},
			Containers: containerMetrics[i],
		})
	}
	return res, nil
}
```

这里我们终于又回到了4.2.2中的`sinkMetricsProvider`，这不过这次是从其`map`中读取数据。在此数据抓取和`metric-server`这两部分就连到一起了，我们其实也可以把这两部分当成一个生产者消费者模式，前者负责生产数据，后者则读取数据。

```go
// pkg/provider/sink/sinkprov.go
// metric-server收到pod的GET请求调用这个函数来获取metric数据
func (p *sinkMetricsProvider) GetContainerMetrics(pods ...apitypes.NamespacedName) ([]provider.TimeInfo, [][]metrics.ContainerMetrics, error) {
	p.mu.RLock()
	defer p.mu.RUnlock()

	timestamps := make([]provider.TimeInfo, len(pods))
	resMetrics := make([][]metrics.ContainerMetrics, len(pods))

	for i, pod := range pods {
        // 从pod的map中获取metric数据
		metricPoint, present := p.pods[pod]
		if !present {
			continue
		}

		// 省略中间的处理逻辑
		resMetrics[i] = contMetrics
	}
	return timestamps, resMetrics, nil
}
```

