---
title: ConfigMap热更新
categories: Kubernetes
---

有一天，同事遇到了客户的奇怪的需求：主动触发pod的重建，但是又不能影响到pod中的业务。当时我就想到，给相关deployment配置滚动更新呀。但是，紧接着问题就来了，怎么触发这个滚动更新呢？接下来，就想到了ConfigMap的热更新可以触发Deploy的滚动升级呀，这样不就能将pod重建了吗？但是，同事测试后却发现ConfigMap更新后，但是缺没看到Deployment的滚动升级呢？这是为何呢？这就涉及到ConfigMap被挂载进pod的方式了。

## 以环境变量的方式挂载ConfigMap
当我们编写Deployment以环境变量的方式挂载ConfigMap的时候，有时候可能看到ValueFrom，有时候可能会使用EnvFrom。这是为啥呢？
主要是k8s中一开始都是使用的ValueFrom，一次只能配置一个变量，我们可能要写多个如下结构

```yaml
configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
  name: special-config
              # Specify the key associated with the value
  key: special.how
```
但是，在kubernetes1.6开始，引入了一个新字段envFrom，实现了在pod环境中将ConfigMap（或者Secret资源对象）中定义的key=value自动生成为环境变量。如下：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: special-config
  restartPolicy: Never
```
**测试后发现通过这种方式挂载Config Map，更新Config Map并不会触发Deployment更新。**
## 以Volume的方式挂载Config Map

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: charm
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```

**测试后发现通过这种方式挂载Config Map，更新Config Map会触发Deployment更新。**

另外， 如果使用ConfigMap的subPath挂载为Container的Volume，Kubernetes不会做自动热更新
[详情点击](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically)

那么问题来了，

## 为什么通过Volume挂载的configMap更新后能触发Deployment的更新呢？
**ENV 是在容器启动的时候注入的，启动之后 kubernetes 就不会再改变环境变量的值**，且同一个 namespace 中的 pod 的环境变量是不断累加的。但是Volume不同，**kubelet的源码里KubeletManager中是有Volume Manager**的，这就说明Kubelet会监控管理每个Pod中的Volume资源，当发现配置的Volume更新后，就会重建Pod，以更新所用的Volume，但是会有一定延迟，大概10秒以内。
源码如下

```go
// Kubelet is the main kubelet implementation.
type Kubelet struct {
	kubeletConfiguration kubeletconfiginternal.KubeletConfiguration

	// hostname is the hostname the kubelet detected or was given via flag/config
	hostname string
	// hostnameOverridden indicates the hostname was overridden via flag/config
	hostnameOverridden bool

	nodeName        types.NodeName
	runtimeCache    kubecontainer.RuntimeCache
	kubeClient      clientset.Interface
	heartbeatClient clientset.Interface
	iptClient       utilipt.Interface
	rootDirectory   string

	lastObservedNodeAddressesMux sync.RWMutex
	lastObservedNodeAddresses    []v1.NodeAddress

	// onRepeatedHeartbeatFailure is called when a heartbeat operation fails more than once. optional.
	onRepeatedHeartbeatFailure func()

	// podWorkers handle syncing Pods in response to events.
	podWorkers PodWorkers

	// resyncInterval is the interval between periodic full reconciliations of
	// pods on this node.
	resyncInterval time.Duration

	// sourcesReady records the sources seen by the kubelet, it is thread-safe.
	sourcesReady config.SourcesReady

	// podManager is a facade that abstracts away the various sources of pods
	// this Kubelet services.
	podManager kubepod.Manager

	// Needed to observe and respond to situations that could impact node stability
	evictionManager eviction.Manager

	// Optional, defaults to /logs/ from /var/log
	logServer http.Handler
	// Optional, defaults to simple Docker implementation
	runner kubecontainer.ContainerCommandRunner

	// cAdvisor used for container information.
	cadvisor cadvisor.Interface

	// Set to true to have the node register itself with the apiserver.
	registerNode bool
	// List of taints to add to a node object when the kubelet registers itself.
	registerWithTaints []api.Taint
	// Set to true to have the node register itself as schedulable.
	registerSchedulable bool
	// for internal book keeping; access only from within registerWithApiserver
	registrationCompleted bool

	// dnsConfigurer is used for setting up DNS resolver configuration when launching pods.
	dnsConfigurer *dns.Configurer

	// masterServiceNamespace is the namespace that the master service is exposed in.
	masterServiceNamespace string
	// serviceLister knows how to list services
	serviceLister serviceLister
	// nodeLister knows how to list nodes
	nodeLister corelisters.NodeLister

	// a list of node labels to register
	nodeLabels map[string]string

	// Last timestamp when runtime responded on ping.
	// Mutex is used to protect this value.
	runtimeState *runtimeState

	// Volume plugins.
	volumePluginMgr *volume.VolumePluginMgr

	// Handles container probing.
	probeManager prober.Manager
	// Manages container health check results.
	livenessManager proberesults.Manager
	startupManager  proberesults.Manager

	// How long to keep idle streaming command execution/port forwarding
	// connections open before terminating them
	streamingConnectionIdleTimeout time.Duration

	// The EventRecorder to use
	recorder record.EventRecorder

	// Policy for handling garbage collection of dead containers.
	containerGC kubecontainer.ContainerGC

	// Manager for image garbage collection.
	imageManager images.ImageGCManager

	// Manager for container logs.
	containerLogManager logs.ContainerLogManager

	// Secret manager.
	secretManager secret.Manager

	// ConfigMap manager.
	configMapManager configmap.Manager

	// Cached MachineInfo returned by cadvisor.
	machineInfo *cadvisorapi.MachineInfo

	// Handles certificate rotations.
	serverCertificateManager certificate.Manager

	// Syncs pods statuses with apiserver; also used as a cache of statuses.
	statusManager status.Manager

	// VolumeManager runs a set of asynchronous loops that figure out which
	// volumes need to be attached/mounted/unmounted/detached based on the pods
	// scheduled on this node and makes it so.
	volumeManager volumemanager.VolumeManager

	// 以下省略
}
```

## 那我们怎么才能让更新Config Map一定触发Deployment滚动更新呢？
可以通过修改 pod annotations 的方式强制触发滚动更新。这样一定会触发更新，亲测有效。
```bash
 kubectl patch deployment test-deploy --patch '{"spec": {"template": {"metadata": {"annotations": {"update": "2" }}}}}'
```

