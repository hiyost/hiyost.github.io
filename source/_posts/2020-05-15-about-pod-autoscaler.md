---
title: 每周学习一个组件系列之Horizontal Pod Autoscaler
date: 2020-05-15 16:05:00
tags: autoscaler
---

Pod 水平自动伸缩（Horizontal Pod Autoscaler）是k8s的`kube-controller-manager`中已经集成的一个`controller`，主要功能是根据pod当前的资源使用率来对deploy/rs自动扩缩容。详细介绍可以参考社区的[官方文档](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/)。

对于Pod 水平自动伸缩而言，最重要的就是其扩缩容的算法，社区原生的算法主要就是根据当前实例个数、当前metric指标和hpa中设置的期望指标来计算的，具体可以看2.2章节。

> 期望副本数 = ceil[当前副本数 * ( 当前指标 / 期望指标 )]

<!-- more --> 

### 1、启动参数

`kube-controller-manager`中包含了多个跟Pod 水平自动伸缩相关的启动参数：

| 参数名                                              | 参数解释                                                     |
| --------------------------------------------------- | ------------------------------------------------------------ |
| horizontal-pod-autoscaler-sync-period               | controller控制循环的检查周期（默认值为15秒）                 |
| horizontal-pod-autoscaler-upscale-delay             | 上次扩容之后，再次扩容需要等待的时间，默认                   |
| horizontal-pod-autoscaler-downscale-stabilization   | 上次缩容执行结束后，再次执行缩容的间隔，默认5分钟            |
| horizontal-pod-autoscaler-downscale-delay           | 上次扩容之后，再次扩容需要等待的时间，                       |
| horizontal-pod-autoscaler-tolerance                 | 缩放比例的容忍值，默认为0.1，即在0.9~1.1不会触发扩缩容       |
| horizontal-pod-autoscaler-use-rest-clients          | 使用rest client获取metric数据，支持custom metric时需要使用   |
| horizontal-pod-autoscaler-cpu-initialization-period | pod 的初始化时间， 在此时间内的 pod，CPU 资源指标将不会被采纳 |
| horizontal-pod-autoscaler-initial-readiness-delay   | pod 准备时间， 在此时间内的 pod 统统被认为未就绪             |

### 2、处理逻辑

#### 2.1、资源使用率来源

控制器将从一系列的聚合 API（`metrics.k8s.io`、`custom.metrics.k8s.io`和`external.metrics.k8s.io`） 中获取指标数据。 `metrics.k8s.io` API 通常由 `metrics-server`（需要额外部署）提供。通过 `metrics-server`获取到的数据如下，其中包含有cpu和memory的使用量：

```shell
[paas@192-168-199-200 ~]$ curl 127.0.0.1:8080/apis/metrics.k8s.io/v1beta1/namespaces/default/pods/test-delete-bcf859ddb-b7mjd
{
  "kind": "PodMetrics",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "name": "test-delete-bcf859ddb-b7mjd",
    "namespace": "default",
    "selfLink": "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods/test-delete-bcf859ddb-b7mjd",
    "creationTimestamp": "2020-05-22T02:58:34Z"
  },
  "timestamp": "2020-05-22T02:58:01Z",
  "window": "30s",
  "containers": [
    {
      "name": "container-0",
      "usage": {
        "cpu": "0",
        "memory": "1396Ki"
      }
    }
  ]
}
```

可以看到通过`metrics-server`获取数据是以pod为单位的，在`Pod Autoscaler`中，也是以pod为单位来做的自动伸缩。

`custom.metrics.k8s.io`需要配合`prometheus`等组件一起实现，暂且不表，后续介绍完了`prometheus`之后再慢慢讲。

#### 2.2、算法细节（来自社区文档）

从最基本的角度来看，pod 水平自动缩放控制器跟据当前指标和期望指标来计算缩放比例。其中当前指标就是通过2.1节中的接口来获取，期望指标则是hpa资源中设置的值。

```
期望副本数 = ceil[当前副本数 * ( 当前指标 / 期望指标 )]
```

例如，当前指标为`200m`，目标设定值为`100m`,那么由于`200.0 / 100.0 == 2.0`， 副本数量将会翻倍。 如果当前指标为`50m`，副本数量将会减半，因为`50.0 / 100.0 == 0.5`。 如果计算出的缩放比例接近1.0（跟据`--horizontal-pod-autoscaler-tolerance` 参数全局配置的容忍值，默认为0.1）， 将会放弃本次缩放。

