---
title: Istio怎么劫持了流量
categories: Kubernetes
---

# Istio怎么劫持了流量
当我们使用istio的时候，有时候会思考Istio到底是怎么劫持了k8s集群的流量呢？
首先我们要明白k8s本身的流量是怎么流转的。举个例子，服务A通过服务B部署时注册的Service名称来填写调用地址，这个地址被翻译成一个域名，通过k8s中的私有DNS系统翻译成一个虚拟的ClusterIP，再通过部署在每一个节点上的Kube-proxy服务负载给Service对应的Pod。那使用Istio的部署会对k8s
原有的服务网格结构造成哪些影响呢？
# Sidecar
Sidecar是与应用一起部署的，它代理一切出入应用的请求，并将其转发到正确的目的地。这样一来，复杂的分布式调用网络本身就对应用透明了。也许会有人对单独部署一个Sidecar进程的稳定性产生疑问，Sidecar的独特之处就在于每个应用本身都独立部署了一套进程，即使出问题也只影响一个或几个服务。
## Sidecar的注入方式
Sidecar的部署支持自动注入与手动配置两种方式，手动注入需要对部署的yaml文件进行简单的修改即可。

```bash
istioctl kube-inject -f book-info.yml | kubectl apply -f -
```
自动注入目前需要k8s配合使用，只需将应用对应的Namespace设置一个label即可

```bash
kubectl label namespace default istio-injection=enabled
```
这样一来，服务一旦启动，Sidecar就会使用Kubernetes的Mutating Webhook Admission Controller扩展自动注入。
那么，Mutating Webhook Admission Controller是怎么自动注入的呢？为什么配置了自动注入的namespace就会在创建pod的时候创建Sidecar容器呢？
# 动态准入控制
Admission webhook是一种用于接收准入请求并对其进行处理的HTTP回调机制。可以定义两种类型的admission webhook，即 validating admission webhook 和 mutating admission webhook。 Mutating admission webhook 会先被调用。**它们可以更改发送到 API 服务器的对象以执行自定义的设置默认值操作**。在完成了所有对象修改并且 API 服务器也验证了所传入的对象之后，validating admission webhook 会被调用，并通过拒绝请求的方式来强制实施自定义的策略。这样，就回答了为什么配置自动注入后创建的pod就能自动创建Sidecar容器。
那么，Istio中的mutating admission webhook是怎么定义的呢？其实，要想使用admission webhook首先要定义 ValidatingWebhookConfiguration 或者 MutatingWebhookConfiguration 动态配置哪些资源要被哪些 admission webhook 处理。Istio中定义的MutatingWebhookConfiguration如下：

```yaml
# Source: istio/charts/sidecarInjectorWebhook/templates/mutatingwebhook.yaml

apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
  labels:
    app: sidecarInjectorWebhook
    chart: sidecarInjectorWebhook
    heritage: Tiller
    release: istio
webhooks:
  - name: sidecar-injector.istio.io
    clientConfig:
      service:
        name: istio-sidecar-injector
        namespace: istio-system
        path: "/inject"
      caBundle: ""
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    failurePolicy: Fail
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
```
创建一个 service 作为 webhook 服务器的前端，这里就是istio-sidecar-injector service。这个MutatingWebhookConfiguration很清晰，就是匹配带有istio-injection标签的namespace中的 CREATE POD 请求，匹配到之后按照clientConfig中制定的方向发送请求给istio-sidecar-injector service。
istio-sidecar-injector指定的是一个deployment，内容如下：

