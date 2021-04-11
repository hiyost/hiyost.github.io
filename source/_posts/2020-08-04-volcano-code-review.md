---
title: volcano代码分析
date: 2020-08-04 15:47:30
tags: volcano
---



### 1、整体框架

在了解volcano的结构之前，我们需要先知道为什么要有volcano，它到底解决了哪些kube-scheduler无法解决的问题。要回答这个问题，我们需要先了解kube-scheduler的调度机制，我们知道，kube-scheduler是以pod为单位来进行调度的，除了通过亲和性来做一些pod之间的关系处理之外，并没有任何pod间的关联机制。举一个例子，在AI等训练的场景，是需要一批pod同时工作的，而且这些pod要么一起调度成功，要么一起调度失败，部分调度成功部分调度失败会导致整个任务最重还是失败的，而且调度成功的那些pod还浪费了资源，这种要么一起成功要么一起失败的场景是kube-scheduler无法解决的，所以才催生了volcano以及volcano的前身kube-batch的诞生。

<!-- more -->



volcano整体架构可以直接参考[官方网站的文档](https://volcano.sh/docs/architecture/)，这里做一下简要的概述：



![image-20200805153351133](2020-08-04-volcano-code-review/arch_2.png)

volcano主要由scheduler、controller和webhook三部分组成，其中：

- scheduler主要负责batch调度，如fair-share（优先调度占用资源少的）、gang-scheduling（pod组批量调度）等
- controller主要负责对用户创建的`batch.volcano.sh/v1alpha1/job`以及其他crd资源进行reconcile
- webhook则是对kube-apiserver收到的请求镜像validate和admission处理

### 2、controller

在了解controller之前，我们需要先了解volcano中到底管理的是哪些CRD资源，在部署volcano的[yaml](https://github.com/volcano-sh/volcano/blob/master/installer/volcano-development.yaml)中可以看到一共创建了4个CRD，分别是`Job`、`Command`、`PodGroup`和`Queue`。其中每一个controller的功能如下：





#### 2.1、Job

##### 2.1.1、数据结构





##### 2.1.2、代码逻辑

job-controller中主要list-watch的是`Job`和`Pod`，同时也会watch`Command`资源的add、`PodGroup`的update和`PriorityClasses`的add和delete。job controller中有两个特别重要的成员，分别是`queueList`和`cache`：

```go
// pkg/controllers/job/job_controller.go
// jobcontroller the Job jobcontroller type.
type jobcontroller struct {
	// queueList是所有watch到的对象，注意这里之所以是slice是为了多worker协同，每个worker一个queue
    // controller根据job的namecepace-name来进行hash后随机分配到各个queue中，其实这可能存在一定的
	queueList    []workqueue.RateLimitingInterface
    // cache是所有list到的job对象的缓存
	cache        jobcache.Cache
	// 其他成员略
}
```

`queueList`的本质是一个队列，队列的元素是自定义的一个`Request`对象，可以看到`Request`中主要包含的是跟`Job`相关的key信息，这也符合一般的队列模型，queue中存放key，cache中存放实际的数据：

```go
// pkg/controllers/apis/request.go
// Request struct.
type Request struct {
	Namespace string    // job的namespace
	JobName   string    // job的name
	TaskName  string    // task的name
	QueueName string    // 分配到的Queue的name

	Event      v1alpha1.Event
	ExitCode   int32
	Action     v1alpha1.Action
	JobVersion int32
}
```

`cache`的本质是一个`Job`资源的map，key是`namespace/name`：

```go
// pkg/controllers/cache/cache.go
type jobCache struct {
	sync.Mutex
    // 保存所有的job资源，key是namespace/name
	jobs        map[string]*apis.JobInfo
    // 待清理的job
	deletedJobs workqueue.RateLimitingInterface
}
```

value中既包含了`Job`的信息，也包含了这个job对应的`Pods`的信息

```go
type JobInfo struct {
	Namespace string
	Name      string
	Job  *batch.Job    // 保存的Job实体
	Pods map[string]map[string]*v1.Pod  // 
}
```

知道了controller中的关键的数据结构，我们也就能猜测controller的reconcile的逻辑了：生产者通过list-watch将`Job`的key信息加入到`queueList`中，将`Job`的实体信息保存到`jobCache`中缓存，消费者从`queueList`中获取数据并进行处理。其主要代码在`processNextReq`函数中：

```go
// pkg/controllers/job/job_controller.go
func (cc *jobcontroller) processNextReq(count uint32) bool {
    // 前面部分代码略，主要是从队列中获取key信息
    // 然后从cache中获取job实体
	jobInfo, err := cc.cache.Get(jobcache.JobKeyByReq(&req))
    // 1、将job根据其job.Status.State.Phase封装成state
	st := state.NewState(jobInfo)
    // 2、根据job的policy等信息获取action
	action := applyPolicies(jobInfo.Job, &req)
    // 3、执行对应state的逻辑，里面主要是KillJob或者SyncJob
	if err := st.Execute(action); err != nil {
		// 错误处理，如果执行失败则重试，如果失败次数超过最大重试次数则重新入队列
	}
	// 处理成功则从队列中移除
	queue.Forget(req)
	return true
}
```

不同Action和State对应的处理逻辑（空白的为`KillJob`）：

| Action\State | Pending | Aborting | Aborted | Running | Restarting | Completing | Terminating | Terminated<br>Completed<br>Failed |
| ------------ | ------- | -------- | ------- | ------- | ---------- | ---------- | ----------- | --------------------------------- |
| AbortJob     |         |          |         |         |            |            |             |                                   |
| RestartJob   |         |          |         |         |            |            |             |                                   |
| RestartTask  | SyncJob |          |         | SyncJob |            |            |             |                                   |
| TerminateJob |         |          |         |         |            |            |             |                                   |
| CompleteJob  |         |          |         |         |            |            |             |                                   |
| ResumeJob    | SyncJob |          |         | SyncJob |            |            |             |                                   |
| SyncJob      | SyncJob |          |         | SyncJob |            |            |             |                                   |
| EnqueueJob   | SyncJob |          |         | SyncJob |            |            |             |                                   |
| SyncQueue    | SyncJob |          |         | SyncJob |            |            |             |                                   |
| OpenQueue    | SyncJob |          |         | SyncJob |            |            |             |                                   |
| CloseQueue   | SyncJob |          |         | SyncJob |            |            |             |                                   |

通过以上过程我们可以看到Job的reconcile中存在比较多的状态，因此代码中使用了Action和State两个状态来进行状态机的转移，不过最终处理的逻辑主要就是`SyncJob`和`KillJob`两种，因此我们主要分析这两部分的逻辑。

###### `SyncJob`

从前面状态转移的表格中，我们看到只有`Pending`和`Running`状态才有机会进入到`SyncJob`的流程中，其实也就是从创建Job到Job正常运行的过程，因此可以预料`SyncJob`主要就是将Job运行起来。主要的流程如下：

- 对于新Job先进行初始化：创建对应的PVC（volcano中的pvc需要自己管理，没有k8s的controller）和PodGroup（一个Job对应一个PodGroup），注意创建出来的PodGroup则由PodGroup controller管理且其name和Job保持一致
- 根据Job中的Task变化来生成要创建的pod list和要删除的pod list
- 分别起协程调用kube-apiserver创建和删除这两个list中的pod，需要注意的是，为了不让k8s的调度器处理这些新创建的pod，Job中需要携带调度器的信息并最终传入到pod上，这样k8s的调度器会过滤掉这些带有volcano调度器名字的pod，同样volcano的调度器则只会过滤出这些带有volcano调度器名字的pod，避免相互影响，在创建Job的时候，webhook中会默认给Job加上`volcano`这个调度器名字
- 更新Job的状态，更新jobCache缓存

```go
// pkg/controllers/job/job_controller_actions.go
func (cc *jobcontroller) syncJob(jobInfo *apis.JobInfo, updateStatus state.UpdateStatusFn) error {
    // 部分略
	// job的状态（job.Status.State.Phase）为空或者Pending或者Restarting则进行初始化
	if !isInitiated(job) {
		var err error
        // 初始化Job，主要是创建对应的PVC和PodGroup，注意PodGroup中的Name同Job的Name
        //     PodGroup中的QueueName是直接使用了Job中的QueueName
		if job, err = cc.initiateJob(job); err != nil {
			return err
		}
	} else {
		var err error
		// 如果job已经初始化了，则根据job的MinAvailable字段是否变化来更新PodGroup
		if err = cc.initOnJobUpdate(job); err != nil {
			return err
		}
	}
    // 初始化后更新job状态，代码略

    // 根据job中的Tasks的template创建对应的Pod样例
	for _, ts := range job.Spec.Tasks {
		// 部分代码略
		for i := 0; i < int(ts.Replicas); i++ {
            // 可以看到podname是带有编号的
			podName := fmt.Sprintf(jobhelpers.PodNameFmt, job.Name, name, i)
			if pod, found := pods[podName]; !found {
                // pod不存在时，创建新pod的样例，注意创建的时会将job、task相关的信息写到pod的annotation和label中
                //   其中有一个很关键的信息是会把前面创建的PodGroup的name写到pod的annotation中
				newPod := createJobPod(job, tc, i)
				if err := cc.pluginOnPodCreate(job, newPod); err != nil {
					return err
				}
				podToCreate = append(podToCreate, newPod)
			} else {
                // 删掉已经存在的pod，目的是缩容时删除缓存中前面的pod，然后将剩下的加入到删除队列中
				delete(pods, podName)
				if pod.DeletionTimestamp != nil {
					klog.Infof("Pod <%s/%s> is terminating", pod.Namespace, pod.Name)
					atomic.AddInt32(&terminating, 1)
					continue
				}

				classifyAndAddUpPodBaseOnPhase(pod, &pending, &running, &succeeded, &failed, &unknown)
			}
		}
        // 将剩下的pod加入到删除队列中（缩容时的场景），可以看到缩容时时优先删除
		for _, pod := range pods {
			podToDelete = append(podToDelete, pod)
		}
	}

	waitCreationGroup := sync.WaitGroup{}
	waitCreationGroup.Add(len(podToCreate))
	for _, pod := range podToCreate {
        // 起协程创建pod
		go func(pod *v1.Pod) {
			defer waitCreationGroup.Done()
			newPod, err := cc.kubeClient.CoreV1().Pods(pod.Namespace).Create(context.TODO(), pod, metav1.CreateOptions{})
            // 错误处理逻辑略
		}(pod)
	}
	waitCreationGroup.Wait() // 等待协程执行完

	// 错误处理逻辑略

	waitDeletionGroup := sync.WaitGroup{}
	waitDeletionGroup.Add(len(podToDelete))
	for _, pod := range podToDelete {
        // 起协程删除pod
		go func(pod *v1.Pod) {
			defer waitDeletionGroup.Done()
			err := cc.deleteJobPod(job.Name, pod)
            // 错误处理逻辑略
		}(pod)
	}
	waitDeletionGroup.Wait()

	// 更新Job状态
	newJob, err := cc.vcClient.BatchV1alpha1().Jobs(job.Namespace).UpdateStatus(context.TODO(), job, metav1.UpdateOptions{})
    // 更新缓存
	if e := cc.cache.Update(newJob); e != nil {
		klog.Errorf("SyncJob - Failed to update Job %v/%v in cache:  %v",
			newJob.Namespace, newJob.Name, e)
		return e
	}

	return nil
}
```



###### `KillJob`

从前面的表格中可以看出`KillJob`主要是删除Job或者异常场景触发的，Job并不支持升级操作，只支持扩缩容，因此一旦遇到异常场景会直接触发`KillJob`，其主要的代码逻辑为：

- 删除这个job对应的所有的pod，同时统计各个状态的pod的数量
- 更新Job的状态
- 删除Job对应的PodGroup

```go
// pkg/controllers/job/job_controller_actions.go
func (cc *jobcontroller) killJob(jobInfo *apis.JobInfo, podRetainPhase state.PhaseMap, updateStatus state.UpdateStatusFn) error {
	job := jobInfo.Job
	if job.DeletionTimestamp != nil {
		// 不处理正在删除中的job（可能有其他协程的worker在处理）
		return nil
	}
	var pending, running, terminating, succeeded, failed, unknown int32
	var errs []error
	var total int
    // 删除该job下的所有pod
	for _, pods := range jobInfo.Pods {
		for _, pod := range pods {
			total++
			if pod.DeletionTimestamp != nil {
				// 跳过已经在删除中的pod
				terminating++
				continue
			}
			_, retain := podRetainPhase[pod.Status.Phase]
			if !retain {
                // 调用kube-apiserver的接口删除状态不是Success或者Failed的pod
				err := cc.deleteJobPod(job.Name, pod)
				if err == nil {
					terminating++
					continue
				}
				errs = append(errs, err)
				cc.resyncTask(pod)
			}

			classifyAndAddUpPodBaseOnPhase(pod, &pending, &running, &succeeded, &failed, &unknown)
		}
	}
	// 错误处理略

	// 更新Job状态，将各种状态的pod数量更新到status中
	newJob, err := cc.vcClient.BatchV1alpha1().Jobs(job.Namespace).UpdateStatus(context.TODO(), job, metav1.UpdateOptions{})
	// 错误处理略
    // 更新job缓存
	if e := cc.cache.Update(newJob); e != nil {
		klog.Errorf("KillJob - Failed to update Job %v/%v in cache:  %v",
			newJob.Namespace, newJob.Name, e)
		return e
	}
	// 删除这个Job对应的PodGroups
	if err := cc.vcClient.SchedulingV1beta1().PodGroups(job.Namespace).Delete(context.TODO(), job.Name, metav1.DeleteOptions{}); err != nil {
		// 错误处理略
	}
	if err := cc.pluginOnJobDelete(job); err != nil {
		return err
	}
	return nil
}
```



注：job的garbage collector也可以在这里说一下。



###### 总结

通过前面的代码分析可以看到Job Controller主要的逻辑就是根据创建的Job创建对应的PodGroup以及Pod，并且维持Job的各种状态的状态机转移流程。

#### 2.2、PodGroup

##### 2.2.1、数据结构



```go
// PodGroup is a collection of Pod; used for batch workload.
type PodGroup struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	Spec PodGroupSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status PodGroupStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

// PodGroupSpec represents the template of a pod group.
type PodGroupSpec struct {
	// MinMember defines the minimal number of members/tasks to run the pod group;
	// if there's not enough resources to start all tasks, the scheduler
	// will not start anyone.
	MinMember int32 `json:"minMember,omitempty" protobuf:"bytes,1,opt,name=minMember"`

	// Queue defines the queue to allocate resource for PodGroup; if queue does not exist,
	// the PodGroup will not be scheduled. Defaults to `default` Queue with the lowest weight.
	// +optional
	Queue string `json:"queue,omitempty" protobuf:"bytes,2,opt,name=queue"`

	// If specified, indicates the PodGroup's priority. "system-node-critical" and
	// "system-cluster-critical" are two special keywords which indicate the
	// highest priorities with the former being the highest priority. Any other
	// name must be defined by creating a PriorityClass object with that name.
	// If not specified, the PodGroup priority will be default or zero if there is no
	// default.
	// +optional
	PriorityClassName string `json:"priorityClassName,omitempty" protobuf:"bytes,3,opt,name=priorityClassName"`

	// MinResources defines the minimal resource of members/tasks to run the pod group;
	// if there's not enough resources to start all tasks, the scheduler
	// will not start anyone.
	MinResources *v1.ResourceList `json:"minResources,omitempty" protobuf:"bytes,4,opt,name=minResources"`
}

// PodGroupStatus represents the current state of a pod group.
type PodGroupStatus struct {
	// Current phase of PodGroup.
	Phase PodGroupPhase `json:"phase,omitempty" protobuf:"bytes,1,opt,name=phase"`

	// The conditions of PodGroup.
	// +optional
	Conditions []PodGroupCondition `json:"conditions,omitempty" protobuf:"bytes,2,opt,name=conditions"`

	// The number of actively running pods.
	// +optional
	Running int32 `json:"running,omitempty" protobuf:"bytes,3,opt,name=running"`

	// The number of pods which reached phase Succeeded.
	// +optional
	Succeeded int32 `json:"succeeded,omitempty" protobuf:"bytes,4,opt,name=succeeded"`

	// The number of pods which reached phase Failed.
	// +optional
	Failed int32 `json:"failed,omitempty" protobuf:"bytes,5,opt,name=failed"`
}
```



##### 2.2.2、代码逻辑

在`Job`的控制器中我们看到`PodGroup`是由`Job`创建的，并且name还和`Job`一毛一样，然后在`KillJob`的时候就会把它直接删了，似乎并没有看到这个`PodGroup`的功能。实际上这是因为`PodGroup`的功能相对而言比较简单，主要就是给**不是通过Job创建的**pod创建对应的`PodGroup`。在`PodGroup`的controller中，分别list了`Pod`和`PodGroup`资源，但是只watch了pod的add事件，并且只过滤了没有对应PodGroup的pod，也就是说只有单独创建pod的时候才会加入到消费者队列中：

```go
    // pkg/controllers/podgroup/pg_controller.go
    pg.podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch v := obj.(type) {
				case *v1.Pod:
                    // 注意只监听没有对应PodGroup的pod
					if v.Spec.SchedulerName == opt.SchedulerName &&
						(v.Annotations == nil || v.Annotations[scheduling.KubeGroupNameAnnotationKey] == "") {
						return true
					}
					return false
				default:
					return false
				}
			},
            // 从EventHandler中可以看到这里只watch pod的add事件
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc: pg.addPod,
			},
		})
```

在消费者队列中，controller的处理逻辑非常简单，仅对add的新pod进行处理，

```go
// pkg/controllers/podgroup/pg_controller.go
func (pg *pgcontroller) processNextReq() bool {
	obj, shutdown := pg.queue.Get()
	if shutdown {
		klog.Errorf("Fail to pop item from queue")
		return false
	}

	req := obj.(podRequest)
	defer pg.queue.Done(req)
    // 从list缓存中获取pod信息
	pod, err := pg.podLister.Pods(req.podNamespace).Get(req.podName)
	if err != nil {
		klog.Errorf("Failed to get pod by <%v> from cache: %v", req, err)
		return true
	}

	// 对于没有创建对应PodGroup的pod创建其PodGroup，并将PodGroup的name写到pod的annotation中
	if err := pg.createNormalPodPGIfNotExist(pod); err != nil {
		klog.Errorf("Failed to handle Pod <%s/%s>: %v", pod.Namespace, pod.Name, err)
		pg.queue.AddRateLimited(req)
		return true
	}

	// If no error, forget it.
	pg.queue.Forget(req)

	return true
}
```

有一点需要注意的是，在创建`PodGroup`的时候，如果pod中已经有`QueueName`的annotation了，则将`QueueName`写到`PodGroup`的Spec中，如果没有则空着。不过admission-controller会给空的Job中加上默认的QueueName，因此通过Job创建的PodGroup会打上默认的QueueName，但是通过Pod创建出来的似乎并没有。

```go
// pkg/controllers/podgroup/pg_controller_handler.go
func (pg *pgcontroller) createNormalPodPGIfNotExist(pod *v1.Pod) error {
	pgName := helpers.GeneratePodgroupName(pod)
    // PodGroup不存在时创建
	if _, err := pg.pgLister.PodGroups(pod.Namespace).Get(pgName); err != nil {
		// 错误处理略

		obj := &scheduling.PodGroup{
			ObjectMeta: metav1.ObjectMeta{
				Namespace:       pod.Namespace,
				Name:            pgName,
				OwnerReferences: newPGOwnerReferences(pod),
			},
			Spec: scheduling.PodGroupSpec{
				MinMember:         1,
				PriorityClassName: pod.Spec.PriorityClassName,
			},
		}
        // 如果pod中已经有`QueueName`的annotation了，则将`QueueName`写到`PodGroup`的Spec中
		if queueName, ok := pod.Annotations[scheduling.QueueNameAnnotationKey]; ok {
			obj.Spec.Queue = queueName
		}
        // 创建PodGroup
		if _, err := pg.vcClient.SchedulingV1beta1().PodGroups(pod.Namespace).Create(context.TODO(), obj, metav1.CreateOptions{}); err != nil {
			klog.Errorf("Failed to create normal PodGroup for Pod <%s/%s>: %v",
				pod.Namespace, pod.Name, err)
			return err
		}
	}

	return pg.updatePodAnnotations(pod, pgName)
}
```



#### 2.3、Queue

##### 2.3.1、数据结构

在volcano中，Queue是一个nonNamespaced资源，它是一个PodGroup的队列



##### 2.3.2、代码逻辑

queue-controller中主要list-watch的是`Queue`和`PodGroup`资源，其中有两个成员比较关键，一个是用来保存事件队列的`queue`，另一个是用来标识所有`PodGroup`的`podGroups`。

```go
// queuecontroller manages queue status.
type queuecontroller struct {
    // 其他成员略

	// watch到的事件队列，队列中的元素是2.1.2节中已经说过的Request
	queue     workqueue.RateLimitingInterface
	// queue name -> podgroup namespace/name
    // 标识所有watch到的podgroup
	podGroups map[string]map[string]struct{}
}
```

当watch到新的`Queue`或者`PodGroup`资源时，都是将事件组装成带有`QueueName`的`Request`加入队列中，需要注意的是，无论是job-controller中还是podgroup-controller中创建的`PodGroup`都是带有podgroup.Spec.Queue的。

```go
// pkg/controllers/queue/queue_controller_handler.go
func (c *queuecontroller) addQueue(obj interface{}) {
	queue := obj.(*schedulingv1beta1.Queue)

	req := &apis.Request{
		QueueName: queue.Name,
		Event:  busv1alpha1.OutOfSyncEvent,
		Action: busv1alpha1.SyncQueueAction,
	}

	c.enqueue(req)
}

func (c *queuecontroller) addPodGroup(obj interface{}) {
	pg := obj.(*schedulingv1beta1.PodGroup)
	key, _ := cache.MetaNamespaceKeyFunc(obj)

	c.pgMutex.Lock()
	defer c.pgMutex.Unlock()

	if c.podGroups[pg.Spec.Queue] == nil {
		c.podGroups[pg.Spec.Queue] = make(map[string]struct{})
	}
    // hashset，用来标识这个podgroup
	c.podGroups[pg.Spec.Queue][key] = struct{}{}

	req := &apis.Request{
		QueueName: pg.Spec.Queue,
		Event:  busv1alpha1.OutOfSyncEvent,
		Action: busv1alpha1.SyncQueueAction,
	}

	c.enqueue(req)
}
```

在消费者端，queue-controller中的逻辑跟job-controller有点类似，先根据`Queue`的状态生成处理函数，然后根据req中的Action来进行处理。

```go
func (c *queuecontroller) handleQueue(req *apis.Request) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing queue %s (%v).", req.QueueName, time.Since(startTime))
	}()
    // 从informer缓存中获取Queue信息
	queue, err := c.queueLister.Get(req.QueueName)
	// 错误处理略

    // 根据queue的状态生成处理函数
	queueState := queuestate.NewState(queue)
	// 错误处理略
    
    // 执行queue处理函数，根据Action和State来执行
	if err := queueState.Execute(req.Action); err != nil {
		return fmt.Errorf("sync queue %s failed for %v, event is %v, action is %s",
			req.QueueName, err, req.Event, req.Action)
	}
	return nil
}
```

类似的，queue-controller也有自己的处理函数转移表

| Action\State | Open       | Closed    | Closing   | Unknown    |
| ------------ | ---------- | --------- | --------- | ---------- |
| OpenQueue    | SyncQueue  | OpenQueue | OpenQueue | OpenQueue  |
| CloseQueue   | CloseQueue | SyncQueue | SyncQueue | CloseQueue |
| SyncQueue    | SyncQueue  | SyncQueue | SyncQueue | SyncQueue  |

可以看到主要的处理仅包含OpenQueue、SyncQueue和CloseQueue三种，这三种的处理逻辑比较简单，这里就不贴代码直接描述一遍：

- `OpenQueue`：进入到`OpenQueue`的Queue状态是Closed、Closing和Unknown中的一种，因此`OpenQueue`做的仅仅是把Queue的状态更新成Open
- `SyncQueue`：主要对该`Queue`中所有的PodGroup状态进行统计并更新到queueStatus中
- `CloseQueue`：跟`OpenQueue`类似，把Queue的状态更新成Closed



### 3、scheduler

scheduler是volcano的核心，它是以`PodGroup`为基本单位来进行调度的。

> 在设计之初我们就把 job和podgroup两个概念分开。所有跟作业相关的信息，都是放在 job里面；所有跟调度相关的信息都放在podgroup里面，这个设计与Kubernetes非常相像。

scheduler的架构可以参考下图：

![img](2020-08-04-volcano-code-review/scheduler.png)

调度器的本质还是给所有没有绑定到节点上的pod找到合适的节点并绑定上去，但是为了实现gang调度、抢占、资源预留等功能，不能跟k8s的调度器一样通过watch到的pod事件来触发调度（大多数情况下，每一个pod的调度都是单pod最优），所以volcano的调度器采用的是周期性全局调度的方式。我们在看volcano的调度器代码时也能够看到调度逻辑也是这样的思路：

- list-watch的是PodGroup和Node
- 周期性创建一个全局调度的Session，对集群做一次快照
- 在每一个Session中，根据配置的调度算法和策略对快照中的所有PodGroup进行调度

```go
// Scheduler watches for new unscheduled pods for volcano. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
type Scheduler struct {
	cache          schedcache.Cache   // Cache是一个接口，实现了这个接口的是SchedulerCache，所有调度资源的缓存
	actions        []framework.Action // 调度器的配置文件中设置的每一轮调度的处理逻辑，有enqueue, allocate, backfill等
	plugins        []conf.Tier        // 调度器的配置文件中设置的调度算法的各个层级的算法集合
	configurations []conf.Configuration  // 调度器每个Action对应的参数
	schedulerConf  string             // 调度器配置文件路径
	schedulePeriod time.Duration      // 调度器的调度周期，默认1s
}
```

一个默认的配置文件如下：

```yaml
actions: "enqueue, allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
- plugins:
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder
```

可以看到其中Action的顺序是`enqueue`、`allocate`和`backfill`，调度器分成两层，一层是priority和gang调度，另一层是drf、predicates、proportion和nodeorder调度。

```go
// pkg/scheduler/scheduler.go
// Run runs the Scheduler
func (pc *Scheduler) Run(stopCh <-chan struct{}) {
	// list-watch调度器所需要的资源
	go pc.cache.Run(stopCh)
	pc.cache.WaitForCacheSync(stopCh)
    // 周期性执行runOnce
	go wait.Until(pc.runOnce, pc.schedulePeriod, stopCh)
}

func (pc *Scheduler) runOnce() {
	// metric代码略
    
    // 注意每次进行调度的时候都会load一次配置文件（目的是为了热更新configmap？）
	pc.loadSchedulerConf()
    // 创建一个新的Session，将cache、配置等复制到其中，并为每一个调度器注册其对应的调度算法函数
	ssn := framework.OpenSession(pc.cache, pc.plugins, pc.configurations)
	defer framework.CloseSession(ssn)
    // 根据配置文件中的action顺序执行调度操作
	for _, action := range pc.actions {
		actionStartTime := time.Now()
		action.Execute(ssn)
		metrics.UpdateActionDuration(action.Name(), metrics.Duration(actionStartTime))
	}
}
```

各个调度器所注册的调度算法如下表所示，“*”表示该调度器注册了该算法。如果要新增一个调度器，那么就只需要去实现对应过程的function并注册进去就可以了。

| function\plugin  | binpack | conformance | drf  | gang | nodeorder | predicate | priority | proportion |
| ---------------- | ------- | ----------- | ---- | ---- | --------- | --------- | -------- | ---------- |
| jobOrderFn       |         |             | *    | *    |           |           | *        |            |
| queueOrderFn     |         |             |      |      |           |           |          | *          |
| taskOrderFn      |         |             |      |      |           |           | *        |            |
| namespaceOrderFn |         |             | *    |      |           |           |          |            |
| predicateFn      |         |             |      |      |           | *         |          |            |
| nodeOrderFn      | *       |             |      |      | *         |           |          |            |
| batchNodeOrderFn |         |             |      |      | *         |           |          |            |
| preemptableFn    |         | *           | *    | *    |           |           | *        |            |
| reclaimableFn    |         | *           |      | *    |           |           |          | *          |
| overusedFn       |         |             |      |      |           |           |          | *          |
| jobReadyFn       |         |             |      | *    |           |           |          |            |
| jobPipelinedFn   |         |             |      | *    |           |           |          |            |
| jobValidFn       |         |             |      | *    |           |           |          |            |
| jobEnqueueableFn |         |             |      |      |           |           |          | *          |

在volcano的调度器中，当前实现的Action有五个：

- enqueue（入队）：入队主要就是过滤出需要处理的Job，先通过`QueueOrderFn`根据优先级将所有要处理的Queue加入到一个队列中，同时每一个Queue上的Job也通过`JobOrderFn`根据优先级将所有要处理的Job加入到这个Queue的队列中，然后根据Queue和Job的优先级来对每一个Job进行`jobEnqueueableFn`预判断（当前资源是否满足Job的需求）

- allocate（分配）：分配其实就是给每一个Task绑定节点，是调度的核心，其处理逻辑主要分以下6步，主要逻辑就是每次选择一个优先级最高的Task，并找到打分最高的节点bind过去，直到所有的Task都处理完
  - 通过`NamespaceOrderFn`根据优先级选择一个需要去处理的namespace
  - 通过`QueueOrderFn`根据优先级选择一个需要去处理的Queue
  - 通过`JobOrderFn`根据优先级选择一个需要去处理的Job
  - 通过`TaskOrderFn`根据优先级选择一个需要去处理的Task
  - 通过`predicateFn`过滤去除不满足要求的节点
  - 通过`NodeOrderFn`来给节点进行打分，并将分数最高的节点bind给这个Task
- backfill（回填）：volcano中为了避免饥饿而有条件地为大作业保留了一些资源，回填是对剩下来未调度小Task进行bind的过程，对于每一个未调度的Task：
  - 遍历所有节点，通过`predicateFn`滤除不满足要求的node
  - 尝试将该Task调度到满足要求的节点上

- [preempt](https://volcano.sh/docs/scheduler-preempt/)（抢占）：抢占是一种特殊的Action，它主要处理的场景是当一个高优先级的Task进入调度器但是当前环境中的资源已经无法满足这个Task的时候，需要能将已经调度的任务中驱逐一部分优先级低的Task，以便这个高优先级的Task能够正常运行，因此其处理过程包含选择优先级低的Task并驱逐的逻辑。其处理流程为，对于PodGroup状态不为Pending的Job
  - 通过`jobValidFn`和`jobPipelinedFn`进行过滤
  - 通过`JobOrderFn`和`TaskOrderFn`对集群中的Job和Task进行优先级队列的初始化
  - 对于每一个需要进行抢占调度的Task：
    - 通过`predicateFn`对所有节点进行过滤，通过`batchNodeOrderFn`、`nodeOrderFn`、`nodeReduceFn`对所有节点进行打分和排序
    - 按照分数排序对每个节点上的Task调用`preemptableFns`判断该Task是否可以抢占（也就是这个Task是否可以驱逐用来腾出资源给待调度的Task），指导找到节点并且可以驱逐的Task腾出来的资源满足待调度的Task为止
    - 对于抢占而言，该Action中同时考虑了跨Queue和Queue内部跨Job之间的抢占

- [reclaim](https://volcano.sh/docs/scheduler-reclaim/)（回收）：在volcano中，集群的资源是根据权重给每一个Queue分配的，当有一个新的Queue创建出来时，第一个Job的Task进行资源调度的时候就会触发回收，也就是对之前创建的Queue中的Task进行驱逐，腾出对应比例的资源给这个新Queue。其处理流程为：
  - 通过`queueOrderFn`对当前集群中的Queue进行优先级排序
  - 通过`JobOrderFn`和`TaskOrderFn`对集群中的Job和Task进行优先级队列的初始化
  - 通过`overusedFn`过滤掉超配额的Queue
  - 对于每一个Task，通过`reclaimableFn`来判断是否需要触发回收
  - 对于每一个需要触发回收的Task，执行驱逐操作（其实就是把要驱逐的Pod删掉）

通过以上的归纳其实也可以得到function和action之间的关系（表格中的数字表示调用顺序）：

| function\action  | enqueue | allocate | backfill | preempt | reclaim |
| ---------------- | ------- | -------- | -------- | ------- | ------- |
| jobOrderFn       | 2       | 3        |          | 3       | 2       |
| queueOrderFn     | 1       | 2        |          |         | 1       |
| taskOrderFn      |         | 4        |          | 4       | 3       |
| namespaceOrderFn |         | 1        |          |         |         |
| predicateFn      |         | 5        | 1        | 5       |         |
| nodeOrderFn      |         | 6        |          | 7       |         |
| batchNodeOrderFn |         |          |          | 6       |         |
| preemptableFn    |         |          |          | 8       |         |
| reclaimableFn    |         |          |          |         | 5       |
| overusedFn       |         |          |          |         | 4       |
| jobReadyFn       |         |          |          |         |         |
| jobPipelinedFn   |         |          |          | 2       |         |
| jobValidFn       |         |          |          | 1       |         |
| jobEnqueueableFn | 3       |          |          |         |         |



### 参考文章