如果 HorizontalPodAutoscaler 指定的是`targetAverageValue` 或 `targetAverageUtilization`， 那么将会把指定pod的平均指标做为`currentMetricValue`。 然而，在检查容忍度和决定最终缩放值前，我们仍然会把那些无法获取指标的pod统计进去。

所有被标记了删除时间戳(Pod正在关闭过程中)的 pod 和 失败的 pod 都会被忽略。

如果某个 pod 缺失指标信息，它将会被搁置，只在最终确定缩值时再考虑。

当使用 CPU 指标来缩放时，任何还未就绪（例如还在初始化）状态的 pod *或* 最近的指标为就绪状态前的 pod， 也会被搁置

由于受技术限制，pod 水平缩放控制器无法准确的知道 pod 什么时候就绪， 也就无法决定是否暂时搁置该 pod。 `--horizontal-pod-autoscaler-initial-readiness-delay` 参数（默认为30s），用于设置 pod 准备时间， 在此时间内的 pod 统统被认为未就绪。 `--horizontal-pod-autoscaler-cpu-initialization-period`参数（默认为5分钟），用于设置 pod 的初始化时间， 在此时间内的 pod，CPU 资源指标将不会被采纳。

在排除掉被搁置的 pod 后，缩放比例就会跟据`currentMetricValue / desiredMetricValue`计算出来。

如果有任何 pod 的指标缺失，我们会更保守地重新计算平均值， 在需要缩小时假设这些 pod 消耗了目标值的 100%， 在需要放大时假设这些 pod 消耗了0%目标值。 这可以在一定程度上抑制伸缩的幅度。

此外，如果存在任何尚未就绪的pod，我们可以在不考虑遗漏指标或尚未就绪的pods的情况下进行伸缩， 我们保守地假设尚未就绪的pods消耗了试题指标的0%，从而进一步降低了伸缩的幅度。

在缩放方向（缩小或放大）确定后，我们会把未就绪的 pod 和缺少指标的 pod 考虑进来再次计算使用率。 如果新的比率与缩放方向相反，或者在容忍范围内，则跳过缩放。 否则，我们使用新的缩放比例。

注意，平均利用率的*原始*值会通过 HorizontalPodAutoscaler 的状态体现（ 即使使用了新的使用率，也不考虑未就绪 pod 和 缺少指标的 pod)。

如果创建 HorizontalPodAutoscaler 时指定了多个指标， 那么会按照每个指标分别计算缩放副本数，取最大的进行缩放。 如果任何一个指标无法顺利的计算出缩放副本数（比如，通过 API 获取指标时出错）， 那么本次缩放会被跳过。

最后，在 HPA 控制器执行缩放操作之前，会记录缩放建议信息（scale recommendation）。 控制器会在操作时间窗口中考虑所有的建议信息，并从中选择得分最高的建议。 这个值可通过 kube-controller-manager 服务的启动参数 `--horizontal-pod-autoscaler-downscale-stabilization` 进行配置， 默认值为 5min。 这个配置可以让系统更为平滑地进行缩容操作，从而消除短时间内指标值快速波动产生的影响。

### 3、代码解析

#### 3.1、重要的数据结构

在进入代码之前，我们需要先说明一下与`Pod Autoscaler`相关的几个重要的`k8s`资源，这些资源是弹性伸缩实现的基石。

##### 3.1.1、`HorizontalPodAutoscaler`（v2beta1）

在最早期的`v1`版本的`HorizontalPodAutoscaler`中，只能指定`targetCPUUtilizationPercentage`，也就是说只能通过`pod`的`cpu`使用率来做弹性伸缩，功能比较局限。后面的`v2beta1`将这些资源做了拓展，已经不局限于`cpu`了。我们先举一个`v2beta1`的样例：

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta2
metadata:
  name: fff
  namespace: default
  selfLink: "/apis/autoscaling/v2beta2/namespaces/default/horizontalpodautoscalers/fff"
  uid: 698d7d88-82f7-4b2d-ad1a-467e07a45edb
  resourceVersion: '44279715'
  creationTimestamp: '2020-05-14T06:51:45Z'
