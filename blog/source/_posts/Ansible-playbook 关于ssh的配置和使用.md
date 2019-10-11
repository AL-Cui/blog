---
title: ansible中配置ssh
categories: Ansible
---

## 使用ansible时关于ssh的配置和使用
   ansible是基于SSH开发的一款用于远程（批量）地管理服务器资源的工具，这就表示其无需安装客户端，在一台全新的服务器上线之后（只要其有sshd服务在运行）就可以直接加入被管理的集群了。

  关于ansible的配置在/etc/ansible/ansible.cfg文件中，所以关于ansible运行时所使用的ssh配置也可以在此文件中配置。在目前的ansible中，运行ansible时会依次加载 环境变量ANSIBLE_CONFIG，当前目录的ansible.cfg，~/.ansible.cfg，/etc/ansible/ansible.cfg，针对同一个配置项以最先加载到的为准。所以，我们可以单独编写自己的ansible.cfg文件放在当前目录下。

<!--more-->

在使用过程中我遇到了两个问题。一、在调用ansible-playbook 指令时如何很快检验所管理的主机节点ssh连接是否正常？二、当在执行过程如果某个主机节点ssh连接断开时，如何很快获取异常并中断playbook的执行？
## 在调用ansible-playbook 指令时如何很快检验所管理的主机节点ssh连接是否正常？
 通过ansible-playbook我们可以看到--timeout可以设置ssh连接的超时时间，默认是10s。我们可以通过设置改参数灵活判断目标主机节点ssh连接是否已经断开。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019100914182765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)

## 当在执行过程如果某个主机节点ssh连接断开时，如何很快获取异常并中断playbook的执行？
我们开始已经提到，通过ansible.cfg文件我们可以配置有关ssh的一些参数。![在这里插入图片描述](https://img-blog.csdnimg.cn/20191009143037904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)

ssh_args就是配置有关ssh连接的参数。默认情况下，playbook运行过程中某节点ssh连接断开后，要过很长一段时间ansible才会返回错误，期间一直hold在那，严重影响运行效率。

```
ssh_args = -o ControlMaster=auto -o ControlPersist=600s -o ServerAliveInterval=30 -o ServerAliveCountMax=2
```
我们可以通过设置ssh的心跳时间间隔和超时次数解决此问题。

