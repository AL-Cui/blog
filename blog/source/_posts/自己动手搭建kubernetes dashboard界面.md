---
title: kubernetes dashboard
categories: Kubernetes
---

# 自己动手搭建kubernetes dashboard界面

kubernetes官方提供了一套实用的dashboard界面。但是，入学者按照github上搭建安装时往往会遇到各种问题。顺手写个博客记录一下，也希望能帮到一些初学者。

## 搭建前准备
kubernetes官方提供了一套实用的dashboard界面。但是，入学者按照github上搭建安装时往往会遇到各种问题。顺手写个博客记录一下，也希望能帮到一些初学者。目前，我搭建的kubernetes dashboard版本是1.10.1.搭建之前我们首先要有一套k8s集群环境。搭建k8s环境有空再写博客，另外还需要准备以下东西。
<!--more-->

 1. 从github上下载kubernetes dashboard的安装yaml文件，github地址在[此](https://github.com/kubernetes/dashboard)，yaml文件在[此](https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml)
 2. 默认安装dashboard的时候，会需要**heapster**服务，yaml文件的地址在[此](https://github.com/kubernetes-retired/heapster)，依据README安装即可；
 3. 还需要**metrics**，地址在[此](https://github.com/kubernetes-incubator/metrics-server)，依据README安装即可；
 4. 所需各类镜像如是局域网环境，需提前下载好，通过docker save压缩成tar包，再copy到局域网后docker load加载。

## 搭建步骤
创建dashboard各服务。将dashboard service改为Nodeport方便我们测试连接
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019091917002441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
然后依次创建heapster、metrics服务。然后我们访问集群任一节点https://Ip:Nodeport url。得到如下界面：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919170322147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
我们看到访问需要token。我们需要一个用户授予admin权限。通过以下yaml文件来创建：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
我们创建了一个名为admin-user的SA，通过ClusterRoleBinding绑定名为cluster-admin的ClusterRole。

```
kubectl get secret -n kube-system|grep admin-token
获取secret名字
kubectl get secret “secret的名字” -o jsonpath={.data.token} -n kube-system |base64 -d
```
获得一个base64字符串，粘贴到网页中。
**切记，请使用火狐浏览器，我使用google和IE都无法进入页面**