spec:
  scaleTargetRef:  ## 指定这个hpa策略作用于哪个资源上，其namespace跟当前hpa的namespace一致
    kind: Deployment
    name: fffff
    apiVersion: apps/v1beta2
  minReplicas: 1   ## 最小实例个数
  maxReplicas: 10  ## 最大实例个数
  metrics:   ## 从v2beta1开始targetCPUUtilizationPercentage已经改成了metrics，可以指定更多维度的指标
  - type: Resource   ## metric类型，有Resource、Pods、Object、External四种，这里的Resource是根据资源使用情况
    resource:
      name: memory
      target:
        type: Utilization   ## target类型，表示资源的平均使用率，还有Value（使用量）、AverageValue（平均使用量）
        averageUtilization: 70
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
status:
  lastScaleTime: '2020-05-14T06:57:01Z'  ## 上次伸缩的时间，冷却时间就是根据这个做的判断
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics: ## 当前获取的指标结果
  - type: Resource
    resource:
      name: memory
      current:
        averageValue: '1433600'
        averageUtilization: 1
  - type: Resource
    resource:
      name: cpu
      current:
        averageValue: '0'
        averageUtilization: 0
  conditions:
  - type: AbleToScale  ## 表明 HPA 是否可以获取和更新伸缩信息，以及是否存在阻止伸缩的各种回退条件
    status: 'True'
    lastTransitionTime: '2020-05-14T06:52:00Z'
    reason: ReadyForNewScale
    message: recommended size matches current size
  - type: ScalingActive  ## 表明HPA是否被启用（即目标的副本数量不为零） 以及是否能够完成伸缩计算。 当这一状态为 False 时，通常表明获取度量指标存在问题。
    status: 'True'
    lastTransitionTime: '2020-05-25T11:27:20Z'
    reason: ValidMetricFound
    message: the HPA was able to successfully calculate a replica count from memory
      resource utilization (percentage of request)
  - type: ScalingLimited  ## 表明所需伸缩的值被 HorizontalPodAutoscaler 所定义的最大或者最小值所限制（即已经达到最大或者最小伸缩值）
    status: 'False'
    lastTransitionTime: '2020-05-14T06:52:00Z'
    reason: DesiredWithinRange
    message: the desired count is within the acceptable range