```yaml
# Source: istio/charts/sidecarInjectorWebhook/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
  labels:
    app: sidecarInjectorWebhook
    chart: sidecarInjectorWebhook
    heritage: Tiller
    release: istio
    istio: sidecar-injector
spec:
  replicas: 1
  selector:
    matchLabels:
      istio: sidecar-injector
  strategy:
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        app: sidecarInjectorWebhook
        chart: sidecarInjectorWebhook
        heritage: Tiller
        release: istio
        istio: sidecar-injector
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      serviceAccountName: istio-sidecar-injector-service-account
      containers:
        - name: sidecar-injector-webhook
          image: "docker.io/istio/sidecar_injector:1.5.1"
          imagePullPolicy: IfNotPresent
          args:
            - --caCertFile=/etc/istio/certs/root-cert.pem
            - --tlsCertFile=/etc/istio/certs/cert-chain.pem
            - --tlsKeyFile=/etc/istio/certs/key.pem
            - --injectConfig=/etc/istio/inject/config
            - --meshConfig=/etc/istio/config/mesh
            - --healthCheckInterval=2s
            - --healthCheckFile=/tmp/health
            - --reconcileWebhookConfig=true
          volumeMounts:
          - name: config-volume
            mountPath: /etc/istio/config
            readOnly: true
          - name: certs
            mountPath: /etc/istio/certs
            readOnly: true
          - name: inject-config
            mountPath: /etc/istio/inject
            readOnly: true
          livenessProbe:
            exec:
              command:
                - /usr/local/bin/sidecar-injector
                - probe
                - --probe-path=/tmp/health
                - --interval=4s
            initialDelaySeconds: 4
            periodSeconds: 4
          readinessProbe:
            exec:
              command:
                - /usr/local/bin/sidecar-injector
                - probe
                - --probe-path=/tmp/health
                - --interval=4s
            initialDelaySeconds: 4
            periodSeconds: 4
          resources:
            requests:
              cpu: 10m
            
      volumes:
      - name: config-volume
        configMap:
          name: istio
      - name: certs
        secret:
          secretName: istio.istio-sidecar-injector-service-account
      - name: inject-config
        configMap:
          name: istio-sidecar-injector
          items:
          - key: config
            path: config
          - key: values
            path: values
      affinity:      
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - "amd64"
                - "ppc64le"
                - "s390x"
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 2
            preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - "amd64"
          - weight: 2
            preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - "ppc64le"
          - weight: 2
            preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - "s390x"   
```
这个Pod主要做的事我们通过现象来分析。
# Pod中的三个容器
我们通过istio官方的demo我们可以发现，开启了自动注入后创建的review Pod中有三个容器，分别是：

 - k8s_reviews_revierws-v3
 - k8s_istio_proxy_reviews-v3
 - k8s_POD_reviews
通过分析，第一个就是应用本身，第三个是kubernetes的pause容器，第二个容器就是istio系统中的Sidecar，并且进入容器我们也能看到里边运行的就是Envoy。这个容器是和Pod一一对应的，**所以Sidecar是部署在Pod这个层级的**，这一点很重要。
## Sidecar是如何劫持流量的呢
我们发现了istio会给我们的每个Pod创建一个Sidecar容器，那么他是如何劫持流量的呢？
我们进入到这个容器后，执行ifconfig会惊奇发现，应用容器使用的IP地址与istio-proxy竟然是一样的。这说明应用容器与istio-proxy两个容器共享了网络。也就是说，istio在pod启动的时候使用自己的特殊pause容器istio-proxy替换掉了Kubernetes官方的pause，使得应用与istio-proxy处于同一个网络栈下。那最后的问题就是istio如何将应用的进出流量都重新定向到istio-proxy容器监听的端口。为了搞明白这个问题，我们可以手动注入Sidecar，来对比yaml文件的变化。主要变化是init container中多了一个proxy_init容器，查看它的Dockerfile文件可以发现，这个容器主要就是运行了一个脚本：

```powershell
FROM ubuntu:xenial
RUN apt-get update && apt-get install -y \
    iproute2 \
    iptables \
 && rm -rf /var/lib/apt/lists/*
ADD istio-iptables.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/istio-iptables.sh"]
```
该脚本源码就不具体分析了，不过也能猜出大概。istio正是通过iptables以及iproute2这个工具对Linux网络流量配置了过滤规则来实现Sidecar的流量劫持，其执行的脚本正是istio-iptables这个脚本文件。

