---
title: k8s集群部署Calico踩坑
categories: Kubernetes
---

## 部署注意事项
在使用 Calico 前当然最好撸一下官方文档，地址在这里 [Calico 官方文档](https://www.projectcalico.org/)，其中部署前需要注意以下几点

 1. 官方文档中要求 kubelet 配置必须增加 --network-plugin=cni 选项
 3. kubec-proxy 组件不能采用 --masquerade-all 启动，因为会与 Calico policy 冲突
 4. 启用 RBAC 后需要设置对应的 RoleBinding，参考 官方文档 RBAC 部分
<!--more-->
关于在k8s集群中部署calico所用yaml有两个，不同点在于是使用Kubernetes API datastore还是etcd.使用etcd的calico文件在[这里](https://docs.projectcalico.org/v3.8/manifests/calico-etcd.yaml)
## Calico 官方部署方式
在已经有了一个 Kubernetes 集群的情况下，官方部署方式描述的很简单，只需要改一改 yml 配置，然后 create 一下即可，具体描述见 [官方文档](https://docs.projectcalico.org/v3.8/getting-started/kubernetes/)
但是，如果我们有使用证书的话，需要对calico的yaml文件做如下修改

```
sed -i 's@.*etcd_endpoints:.*@\ \ etcd_endpoints:\ \"https://192.168.168.2:2379,https://192.168.168.3:2379,https://192.168.168.4:2379\"@gi' calico.yaml
 
### 替换 Etcd 证书
export ETCD_CERT=`cat /opt/kubernetes/ssl/etcd.pem | base64 | tr -d '\n'`
export ETCD_KEY=`cat /opt/kubernetes/ssl/etcd-key.pem | base64 | tr -d '\n'`
export ETCD_CA=`cat /opt/kubernetes/ssl/ca.pem | base64 | tr -d '\n'`
 
sed -i "s@.*etcd-cert:.*@\ \ etcd-cert:\ ${ETCD_CERT}@gi" calico.yaml
sed -i "s@.*etcd-key:.*@\ \ etcd-key:\ ${ETCD_KEY}@gi" calico.yaml
sed -i "s@.*etcd-ca:.*@\ \ etcd-ca:\ ${ETCD_CA}@gi" calico.yaml
 
sed -i 's@.*etcd_ca:.*@\ \ etcd_ca:\ "/calico-secrets/etcd-ca"@gi' calico.yaml
sed -i 's@.*etcd_cert:.*@\ \ etcd_cert:\ "/calico-secrets/etcd-cert"@gi' calico.yaml
sed -i 's@.*etcd_key:.*@\ \ etcd_key:\ "/calico-secrets/etcd-key"@gi' calico.yaml
 
### 替换 IPPOOL 地址
sed -i 's/192.168.0.0/10.2.0.0/g' calico.yaml
```
## Standard Hosted Install 的坑
**由于我的k8s集群各节点存在多网卡，不同子网。结果出现的问题是个别 Calico 节点无法启动，同时创建 deployment 后，执行 route -n 会发现每个 node 只有自己节点 Pod 的路由，正常每个 node 上会有所有 node 上 Pod 网段的路由**。官方文档中直接创建的 calico.yml 文件中，使用 DaemonSet 方式启动 calico-node，同时 calico-node 的 IP 设置和 NODENAME 设置均为空，此时 calico-node 会进行自动获取，网络复杂情况下获取会出现问题；比如 IP 拿到了 不同网卡的 IP，NODENAME 获取不正确等，最终导致出现很奇怪的错误。

解决方法也很简单。直接修改calico的yaml文件即可。有些博客会推荐大家使用service来启动calico node。实在是多此一举。**因为我们在启动pod内的容器时可以获取到当前k8s节点和pod的一些物理信息的，比如ip、hostname等等。** 具体文档可以参考[这里](https://kubernetes.io/zh/docs/tasks/inject-data-application/environment-variable-expose-pod-information/) 所以我们可以修改calico yaml文件如下：
**我们分别使用k8s节点的ip作为calico node容器内的IP和NODENAME信息。** 防止name重复和自动获取所用的ip地址不在同一网段。
```
# Source: calico/templates/calico-node.yaml
# This manifest installs the calico-node container, as well
# as the CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        # This, along with the CriticalAddonsOnly toleration below,
        # marks the pod as a critical add-on, ensuring it gets
        # priority scheduling and that its resources are reserved
        # if it ever gets evicted.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      initContainers:
        # This container installs the CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: calico/cni:v3.8.2
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # CNI MTU Config variable
            - name: CNI_MTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Prevents the container from sleeping forever.
            - name: SLEEP
              value: "false"
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
        # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
        # to communicate with Felix over the Policy Sync API.
        - name: flexvol-driver
          image: calico/pod2daemon-flexvol:v3.8.2
          volumeMounts:
          - name: flexvol-driver-host
            mountPath: /host/driver
      containers:
        # Runs calico-node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: calico/node:v3.8.2
          env:
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Set noderef for node controller.
            - name: CALICO_K8S_NODE_REF
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "192.168.0.0/16"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
            #  set calico node name
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
              host: localhost
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -bird-ready
              - -felix-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
            - name: policysync
              mountPath: /var/run/nodeagent
      volumes:
        # Used by calico-node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400
        # Used to create per-pod Unix Domain Sockets
        - name: policysync
          hostPath:
            type: DirectoryOrCreate
            path: /var/run/nodeagent
        # Used to install Flex Volume Driver
        - name: flexvol-driver-host
          hostPath:
            type: DirectoryOrCreate
            path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
```
## 关于在集群中使用keepalived挂载VIP后NodePort无法通过VIP访问的问题

**目前由于kube-proxy的代理mode使用的是ipvs，已经取代了之前的iptables，导致目前该问题还无法解决。如果切换到iptables就可以了。**
