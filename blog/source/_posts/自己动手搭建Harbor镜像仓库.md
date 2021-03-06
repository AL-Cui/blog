﻿---
title: 自己动手搭建Harbor镜像仓库
categories: Kubernetes
---

## 自己动手搭建Harbor镜像仓库
近期Harbor v1.9 正式面市，该版本称得上是目前功能最多的版本之一，新版本增加了多项优秀功能。
 - Webhook。项目管理员现在可以通过 Webhook 的通知机制，将 Harbor 的项目与技术栈的其余部分连接在一起，简化持续集成和开发过程。
 - 配额。项目管理员可以通过配额限制项目所含 tag 的数目及项目可占用的存储容量（全局和个体），有助于对资源使用加以控制
 - Tag 保留。Harbor 的存储中可能会迅速累积起大量镜像的文件，现在，项目管理员可以利用新的 Tag 保留功能更好地管理镜像生命周期并优化存储分配。
 - CVE 例外策略。该功能允许项目管理员创建一个 CVE 白名单，允许某些镜像在有限的时间段内运行，而不管是否具有特定 CVE 安全漏洞。
 - 内容复制的改进。新版本的 Harbor 可实现与大多数主流云提供商 Registry 的无缝双向复制，满足客户的众多需求和用例。

 下载地址：
 - 官方的[安装文档](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)讲的很详细，这里我们推荐使用离线安装，虽然安装包比较大
 - 离线安装包下载地址[添加链接描述](https://storage.googleapis.com/harbor-releases/release-1.9.0/harbor-offline-installer-v1.9.1.tgz)
## 离线安装HTTP Harbor
安装之前先安装docker-compose
我们下载完安装包以后`tar xvf harbor-offline-installer-<version>.tgz`解压缩，可以看到里边有两个安装脚本prepare.sh和install.sh以及harbor.yml。然后修改harbor.yml中配置参数：
 - 将hostname改为自己的ip
 - 将data_volume改为自己想存放的路径
 - database此参数可以修改为自己的数据库地址，也可以不改harbor会自己创建harbor-db容器
 - http:port可以修改harbor仓库的端口号

配置完参数后我们就可以`./install.sh`来安装harbor了。当然这时也可以通过传入参数来配置，我们可以使用`./install.sh --with-chartmuseum`来使我们的harbor可以管理存储chart。执行指令完成后我们可以看到docker-compose已经为我们启动了一些列容器
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024175314747.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)

Harbor还可以管理其他的镜像仓库，我们可以配置同步策略（手动或者定时），实现Harbor仓库和其他镜像仓库的交互同步。

 1. 新建目标![在这里插入图片描述](https://img-blog.csdnimg.cn/2019102418014919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
 2. 编辑目标![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024180231806.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
 3. 新建规则![在这里插入图片描述](https://img-blog.csdnimg.cn/20191024180304870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
## 向Harbor批量导入docker镜像
这里我们参考rancher的方法，rancher提供了一个自动化脚本，可以批量导入多个docker image。首先我们要将要导入的docker image打成一个tgz包，打包脚本如下：

```
#!/bin/bashIMAGES_LIST=($(docker images | sed '1d' | awk '{print $1":"$2}'))
docker save ${IMAGES_LIST[*]} -o all-images.tar.gz
docker images | sed '1d' | awk '{print $1":"$2}' >> ./all-images.txt
```
rancher脚本下载[地址](https://github.com/rancher/rancher/releases/download/v2.3.1/rancher-load-images.sh)。推送image到远端Harbor指令如下：`./rancher-load-images.sh   -l  all-images.txt   -i  all-images.tar.gz   -r  172.23.5.71:80/library  #仓库地址以及具体的项目`
