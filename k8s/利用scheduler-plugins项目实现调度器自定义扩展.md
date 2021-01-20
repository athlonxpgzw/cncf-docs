# 利用scheduler-plugins项目实现调度器自定义扩展

> > 代码范例参考commit 62333d4bc268e7f0509cb7d10efea16009ba255b

Kubernetes 负责 Kube-scheduler 的小组 sig-scheduling 为了更好的管理调度相关的 Plugin，新建了项目 scheduler-plugins 来方便用户管理不同的插件，用户可以直接基于这个项目来定义自己的out-of-tree(树外)插件。

项目地址是[https://github.com/kubernetes-sigs/scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)。

## 获取项目

项目有gcr仓库镜像和源码编译两种获取方式

### 官方镜像仓库

从v0.18.9版本开始，scheduler-plugins将镜像发布至k8s官方镜像仓库k8s.gcr.io，地址是k8s.gcr.io/scheduler-plugins/kube-scheduler。可以通过以下命令获取镜像(需要爬梯)。

``` shell
docker pull k8s.gcr.io/scheduler-plugins/kube-scheduler:$TAG
```

在将来的版本中，scheduler-plugins的官方容器镜像由controller提供

``` shell
docker pull k8s.gcr.io/scheduler-plugins/controller:$TAG
```

### 源码获取

项目源码有两个地址，一个是https://sigs.k8s.io/scheduler-plugins，另一个是https://github.com/kubernetes-sigs/scheduler-plugins，前者是官方地址需要爬梯，后者是托管在github的地址。

将源码cloning到‘$GOPATH/src/sigs.k8s.io'目录。

``` shell
git clone https://sigs.k8s.io/scheduler-plugins
```

或着

``` shell
go get -d -u sigs.k8s.io/scheduler-plugins
```



## Build

项目构建有两种方式，一种是镜像构建，另一种是binary构建。

### 镜像构建(需要构建环境能访问google)

``` shell
make local-image
```

完成后在本地镜像registry里会生成以下两个镜像

* localhost:5000/scheduler-plugins/kube-scheduler:latest
* localhost:5000/scheduler-plugins/controller:latest

如果修改了源码中interface的相关定义，需要执行以下命令重新生成部分代码

``` shell
hack/update-codegen.sh
```

### 二进制构建

``` shell
make
```

如果修改了项目分支或者加入了新的依赖包，则需要重新生成vendor目录

``` shell
make autogen
```



## Debug

默认情况下debug信息是剥离掉的。如果在程序中保留的话，需要在Makefile中删掉ldflags参数中的-w。

单元测试

``` shell
make unit-test
```

集成测试

``` shell
make integration-test
```



## 简单使用

### 容器方式运行scheduler

apply如下deployment定义将生成一个schedulingplugin的pod。

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: schedulingplugin
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: scheduler
      tier: control-plane
  template:
    metadata:
      labels:
        component: scheduler
        tier: control-plane
    spec:
      nodeSelector:
        node-role.kubernetes.io/master: ""
      containers:
        - image: localhost:5000/scheduler-plugins/kube-scheduler:latest
          imagePullPolicy: Never
          command:
          - /bin/kube-scheduler
          - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
          - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
          - --config=/etc/kubernetes/configs/scheduler-config.yaml
          - -v=9
          name: schedulingplugin
          securityContext:
            privileged: true
          volumeMounts:
          - mountPath: /etc/kubernetes
            name: etckubernetes
      hostNetwork: false
      hostPID: false
      volumes:
      - hostPath:
          path: /etc/kubernetes/
          type: Directory
        name: etckubernetes
```

注意修改/etc/kubernetes/configs/scheduler-config.yaml，配置plugin和pluginConfig。

``` yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
clientConnection:
  kubeconfig: "REPLACE_ME_WITH_KUBE_CONFIG_PATH"
profiles:
- schedulerName: default-scheduler
  plugins:
    preFilter:
      enabled:
      - name: CapacityScheduling
    postFilter:
      enabled:
      - name: CapacityScheduling
      disabled:
      - name: "*"
    reserve:
      enabled:
      - name: CapacityScheduling
  pluginConfig:
  - name: CapacityScheduling
    args:
      kubeConfigPath: "REPLACE_ME_WITH_KUBE_CONFIG_PATH"
```

### 二进制方式运行

执行类似如下命令运行scheduler，或者将相关参数配置到systemd，由systemd接管服务。

``` shell
~/go/src/sigs.k8s.io/scheduler-plugins/bin/kube-scheduler --config /root/k8s/scheduling/tmp/scheduler-config.yaml --bind-address=10.181.103.18 --secure-port=10259 --port=0 --tls-cert-file=/etc/kubernetes/cert/kube-scheduler.pem --tls-private-key-file=/etc/kubernetes/cert/kube-scheduler-key.pem --authentication-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig --client-ca-file=/etc/kubernetes/cert/ca.pem --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --authorization-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig --logtostderr=true --v=9
```



## 代码参考

接下来以其中的 Qos 的插件来为例，演示如何开发自己的插件。

自定义插件需要两个步骤：

1. 实现插件的接口
2. 注册插件并配置插件

### 实现插件的接口

framework的接口有很多，对应不同的扩展点。只需插件实现了对应interface的鸭子类型，即可实现相应的扩展点功能。这里我们实现 QueueSort 扩展点，先看看 QueueSort 扩展点定义的接口：

``` go
// QueueSortPlugin is an interface that must be implemented by "QueueSort" plugins.
// These plugins are used to sort pods in the scheduling queue. Only one queue sort
// plugin may be enabled at a time.
type QueueSortPlugin interface {
	Plugin
	// Less are used to sort pods in the scheduling queue.
	Less(*QueuedPodInfo, *QueuedPodInfo) bool
}
```

默认的调度器会优先调度优先级较高的 Pod , 其具体实现的方式是使用 QueueSort 这个插件，默认的实现，是对 Pod 的 Priority 值进行排序，但当优先级相同时，再比较 Pod 的 timestamp ,  timestamp 是 Pod 加入 queue 的时间。我们现在想先根据 Pod 的 Priority 值进行排序，当 Priority 值相同，再根据 Pod 的 QoS 类型进行排序，最后再根据 Pod 的 timestamp 排序。每个调度器中的QueueSort插件只能有一个,用来实现SchedulingQueue。SchedulingQueue中有activeQ,podBackoffQ两个heap，QueueSort的Less函数即用来实现这个两heap排序的lessFunc:

``` go
// data is an internal struct that implements the standard heap interface
// and keeps the data stored in the heap.
type data struct {
	// items is a map from key of the objects to the objects and their index.
	// We depend on the property that items in the map are in the queue and vice versa.
	items map[string]*heapItem
	// queue implements a heap data structure and keeps the order of elements
	// according to the heap invariant. The queue keeps the keys of objects stored
	// in "items".
	queue []string

	// keyFunc is used to make the key used for queued item insertion and retrieval, and
	// should be deterministic.
	keyFunc KeyFunc
	// lessFunc is used to compare two objects in the heap.
	lessFunc lessFunc
}
```



#### Qos 类型如下：

1. Guaranteed : resources limits 和 requests 相等
2. Burstable : resources limits 和 requests 不相等
3. BestEffort : 未设置 resources limits 和 requests

Qos 的插件主要基于 Pod 的 QoS(Quality of Service) class 来实现的，调度顺序是: 1. Guaranteed (requests == limits) 2. Burstable (requests < limits) 3. BestEffort (requests and limits not set)

### 插件的实现

首先插件要定义插件的对象和构造函数：

``` go
// Name is the name of the plugin used in the plugin registry and configurations.
const Name = "QOSSort"

// Sort is a plugin that implements QoS class based sorting.
type Sort struct{}

// Name returns name of the plugin.
func (pl *Sort) Name() string {
	return Name
}

// New initializes a new plugin and returns it.
func New(_ runtime.Object, _ framework.FrameworkHandle) (framework.Plugin, error) {
	return &Sort{}, nil
}
```

重新定义了 Less 函数，在优先级相同的情况下，通过比较 Qos 来决定优先级。

``` go
var _ framework.QueueSortPlugin = &Sort{}

// Less is the function used by the activeQ heap algorithm to sort pods.
// It sorts pods based on their priorities. When the priorities are equal, it uses
// the Pod QoS classes to break the tie.
func (*Sort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
	p1 := pod.GetPodPriority(pInfo1.Pod)
	p2 := pod.GetPodPriority(pInfo2.Pod)
	return (p1 > p2) || (p1 == p2 && compQOS(pInfo1.Pod, pInfo2.Pod))
}

func compQOS(p1, p2 *v1.Pod) bool {
	p1QOS, p2QOS := v1qos.GetPodQOS(p1), v1qos.GetPodQOS(p2)
	if p1QOS == v1.PodQOSGuaranteed {
		return true
	}
	if p1QOS == v1.PodQOSBurstable {
		return p2QOS != v1.PodQOSGuaranteed
	}
	return p2QOS == v1.PodQOSBestEffort
}
```

### 插件的注册

``` go
func main() {
	rand.Seed(time.Now().UnixNano())

	// Register custom plugins to the scheduler framework.
	// Later they can consist of scheduler profile(s) and hence
	// used by various kinds of workloads.
	command := app.NewSchedulerCommand(
		app.WithPlugin(qos.Name, qos.New),
	)

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

### 插件的配置

通过配置让 Sheduler 知道哪些插件需要被初始化，如下面指定了 QueueSort 的插件 Qos，其他的扩展点的插件没有被指定，则都会 Kube-Scheduler 的默认的实现。可以看到，schedulerName 字段代表扩展的调度器名称， plugins 字段中各个扩展点的插件名称，enable 代表该扩展点关于运行你的插件。

``` yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
clientConnection:
  kubeconfig: "REPLACE_ME_WITH_KUBE_CONFIG_PATH"
profiles:
- schedulerName: default-scheduler
  plugins:
    queueSort:
      enabled:
      - name: QOSSort
      disabled:
      - name: "*"
```

实现完成后即可根据需求构建scheduler并运行。

## 延伸

scheduler-plugins项目中除了QoS插件之外还实现了capacityscheduling, coscheduling, noderesources-callocatable, crossnodepreemption, podstate等插件

``` go
	// Register custom plugins to the scheduler framework.
	// Later they can consist of scheduler profile(s) and hence
	// used by various kinds of workloads.
	command := app.NewSchedulerCommand(
		app.WithPlugin(capacityscheduling.Name, capacityscheduling.New),
		app.WithPlugin(coscheduling.Name, coscheduling.New),
		app.WithPlugin(noderesources.AllocatableName, noderesources.NewAllocatable),
		// Sample plugins below.
		app.WithPlugin(crossnodepreemption.Name, crossnodepreemption.New),
		app.WithPlugin(podstate.Name, podstate.New),
		app.WithPlugin(qos.Name, qos.New),
	)
```

具体的KEPs详见https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/kep



