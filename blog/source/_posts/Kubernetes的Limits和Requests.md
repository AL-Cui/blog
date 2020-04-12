---
title: kubernetes中的Requests和Limits
categories: Kubernetes
---

在k8s的集群环境中，资源的合理分配和使用至关重要。毕竟容器化要解决的问题之一就是资源的充分利用。在集群中分配资源的时候就不得不提到Limits和Requests。
# Namespace配额
众所周知，Kubernetes 是允许管理员在命名空间中指定资源 Requests 和 Limits 的，这一特性对于资源管理限制非常有用。但它目前还存在一定局限：**如果管理员在命名空间中设置了 CPU  Requests 配额，那么所有 Pod 也要在其定义中设置 CPU  Requests，否则就无法被调配资源。**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-example
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 6
    limits.memory: 6Gi
```
这是一个简单的ResourceQuota类型，也就是针对Namespace的配额。它会针对Namespace做如下限额：

 - 所有 CPU Requests 的总和不能超过 4 个内核
 - 所有 RAM Requests 的总和不能超过 4GiB
 - 所有 CPU Limits 的总和不能超过 6个内核
 - 所有 RAM Limits 的总和不能超过 6GiB

# 针对Pod的Request和Limit
刚才讲到我们可以针对Pod进行资源限额，同样也可以设置Pod申请资源的Request和Limit。k8s中会将一个CPU分成1000个shares，这和Cgroup中分成1024略有差异。正常情况下requests的数值应该小于limits，那么该Pod获得的资源可以分为两部分：

 - 完全可靠的资源，资源量大小等于requests值
 - 不可靠的资源，资源量最大等于limits和requests的差额，这份不可靠的资源能够申请到多少，取决于当时主机上容器可用资源的余量。
如下例：

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
 name: redis
 labels:
   name: redis
   app: redis-app
spec:
 replicas: 2
 selector:
   matchLabels:
    name: redis
    role: redisdb
    app: redis-app
 template:
   spec:
     containers:
       - name: redis
         image: redis:5.0.3-alpine
         resources:
           limits:
             memory: 500Mi
             cpu: 1
           requests:
             memory: 250Mi
             cpu: 500m
       - name: busybox
         image: busybox:1.28
         resources:
           limits:
             memory: 100Mi
             cpu: 100m
           requests:
             memory: 50Mi
             cpu: 50m
```

 - Pod 中共两个容器，的有效  Request  是 300MiB 的内存和 550 毫核（millicore）的 CPU。我们需要一个具有足够可用可分配空间的节点来调度 Pod
 - 如果 Redis 容器尝试分配超过 500MB 的 RAM，就会被 OOM-killer
 - 如果 Redis 容器尝试每 100ms 使用 100ms 以上的 CPU，那么 Redis 就会受到 CPU 限制（如果我们一共有 4 个内核，可用时间为 400ms/100ms），从而导致性能下降
 - 如果 busybox 容器尝试分配 100MB 以上的 RAM，也会引起 OOM
 - 如果 busybox 容器尝试每 100ms 使用 10ms 以上的 CPU，也会使 CPU 受到限制，从而导致性能下降
需要注意的是，**Kubernetes 限制的是每个容器，而不是每个 Pod**。其他 CPU 指标，如共享 CPU 资源使用情况，只对分配有参考价值，所以如果遇到了性能上的问题，建议不要在这些指标上浪费时间。
# Requests等于Limits 的时候
一般情况下我们设置的Requests值一般都要小于Limits，但是也存在特殊情况。涉及到一个概念就是**服务质量等级**
 - **Guaranteed**（完全可靠的）；Limits==requests,或者只设置了Limits，此时默认requests等于limits
 - **Burstable**（弹性波动、较可靠的）；分为两种情况：1、Pod中一部分容器在一种或多种资源类型中配置了requests和limits，2、Pod中一部分容器未定义资源配置（requests和limits都未配置）
 - **BestEffort**（尽力而为、不太可靠的）；Pod所有中所有容器都未定义requests和limits

**注意：在容器未定义limits时，limits值默认是节点资源容量的上限。**

另外，**当我们分配CPU的requests和limits相等的时候，就是指该容器独占CPU，需要在kubelet服务的配置中增加`--cpu-manager-policy=static`**
# 可压缩资源和不可压缩资源
我们上文提到，在容器可使用的资源有CPU和内存。所以我们拓展一下k8s集群中的可压缩资源和不可压缩资源概念。
在k8s中，**CPU就是可压缩资源**。空闲的CPU资源会按照容器的requests值得比例进行分配，举例说明：容器A requests1 limits 10；容器B requests2 limits 8，加入一开始该节点上可用CPU位3，那么两个容器恰好得到各自requests的量。此时节点又释放了1.5CPU，A和B都需要更多CPU资源，那么这1.5CPU就会按照A和B的requests量按比例分配，最后A得到1.5CPU，B得到3CPU。


**目前，k8s支持的不可压缩资源是内存**。Pod中可以得到requests的内存量，如果Pod使用小于该值，那么Pod正常运行；如果Pod使用超过了该值极=就有可能被k8s杀掉。比如，Pod A使用了大于requests但是小于limits的内存，此时Pod B使用了小于requests的内存，但是Pod B中的程序突然压力增大，向k8s请求更多的但是不超过自己requests的内存资源，而节点上已没有空闲内存资源，这时候k8s就可能会直接kill Pod A。

# 选择可靠的Requests和Limits
具备一定 Kubernetes 经验的人都知道正确设置 Request 和 Limit 对于应用程序和集群的性能的重要性。

理想情况下，Pod 请求多少资源，它就用多少资源，但在现实场景下，资源使用是不断变化的，而且它的变化没有规律，不可预测。

如果 Pod 的资源使用量远低于请求量，那会导致资源（金钱）浪费；如果资源使用量高于请求量，那就会使节点出现性能问题。因此在实际操作中，我们可以把  Request 值上下浮动 25％ 作为一个良性参考标准。

而关于 Limit，设置合理的 Limit 数值其实需要尝试，因为它主要取决于应用程序的性质、需求模型、对错误的容忍度以及许多其他因素，没有固定答案。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200412200909653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
**另一件需要考虑的事是在节点上允许的 Limits 过量使用。**
![
](https://img-blog.csdnimg.cn/20200412200932891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)

**这些 Limits 由用户执行，因为 Kubernetes 没有关于超额使用的自动化机制。**
