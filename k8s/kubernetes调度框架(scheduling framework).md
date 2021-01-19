# kubernetes调度框架(scheduling framework)

> 文档代码部分参考Kubernetes V1.19.7  commit 1be54aaa7077826dce2f174c14821f07703445ce

Kube-Scheduler 作为 Kubernetes 的核心组件之一，主要负责整个集群资源的调度功能，根据特定的调度算法和策略，将 Pod 调度到最优的工作节点，从而更加合理、充分地利用集群资源。

但是随着 Kubernetes 部署的任务类型越来越多，原生 Kube-Scheduler 已经不能应对多样的调度需求：比如机器学习、深度学习训练任务对于协同调度功能的需求；高性能计算作业，基因计算工作流对于一些动态资源 GPU、网络、存储卷的动态资源绑定需求等。因此自定义 Kubernetes 调度器的需求愈发迫切，本文讨论了扩展 Kubernetes 调度程序的各种方法，然后使用目前最佳的扩展方式 Scheduling Framework，演示如何扩展 Scheduler。

## 自定义调度器的主要方式

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/FAAWDPLTI7SHtxxQB48tiaO5USFjWMLHJX6sunp8pd7jbFHCV7BwVjXoxQZFWhia34NLKlcKfgYYyiciaRibJytER2w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## scheduling framework

为了使 Kube-scheduler 扩展性更好、代码更简洁，社区从 Kubernetes 1.16 版本开始, 构建了一种新的调度框架 Kubernetes Scheduling Framework 的机制。

Scheduling Framework是一个嵌入到k8s调度器的可拔插架构，为方便定制化调度策略而设计。在原有的调度流程中, 定义了丰富扩展点接口(Plugin APIs)，开发者可以通过实现扩展点所定义的接口来实现插件，将插件注册到扩展点。Scheduling Framework 在执行调度流程时，运行到相应的扩展点时，会调用用户注册的插件，影响调度决策的结果。通过这种方式来将用户的调度逻辑集成到 Scheduling Framework 中。同时保持了调度器核心的简洁和可维护性。这个[链接](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md)提供了scheduling framework的技术设计议案。



## framework工作流程

每个pod的调度过程分为两个阶段：调度周期(scheduling cycle)和绑定周期(bind cycle)。



### 调度周期和绑定周期

调度周期为 Pod 选择一个节点，绑定周期则调用 Apiserver，用选好的 Node，更新 Pod 的 spec.nodeName 字段。调度周期和绑定周期统称为 “Scheduling Context” (调度上下文)。调度周期是串行运行的，同一时间只能有一个 Pod 被调度，是线程安全的；而绑定周期因为需要访问 Apiserver 的接口，耗时较长，为了提高调度的效率，需要异步执行，即同一时间，可执行多个 bind 操作，是非线程安全的。

如果 Pod 被确定为不可调度或存在内部错误，那么调度周期或绑定周期将被中止。Pod 将返回队列并等待下一次重试。如果一个绑定周期被终止，它将触发 Reserve 插件中的 UnReserve 方法。

### 扩展点

如下图所示，调度框架提供了丰富的扩展点，在这幅图中，Filter 相当于之前 Predicate 预选模块, Score 相当于 Priority 优选模块，每一个扩展点模块都提供了接口，我们可以实现扩展点定义的接口来实现自己的调度逻辑，并将实现的插件注册到扩展点。

Scheduling Framework 在执行调度流程时，当运行到扩展点时，会调用我们注册的插件，通过执行自定义插件的策略，满足调度需求。此外，一个插件可以在多个扩展点注册，用以执行更复杂或有状态的任务。