```

在`HorizontalPodAutoscaler`中，很关键的一个字段就是`spec.metrics`，用来指定自动伸缩所根据的指标，分别有4种类型：

- `Resource`：资源使用率，可以是`pod`的`cpu`、`memory`，对应的指标可以是使用率、使用量、平均使用量，前面的样例中使用的就是这种类型

- `Pods`：表示指定的指标在不同Pod之间进行平均，并通过与一个目标值比对来确定副本的数量。 它们的工作方式与资源度量指标非常相像，差别是它们仅支持`target` 类型为`AverageValue`。样例如下：

  ```yaml
  type: Pods
  pods:
    metric:
      name: packets-per-second
    target:
      type: AverageValue
      averageValue: 1k
  ```

- `Object`：`k8s`中某一个可以描述资源指标的对象，例如有`hits-per-second`指标的`Ingress`对象，表示`requests-per-second`的度量指标样例如下。需要注意的是，这个资源对象需要与其作用的`pod`在相同`namespace`中。

  ```yaml
  type: Object
  object:
    metric:
      name: requests-per-second
    describedObject:
      apiVersion: networking.k8s.io/v1beta1
      kind: Ingress
      name: main-route
    target:
      type: Value
      value: 2k
  ```

- `External`：使用外部的度量指标，这些指标依赖于外部的监控系统，这种方式不太推荐，更建议使用`custom metric`的方式。

由于`spec.metrics`是一个slice，因此可以同时指定多个指标类型，`HorizontalPodAutoscaler` 会计算每一个指标所提议的副本数量，然后最终选择一个最高值。

#### 3.2、代码目录

跟k8s中其他官方的controller一样，`Pod Autoscaler`控制器的代码主要都位于k8s中，主要代码逻辑在`horizontal.go`中，其他文件中的代码见下面的说明。

```shell
# pkg/controller/podautoscaler/
├── config   # HPAController的启动参数相关
│   ├── BUILD
│   ├── doc.go
│   ├── OWNERS
│   ├── types.go
│   ├── v1alpha1
│   │   ├── BUILD
│   │   ├── conversion.go
│   │   ├── defaults.go
│   │   ├── doc.go
│   │   ├── register.go
│   │   ├── zz_generated.conversion.go
│   │   └── zz_generated.deepcopy.go
│   └── zz_generated.deepcopy.go
├── doc.go
├── horizontal.go   # 主代码逻辑
├── horizontal_test.go
├── legacy_horizontal_test.go
├── legacy_replica_calculator_test.go
├── metrics
│   ├── BUILD
│   ├── interfaces.go   # 获取metric数值的接口
│   ├── legacy_metrics_client.go   # 调用heapster获取metric的实现
│   ├── legacy_metrics_client_test.go
│   ├── rest_metrics_client.go   # 调用metric-server获取metric的实现
│   ├── rest_metrics_client_test.go
│   ├── utilization.go   # 计算当前使用率的公共函数
│   └── utilization_test.go
├── OWNERS
├── rate_limiters.go   # HPAController限流器
├── replica_calculator.go   # 根据当前使用率计算pod期望个数的函数实现
└── replica_calculator_test.go
```

#### 3.3、代码逻辑

为了方便理解`HorizontalController`本身的逻辑，这里不去细究`kube-controller-manager`本身的框架，也不去细究`k8s`中`list-watch`机制中的`informer`相关的具体细节，遇到时也仅解析其功能和用法。

##### 3.3.1、创建`HorizontalController`

创建`HorizontalController`时一个十分关键的参数是`resyncPeriod`，即启动参数`horizontal-pod-autoscaler-sync-period`，`controller`中并不是通过`Ticker`的方式来定时去做检查，而是在`informer`中定时把所有资源发送到`controller`的处理队列中，这种方式的好处是不需要给每个`hpa`资源都创建一个`Ticker`。这个功能主要是`informer`自己去实现的，对这部分感兴趣的话可以去瞅瞅`staging/src/k8s.io/client-go/tools/cache`里面的代码，非常值得研究。

```go
// pkg/controller/podautoscaler/horizontal.go
func NewHorizontalController(
	evtNamespacer v1core.EventsGetter,
	scaleNamespacer scaleclient.ScalesGetter,
	hpaNamespacer autoscalingclient.HorizontalPodAutoscalersGetter,
	mapper apimeta.RESTMapper,
	metricsClient metricsclient.MetricsClient,
	hpaInformer autoscalinginformers.HorizontalPodAutoscalerInformer,
	podInformer coreinformers.PodInformer,
	resyncPeriod time.Duration,
	downscaleStabilisationWindow time.Duration,
	tolerance float64,
	cpuInitializationPeriod,
	delayOfInitialReadinessStatus time.Duration,
) *HorizontalController {
    // 其他代码略

	hpaInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    hpaController.enqueueHPA,
			UpdateFunc: hpaController.updateHPA,
			DeleteFunc: hpaController.deleteHPA,
		},
		resyncPeriod, // 设置informer的resync时间，即每隔一段时间informer会将缓存中的所有对象的key都放入队列中
	)
    // 其他代码略

	return hpaController
}
```

##### 3.3.2、运行`controller`

相比于创建`controller`时的各种涉及到`informer`参数的设置，运行`controller`的逻辑明显清晰很多，入口就是其`Run`函数，其中会起一个携程来运行实际的处理逻辑。在这个协程中，会不断从informer的队列中取出需要处理的HPA对象的key，然后使用这个key去缓存中获取对应的HPA对象的完整对象。

```go
// pkg/controller/podautoscaler/horizontal.go
// Run begins watching and syncing.
func (a *HorizontalController) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer a.queue.ShutDown()
	klog.Infof("Starting HPA controller")
	defer klog.Infof("Shutting down HPA controller")
	if !controller.WaitForCacheSync("HPA", stopCh, a.hpaListerSynced, a.podListerSynced) {
		return
	}
    // 起一个协程来运行worker
	go wait.Until(a.worker, time.Second, stopCh)
	<-stopCh
}

func (a *HorizontalController) worker() {
    // 循环调用processNextWorkItem直到其返回false时停止
	for a.processNextWorkItem() {
	}
	klog.Infof("horizontal pod autoscaler controller worker shutting down")
}

func (a *HorizontalController) processNextWorkItem() bool {
    // 从队列中取出待处理的HPA对象的key
	key, quit := a.queue.Get()
	if quit {
		return false
	}
	defer a.queue.Done(key)
    // 关键逻辑：处理这个HPA对象
	deleted, err := a.reconcileKey(key.(string))
	if err != nil {
		utilruntime.HandleError(err)
	}
	// 处理出现问题时重新入队列
	if !deleted {
		a.queue.AddRateLimited(key)
	}
	return true
}

func (a *HorizontalController) reconcileKey(key string) (deleted bool, err error) {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return true, err
	}
    // 根据hap的name和namespace从缓存中获取hpa的完整对象
	hpa, err := a.hpaLister.HorizontalPodAutoscalers(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.Infof("Horizontal Pod Autoscaler %s has been deleted in %s", name, namespace)
		delete(a.recommendations, key)
		return true, nil
	}
    // 去处理这个HPA对象
	return false, a.reconcileAutoscaler(hpa, key)
}
```

在处理HPA对象时，主要分为3步：

1. 获取目标`deploy`或者`rs`的`scale`数据，包含`deploy`或者`rs`中`spec.replicas`指定的实例个数
2. 根据当前的情况计算是否需要扩缩容以及期望的实例个数
3. 需要`scale`的情况则调用接口对改资源进行弹性伸缩，否则保持不变

```go
// pkg/controller/podautoscaler/horizontal.go
func (a *HorizontalController) reconcileAutoscaler(hpav1Shared *autoscalingv1.HorizontalPodAutoscaler, key string) error {
	// make a copy so that we never mutate the shared informer cache (conversion can mutate the object)
	hpav1 := hpav1Shared.DeepCopy()
	// then, convert to autoscaling/v2, which makes our lives easier when calculating metrics
	hpaRaw, err := unsafeConvertToVersionVia(hpav1, autoscalingv2.SchemeGroupVersion)
	if err != nil {
		a.eventRecorder.Event(hpav1, v1.EventTypeWarning, "FailedConvertHPA", err.Error())
		return fmt.Errorf("failed to convert the given HPA to %s: %v", autoscalingv2.SchemeGroupVersion.String(), err)
	}
    // 转换成v2版本的hpa
	hpa := hpaRaw.(*autoscalingv2.HorizontalPodAutoscaler)
	hpaStatusOriginal := hpa.Status.DeepCopy()

	reference := fmt.Sprintf("%s/%s/%s", hpa.Spec.ScaleTargetRef.Kind, hpa.Namespace, hpa.Spec.ScaleTargetRef.Name)

	targetGV, err := schema.ParseGroupVersion(hpa.Spec.ScaleTargetRef.APIVersion)
    // 错误处理略
	targetGK := schema.GroupKind{
		Group: targetGV.Group,
		Kind:  hpa.Spec.ScaleTargetRef.Kind,
	}
	mappings, err := a.mapper.RESTMappings(targetGK)
	// 错误处理略

    // 1、获取目标deploy或者rs的scale数据，包含deploy或者rs中spec.replicas指定的实例个数
	scale, targetGR, err := a.scaleForResourceMappings(hpa.Namespace, hpa.Spec.ScaleTargetRef.Name, mappings)
	// 错误处理略
	currentReplicas := scale.Spec.Replicas
	a.recordInitialRecommendation(currentReplicas, key)
	var (
		metricStatuses        []autoscalingv2.MetricStatus
		metricDesiredReplicas int32
		metricName            string
	)
	desiredReplicas := int32(0)  // 这里最终就是为了根据当前的资源使用情况计算出这个期望的实例个数
	rescaleReason := ""

	rescale := true  // 是否需要伸缩

    // 2、根据当前的情况计算是否需要扩缩容以及期望的实例个数
	if scale.Spec.Replicas == 0 {
		// 如果当前这个deploy或者rs的spec.replicas是0则不做处理
		desiredReplicas = 0
		rescale = false
		setCondition(hpa, autoscalingv2.ScalingActive, v1.ConditionFalse, "ScalingDisabled", "scaling is disabled since the replica count of the target is zero")
	} else if currentReplicas > hpa.Spec.MaxReplicas {
        // 如果当前设置的replicas超过了上限，则设置为上限的值
		rescaleReason = "Current number of replicas above Spec.MaxReplicas"
		desiredReplicas = hpa.Spec.MaxReplicas
	} else if hpa.Spec.MinReplicas != nil && currentReplicas < *hpa.Spec.MinReplicas {
        // 如果当前设置的replicas超过了下限，则设置为下限的值
		rescaleReason = "Current number of replicas below Spec.MinReplicas"
		desiredReplicas = *hpa.Spec.MinReplicas
	} else if currentReplicas == 0 {
        // 没看懂这个跟第一个if有啥区别
		rescaleReason = "Current number of replicas must be greater than 0"
		desiredReplicas = 1
	} else {
        // 关键的分支来了，根据当前的指标计算期望实例个数的分支
		var metricTimestamp time.Time
        // 关键逻辑：根据当前的指标计算期望实例个数的分支
		metricDesiredReplicas, metricName, metricStatuses, metricTimestamp, err = a.computeReplicasForMetrics(hpa, scale, hpa.Spec.Metrics)
		// 错误处理和event略
		rescaleMetric := ""
		if metricDesiredReplicas > desiredReplicas {
			desiredReplicas = metricDesiredReplicas
			rescaleMetric = metricName
		}
		// 中间略
		desiredReplicas = a.normalizeDesiredReplicas(hpa, key, currentReplicas, desiredReplicas)
		rescale = desiredReplicas != currentReplicas
	}

    // 3、需要scale的情况则调用接口对改资源进行弹性伸缩，否则保持不变
	if rescale {
		scale.Spec.Replicas = desiredReplicas
		_, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(targetGR, scale)
		// 错误处理和日志略
	} else {
		klog.V(4).Infof("decided not to scale %s to %v (last scale time was %s)", reference, desiredReplicas, hpa.Status.LastScaleTime)
		desiredReplicas = currentReplicas
	}

	a.setStatus(hpa, currentReplicas, desiredReplicas, metricStatuses, rescale)
	return a.updateStatusIfNeeded(hpaStatusOriginal, hpa)
}
```

前面的逻辑中，最重要的一步就是根据当前的资源使用情况来计算期望的实例个数，也就是`computeReplicasForMetrics`函数。在这个函数中，主要就是遍历所有的`metric`策略（3.1.1节中有提到不同类型），并计算各个策略下的期望实例个数，最终取最大的那个。这段逻辑最终进入到计算单个策略的函数`computeReplicasForMetric`中。

```go
// pkg/controller/podautoscaler/horizontal.go
func (a *HorizontalController) computeReplicasForMetrics(hpa *autoscalingv2.HorizontalPodAutoscaler, scale *autoscalingv1.Scale,
	metricSpecs []autoscalingv2.MetricSpec) (replicas int32, metric string, statuses []autoscalingv2.MetricStatus, timestamp time.Time, err error) {

	specReplicas := scale.Spec.Replicas
	statusReplicas := scale.Status.Replicas
	statuses = make([]autoscalingv2.MetricStatus, len(metricSpecs))
    // 异常处理略
	selector, err := labels.Parse(scale.Status.Selector)
    // 异常处理略
	invalidMetricsCount := 0
	var invalidMetricError error
    // 计算每一个metric配置的期望，并取其中计算结果最大的值（3.1.1节中已经说明了多中类型的metric）
	for i, metricSpec := range metricSpecs {
		replicaCountProposal, metricNameProposal, timestampProposal, err := a.computeReplicasForMetric(hpa, metricSpec, specReplicas, statusReplicas, selector, &statuses[i])
        // 异常处理略
        // 取最大的那个期望值
		if err == nil && (replicas == 0 || replicaCountProposal > replicas) {
			timestamp = timestampProposal
			replicas = replicaCountProposal
			metric = metricNameProposal
		}
	}
	// 异常处理略
	return replicas, metric, statuses, timestamp, nil
}
```

对于每一个策略而言，需要根据其类型做不同的处理，`computeReplicasForMetric`中主要就是一个`case`语句，由于有4中类型，其中`Resource`较为常用，因此后面就只对这种类型进行代码分析。`Resource`类型的计算是由`computeStatusForResourceMetric`来处理的。

```go
// pkg/controller/podautoscaler/horizontal.go
func (a *HorizontalController) computeReplicasForMetric(hpa *autoscalingv2.HorizontalPodAutoscaler, spec autoscalingv2.MetricSpec,
	specReplicas, statusReplicas int32, selector labels.Selector, status *autoscalingv2.MetricStatus) (replicaCountProposal int32, metricNameProposal string,
	timestampProposal time.Time, err error) {

    // 针对不同类型的metric使用不同方法的计算
	switch spec.Type {
	case autoscalingv2.ObjectMetricSourceType:
		metricSelector, err := metav1.LabelSelectorAsSelector(spec.Object.Metric.Selector)
		// 异常处理略
		replicaCountProposal, timestampProposal, metricNameProposal, err = a.computeStatusForObjectMetric(specReplicas, statusReplicas, spec, hpa, selector, status, metricSelector)
		// 异常处理略
	case autoscalingv2.PodsMetricSourceType:
		metricSelector, err := metav1.LabelSelectorAsSelector(spec.Pods.Metric.Selector)
		// 异常处理略
		replicaCountProposal, timestampProposal, metricNameProposal, err = a.computeStatusForPodsMetric(specReplicas, spec, hpa, selector, status, metricSelector)
		// 异常处理略
	case autoscalingv2.ResourceMetricSourceType:
		replicaCountProposal, timestampProposal, metricNameProposal, err = a.computeStatusForResourceMetric(specReplicas, spec, hpa, selector, status)
		// 异常处理略
	case autoscalingv2.ExternalMetricSourceType:
		replicaCountProposal, timestampProposal, metricNameProposal, err = a.computeStatusForExternalMetric(specReplicas, statusReplicas, spec, hpa, selector, status)
		// 异常处理略
	default:
		errMsg := fmt.Sprintf("unknown metric source type %q", string(spec.Type))
		// 异常处理略
	}
	return replicaCountProposal, metricNameProposal, timestampProposal, nil
}
```

在`computeStatusForResourceMetric`中，主要是针对`Resource`类型的策略进行处理，在3.1.1章节中已经说明`Resource`类型的策略对应的指标可以是：

- 平均使用率`AverageUtilization`
- 使用量`Value`
- 平均使用量`AverageValue`

但是在`computeStatusForResourceMetric`函数中，似乎并不支持使用量`Value`这种类型。

```go
// pkg/controller/podautoscaler/horizontal.go
// 针对3中不同的指标类型分别进行计算
func (a *HorizontalController) computeStatusForResourceMetric(currentReplicas int32, metricSpec autoscalingv2.MetricSpec, hpa *autoscalingv2.HorizontalPodAutoscaler, selector labels.Selector, status *autoscalingv2.MetricStatus) (int32, time.Time, string, error) {
	if metricSpec.Resource.Target.AverageValue != nil {
        // 平均使用量的计算方法
		var rawProposal int64
		replicaCountProposal, rawProposal, timestampProposal, err := a.replicaCalc.GetRawResourceReplicas(currentReplicas, metricSpec.Resource.Target.AverageValue.MilliValue(), metricSpec.Resource.Name, hpa.Namespace, selector)
		// 异常处理略
		metricNameProposal := fmt.Sprintf("%s resource", metricSpec.Resource.Name)
		*status = autoscalingv2.MetricStatus{
			Type: autoscalingv2.ResourceMetricSourceType,
			Resource: &autoscalingv2.ResourceMetricStatus{
				Name: metricSpec.Resource.Name,
				Current: autoscalingv2.MetricValueStatus{
					AverageValue: resource.NewMilliQuantity(rawProposal, resource.DecimalSI),
				},
			},
		}
		return replicaCountProposal, timestampProposal, metricNameProposal, nil
	} else {
        // 平均使用率的计算方法
		if metricSpec.Resource.Target.AverageUtilization == nil {
            // 这里可以看到似乎并不支持metricSpec.Resource.Target.Value？
			errMsg := "invalid resource metric source: neither a utilization target nor a value target was set"
			a.eventRecorder.Event(hpa, v1.EventTypeWarning, "FailedGetResourceMetric", errMsg)
			setCondition(hpa, autoscalingv2.ScalingActive, v1.ConditionFalse, "FailedGetResourceMetric", "the HPA was unable to compute the replica count: %s", errMsg)
			return 0, time.Time{}, "", fmt.Errorf(errMsg)
		}
        // 
		targetUtilization := *metricSpec.Resource.Target.AverageUtilization
		var percentageProposal int32
		var rawProposal int64
        // 关键逻辑：根据资源使用率来计算期望的实例个数
		replicaCountProposal, percentageProposal, rawProposal, timestampProposal, err := a.replicaCalc.GetResourceReplicas(currentReplicas, targetUtilization, metricSpec.Resource.Name, hpa.Namespace, selector)
		// 异常处理略
		metricNameProposal := fmt.Sprintf("%s resource utilization (percentage of request)", metricSpec.Resource.Name)
		*status = autoscalingv2.MetricStatus{
			Type: autoscalingv2.ResourceMetricSourceType,
			Resource: &autoscalingv2.ResourceMetricStatus{
				Name: metricSpec.Resource.Name,
				Current: autoscalingv2.MetricValueStatus{
					AverageUtilization: &percentageProposal,
					AverageValue:       resource.NewMilliQuantity(rawProposal, resource.DecimalSI),
				},
			},
		}
		return replicaCountProposal, timestampProposal, metricNameProposal, nil
	}
}
```

由于篇幅有限，本文只说明一下平均使用率`AverageUtilization`这种情况。

```go
// pkg/controller/podautoscaler/replica_calculator.go
func (c *ReplicaCalculator) GetResourceReplicas(currentReplicas int32, targetUtilization int32, resource v1.ResourceName, namespace string, selector labels.Selector) (replicaCount int32, utilization int32, rawUtilization int64, timestamp time.Time, err error) {
    // 1、调用metric-server的接口获取deploy/rs下所有pod的metric
	metrics, timestamp, err := c.metricsClient.GetResourceMetric(resource, namespace, selector)
	// 异常处理略
    
    // 2、从缓存中获取deploy/rs下所有pod信息
	podList, err := c.podLister.Pods(namespace).List(selector)
	// 异常处理略

	itemsLen := len(podList)
	// 异常处理略

    // 3、根据pod信息统计Ready的pod个数、未就绪的ignoredPods、查不到metric数据的missingPods
	readyPodCount, ignoredPods, missingPods := groupPods(podList, metrics, resource, c.cpuInitializationPeriod, c.delayOfInitialReadinessStatus)
    // 4、去掉未就绪的pod的metric（现在metrics里面只剩下ready的pod的指标）
	removeMetricsForPods(metrics, ignoredPods)
    // 5、获取每个pod这个resource的request的总和
	requests, err := calculatePodRequests(podList, resource)
	// 异常处理略
    // 6、根据metric和request的值计算当前资源使用率和期望使用率的比值、当前资源使用率、当前资源使用平均值
	usageRatio, utilization, rawUtilization, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
	// 异常处理略
    
    //   如果当前计算出的结果是需要扩容，并有有查不到metric的pod，后面需要重新计算
	rebalanceIgnored := len(ignoredPods) > 0 && usageRatio > 1.0
	if !rebalanceIgnored && len(missingPods) == 0 {
		if math.Abs(1.0-usageRatio) <= c.tolerance {
			// 当前资源使用率和期望使用率差距在容忍范围内时，不做修改
            return currentReplicas, utilization, rawUtilization, timestamp, nil
		}
		// 没有查不到metric的pod，且确实需要扩容，则进行扩容
		return int32(math.Ceil(usageRatio * float64(readyPodCount))), utilization, rawUtilization, timestamp, nil
	}

	if len(missingPods) > 0 {
		if usageRatio < 1.0 {
			// 需要缩容时，将查不到metric的pod当做资源使用了100%看待
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: requests[podName]}
			}
		} else if usageRatio > 1.0 {
			// 需要扩容时，将查不到metric的pod当做资源使用了0%看待
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: 0}
			}
		}
	}

	if rebalanceIgnored {
		// 需要扩容时，将notready的pod当做资源使用了0%看待
		for podName := range ignoredPods {
			metrics[podName] = metricsclient.PodMetric{Value: 0}
		}
	}

	// 7、在考虑了查不到metric的pod和notready的pod的情况下重新计算当前资源使用率和期望使用率的比值
	newUsageRatio, _, _, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
	// 异常处理略

	if math.Abs(1.0-newUsageRatio) <= c.tolerance || (usageRatio < 1.0 && newUsageRatio > 1.0) || (usageRatio > 1.0 && newUsageRatio < 1.0) {
		// 当新结果也在容忍范围内或者新旧结果完全相反时，不做修改
		return currentReplicas, utilization, rawUtilization, timestamp, nil
	}

	// 按照算法计算期望的实例个数
	return int32(math.Ceil(newUsageRatio * float64(len(metrics)))), utilization, rawUtilization, timestamp, nil
}
```


### 4、参考文档

https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/

https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/horizontal-pod-autoscaler.html