![img](https://d33wubrfki0l68.cloudfront.net/4e9fa4651df31b7810c851b142c793776509e046/61a36/images/docs/scheduling-framework-extensions.png)

以下是调度器核心函数scheduleOne,包括了整个调度周期个绑定周期的实现。sched.Algorithm.Schedule函数执行了调度周期中的PreFilter,Filter,PreScore,Score,Normalize Score这５个扩展点．prof.RunReservePluginsReserve函数执行Reserve扩展点．prof.RunPermitPlugins函数执行Permit扩展点．然后开goroutine执行绑定周期的扩展点．

```go
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
	
	pod := podInfo.Pod
	prof, err := sched.profileForPod(pod)
	
	schedulingCycleCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
	.
	.
	.
	

	// Run the Reserve method of reserve plugins.
	if sts := prof.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		metrics.PodScheduleError(prof.Name, metrics.SinceInSeconds(start))
		// trigger un-reserve to clean up state associated with the reserved Pod
		prof.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(prof, assumedPodInfo, sts.AsError(), SchedulerError, "")
		return
	}

	// Run "permit" plugins.
	runPermitStatus := prof.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		var reason string
		if runPermitStatus.IsUnschedulable() {
			metrics.PodUnschedulable(prof.Name, metrics.SinceInSeconds(start))
			reason = v1.PodReasonUnschedulable
		} else {
			metrics.PodScheduleError(prof.Name, metrics.SinceInSeconds(start))
			reason = SchedulerError
		}
		// One of the plugins returned status different than success or wait.
		prof.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(prof, assumedPodInfo, runPermitStatus.AsError(), reason, "")
		return
	}

	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()
		metrics.SchedulerGoroutines.WithLabelValues("binding").Inc()
		defer metrics.SchedulerGoroutines.WithLabelValues("binding").Dec()

		waitOnPermitStatus := prof.WaitOnPermit(bindingCycleCtx, assumedPod)
		.
        .
        .

		// Run "prebind" plugins.
		preBindStatus := prof.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		.
        .
        .

		err := sched.bind(bindingCycleCtx, prof, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			metrics.PodScheduleError(prof.Name, metrics.SinceInSeconds(start))
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			prof.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", err)
			}
			sched.recordSchedulingFailure(prof, assumedPodInfo, fmt.Errorf("Binding rejected: %v", err), SchedulerError, "")
		} else {
			// Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
			.
            .
            .

			// Run "postbind" plugins.
			prof.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}

```



#### QueueSort

用于给调度队列排序，默认情况下，所有的 Pod 都会被放到一个队列中，此扩展用于对 Pod 的待调度队列进行排序，以决定先调度哪个 Pod，QueueSort 扩展本质上只需要实现一个方法 Less(Pod1, Pod2) 用于比较两个 Pod 谁更优先获得调度，同一时间点只能有一个 QueueSort 插件生效。

####  PreFilter

PreFilter 在 scheduling cycle 开始时就被调用，只有当所有的 PreFilter 插件都返回 success 时，才能进入下一个阶段，否则 Pod 将会被拒绝掉，标识此次调度流程失败。PreFilter 类似于调度流程启动之前的预处理，可以对 Pod 的信息进行加工。同时 PreFilter 也可以进行一些预置条件的检查，去检查一些集群维度的条件，判断否满足 pod 的要求。

```go
// Filters the nodes to find the ones that fit the pod based on the framework
// filter plugins and filter extenders.
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.NodeToStatusMap, error) {
	filteredNodesStatuses := make(framework.NodeToStatusMap)

	// Run "prefilter" plugins.
	s := prof.RunPreFilterPlugins(ctx, state, pod)
	if !s.IsSuccess() {
		if !s.IsUnschedulable() {
			return nil, nil, s.AsError()
		}
		// All nodes will have the same status. Some non trivial refactoring is
		// needed to avoid this copy.
		allNodes, err := g.nodeInfoSnapshot.NodeInfos().List()
		if err != nil {
			return nil, nil, err
		}
		for _, n := range allNodes {
			filteredNodesStatuses[n.Node().Name] = s
		}
		return nil, filteredNodesStatuses, nil

	}
```



#### Filter

Filter 插件是 scheduler v1 版本中的 Predicate 的逻辑，用来过滤掉不满足 Pod 调度要求的节点。为了提升效率，Filter 的执行顺序可以被配置，这样用户就可以将可以过滤掉大量节点的 Filter 策略放到前边执行，从而减少后边 Filter 策略执行的次数，例如我们可以把 NodeSelector 的 Filter 放到第一个，从而过滤掉大量的节点。Node 节点执行 Filter 策略是并发执行的，所以在同一调度周期中多次调用过滤器。

```go
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, prof, state, pod, filteredNodesStatuses)
	if err != nil {
		return nil, nil, err
	}
```



#### PostFilter

新的 PostFilter 的接口定义在 1.19 的版本发布，主要是用于处理当 Pod 在 Filter 阶段失败后的操作，例如抢占，Autoscale 触发等行为。

#### PreScore

PreScore 在之前版本称为 PostFilter，现在修改为 PreScore，主要用于在 Score 之前进行一些信息生成。此处会获取到通过 Filter 阶段的节点列表，我们也可以在此处进行一些信息预处理或者生成一些日志或者监控信息。

```go
	// Run PreScore plugins.
	preScoreStatus := prof.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		return nil, preScoreStatus.AsError()
	}
```



#### Score

Scoring 扩展点是 scheduler v1 版本中 Priority 的逻辑，目的是为了基于 Filter 过滤后的剩余节点，根据 Scoring 扩展点定义的策略挑选出最优的节点。

```go
	// Run the Score plugins.
	scoresMap, scoreStatus := prof.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}
```



#### NormalizeScore

在 NormalizeScore 阶段，调度器将会把每个 Score 扩展对具体某个节点的评分结果和该扩展的权重合并起来，作为最终评分结果，评分结果是一个范围内的整数。如果 Score 或 NormalizeScore 返回错误，则调度周期将中止。

#### Reserve

此扩展点为 Pod 预留的在要运行节点上的资源，目的是避免调度器在等待 Pod 与节点绑定的过程中调度新的 Pod 到节点上时，发生实际使用资源超出可用资源的情况。（因为绑定 Pod 到节点上是异步发生的）。这是调度过程的最后一个步骤，Pod 进入 Reserved 状态以后，要么在绑定失败时，触发 Unreserve 扩展，要么在绑定成功时，由 PostBind 扩展结束绑定过程。

#### Permit

Permit 扩展点是 framework v2 版本引入的新功能，当 Pod 在 Reserve 阶段完成资源预留之后，Bind 操作之前，开发者可以定义自己的策略在 Permit 节点进行拦截，根据条件对经过此阶段的 Pod 进行 allow、reject 和 wait 的 3 种操作。

* allow 表示 pod 允许通过 Permit 阶段。
* reject 表示 pod 被 Permit 阶段拒绝，则 Pod 调度失败。
* wait 表示将 Pod 处于等待状态，开发者可以设置超时时间。

#### PreBind

扩展用于在 Pod 绑定之前执行某些逻辑。这个插件引入的原因，是有一些资源，是在不在调度 Pod 时，立即确定可用的节点的资源，所以调度程序需要确保，这些资源已经成功绑定到选定的节点后，才能将 Pod 调度到此节点。例如，PreBind 扩展可以将一个基于网络的数据卷挂载到节点上，以Pod 可以使用。如果任何一个 PreBind 扩展返回错误，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展。

#### Bind

Bind 扩展点是 scheduler v1 版本中的 Bind 操作，会调用 apiserver 提供的接口，将 pod 绑定到对应的节点上。

#### PostBind

PostBind 是一个信息扩展点。成功绑定 Pod 后，将调用 PostBind 插件，可用于清理关联的资源。

### 插件API

自定义插件需要两个步骤：

1. 实现插件的接口

2. 注册插件并配置插件

   

扩展点接口有如下声明：

```go
type Plugin interface {
    Name() string
}

type QueueSortPlugin interface {
    Plugin
    Less(*v1.pod, *v1.pod) bool
}

type PreFilterPlugin interface {
    Plugin
    PreFilter(context.Context, *framework.CycleState, *v1.pod) error
}

// ...
```

### 插件配置

通过配置让 Scheduler 知道那些插件需要被初始化,可以定义配置的扩展点，以及扩展点的对应算法，算法对应的权重等等．以下给出一个基本的KubeSchedulerConfiguration：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:
        disabled:
        - name: NodeResourcesLeastAllocated
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```