---
title: K8S集群日志收集方案
categories: Kubernetes
---

# K8S集群日志收集方案

   在大型分布式部署的架构中，不同的服务模块部署在不同的服务器中，问题出现时，大部分情况需要根据问题暴露的关键信息定位具体的服务器和服务模块。常见的解决思路是建立一套集中式日志收集系统，将所有节点上的日志统一收集、管理、访问，将极大提高定位问题的效率。
 一个完整的集中式日志系统，需要包含以下几个主要特点：

 - 收集－能够采集多种来源的日志数据 
 - 传输－能够稳定的把日志数据传输到中央系统 
 - 存储－如何存储日志数据 
 - 分析－可以支持 UI 分析
 - 警告－能够提供错误报告，监控机制

目前在K8S集群内部收集日志有以下几种方案
| 编号  | 方案 | 优点 | 缺点 |
|----|----|----|----|
| 1     |每个app的镜像中都集成日志收集组件  | 部署方便，kubernetes的yaml文件无须特别配置，可以为每个app自定义日志收集配置 | 强耦合，不方便应用和日志收集组件升级和维护且会导致镜像过大 |
| 2     | 单独创建一个日志收集组件跟app的容器一起运行在同一个pod中 |低耦合，扩展性强，方便维护和升级  | 需要对kubernetes的yaml文件进行单独配置，略显繁琐 |
| 3     | 将所有的Pod的日志都挂载到宿主机上，每台主机上单独起一个日志收集Pod | 完全解耦，性能最高，管理起来最方便 | 需要统一日志收集规则，目录和输出方式 |

## 方案一
基本不考虑
## 方案二
**方案二就是常见的ELK（Elasticsearch、Logstash、Kibana）+ Filebeat**。引入了各类Lib Beats（Package Beat、Top Beat、File Beat等）运行在app应用的Pod中收集日志，转发给Logstash。此方案将手机端的Logstash替换为beats，更灵活，消耗资源更少，扩展性更强。同时可配置Logstash 和Elasticsearch 集群用于支持大集群系统的运维日志数据监控和查询。
### 简单介绍下ELK
Elasticsearch + Logstash + Kibana（ELK）是一套开源的日志管理方案

 - Logstash：负责日志的收集，处理和储存
 - Elasticsearch：负责日志检索和分析
 - Kibana：负责日志的可视化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103150908465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
***Elasticsearch简介***
默认情况下，ES集群节点都是混合节点，即在elasticsearch.yml中默认node.master: true和node.data: true。当ES集群规模达到一定程度以后，就需要注意对集群节点进行角色划分。ES集群节点可以划分为三种：主节点、数据节点和客户端节点。
 - **master-主节点**  维护元数据，管理集群节点状态；不负责数据写入和查询。elasticsearch.yml
   ```
   node.master: true
   node.data: false
   ```
 - **data-数据节点**  负责数据的写入与查询，压力大。elasticsearch.yml
   ```
   node.master: false
   node.data: true
   ```
- **data-客户端节点**  负责任务分发和结果汇聚，分担数据节点压力。elasticsearch.yml
   ```
   node.master: false
   node.data: false
   ```
- **data-混合节点**  综合上述三个节点的功能。elasticsearch.yml
   ```
   node.master: true
   node.data: true
   ```

**Filebeat工作原理**
Filebeat由两个主要组件组成：prospectors 和 harvesters。这两个组件协同工作将文件变动发送到指定的输出中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103154511413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
**Harvester（收割机）**：负责读取单个文件内容。每个文件会启动一个Harvester，每个Harvester会逐行读取各个文件，并将文件内容发送到制定输出中。Harvester负责打开和关闭文件，意味在Harvester运行的时候，文件描述符处于打开状态，如果文件在收集中被重命名或者被删除，Filebeat会继续读取此文件。所以在Harvester关闭之前，磁盘不会被释放。默认情况filebeat会保持文件打开的状态，直到达到close_inactive（如果此选项开启，filebeat会在指定时间内将不再更新的文件句柄关闭，时间从harvester读取最后一行的时间开始计时。若文件句柄被关闭后，文件发生变化，则会启动一个新的harvester。关闭文件句柄的时间不取决于文件的修改时间，若此参数配置不当，则可能发生日志不实时的情况，由scan_frequency参数决定，默认10s。Harvester使用内部时间戳来记录文件最后被收集的时间。例如：设置5m，则在Harvester读取文件的最后一行之后，开始倒计时5分钟，若5分钟内文件无变化，则关闭文件句柄。默认5m）。

**Prospector（勘测者）**：负责管理Harvester并找到所有读取源。

```yaml
filebeat.prospectors:

- input_type: log

  paths:

    - /apps/logs/*/info.log
```
Prospector会找到/apps/logs/*目录下的所有info.log文件，并为每个文件启动一个Harvester。Prospector会检查每个文件，看Harvester是否已经启动，是否需要启动，或者文件是否可以忽略。若Harvester关闭，只有在文件大小发生变化的时候Prospector才会执行检查。只能检测本地的文件。

**Filebeat如何记录文件状态**：

将文件状态记录在文件中（默认在/var/lib/filebeat/registry）。此状态可以记住Harvester收集文件的偏移量。若连接不上输出设备，如ES等，filebeat会记录发送前的最后一行，并再可以连接的时候继续发送。Filebeat在运行的时候，Prospector状态会被记录在内存中。Filebeat重启的时候，利用registry记录的状态来进行重建，用来还原到重启之前的状态。每个Prospector会为每个找到的文件记录一个状态，对于每个文件，Filebeat存储唯一标识符以检测文件是否先前被收集。

**Filebeat如何保证事件至少被输出一次**：

Filebeat之所以能保证事件至少被传递到配置的输出一次，没有数据丢失，是因为filebeat将每个事件的传递状态保存在文件中。在未得到输出方确认时，filebeat会尝试一直发送，直到得到回应。若filebeat在传输过程中被关闭，则不会再关闭之前确认所有时事件。任何在filebeat关闭之前为确认的时间，都会在filebeat重启之后重新发送。这可确保至少发送一次，但有可能会重复。可通过设置shutdown_timeout 参数来设置关闭之前的等待事件回应的时间（默认禁用）。

 

**Logstash工作原理**：
Logstash事件处理有三个阶段：inputs → filters → outputs。是一个接收，处理，转发日志的工具。支持系统日志，webserver日志，错误日志，应用日志，总之包括所有可以抛出来的日志类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103154804689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
**Input：输入数据到logstash**。

一些常用的输入为：

- file：从文件系统的文件中读取，类似于tial -f命令

- syslog：在514端口上监听系统日志消息，并根据RFC3164标准进行解析

- redis：从redis service中读取

- beats：从filebeat中读取

- Filters：数据中间处理，对数据进行操作。

一些常用的过滤器为：

- grok：解析任意文本数据，Grok 是 Logstash 最重要的插件。它的主要作用就是将文本格式的字符串，转换成为具体的结构化的数据，配合正则表达式使用。内置120多个解析语法。

官方提供的grok表达式：https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns
grok在线调试：https://grokdebug.herokuapp.com/

- mutate：对字段进行转换。例如对字段进行删除、替换、修改、重命名等。

- drop：丢弃一部分events不进行处理。

- clone：拷贝 event，这个过程中也可以添加或移除字段。

- geoip：添加地理信息(为前台kibana图形化展示使用)

**Outputs：outputs是logstash处理管道的最末端组件。**一个event可以在处理过程中经过多重输出，但是一旦所有的outputs都执行结束，这个event也就完成生命周期。

一些常见的outputs为：

- elasticsearch：可以高效的保存数据，并且能够方便和简单的进行查询。

- file：将event数据保存到文件中。

- graphite：将event数据发送到图形化组件中，一个很流行的开源存储图形化展示的组件。

- Codecs：codecs 是基于数据流的过滤器，它可以作为input，output的一部分配置。Codecs可以帮助你轻松的分割发送过来已经被序列化的数据。

一些常见的codecs：

- json：使用json格式对数据进行编码/解码。

- multiline：将汇多个事件中数据汇总为一个单一的行。比如：java异常信息和堆栈信息。

***为什么要在ELK基础上引入Lib Beats？***
在进行日志收集的过程中，我们首先想到的是使用Logstash，因为它是ELK stack中的重要成员，但是在测试过程中发现，Logstash是基于JDK的，在没有产生日志的情况单纯启动Logstash就大概要消耗500M内存，在每个Pod中都启动一个日志收集组件的情况下，使用logstash有点浪费系统资源，我们选择使用Filebeat替代，经测试单独启动Filebeat容器大约会消耗12M内存，比起logstash相当轻量级。
**Kibana简介**
Kibana通常与 Elasticsearch 一起部署，Kibana 是 Elasticsearch 的一个功能强大的数据可视化 Dashboard，Kibana 允许你通过 web 界面来浏览 Elasticsearch 日志数据。
## 方案三
**此方案就是K8S官方推荐的EFK（ELasticsearch、Fluentd、Kibana）方案**
此方案通过DaemonSet的方式在集群内部每个节点上运行一个Fluent Pod统一收集上层应用层的日志并反馈到Elasticsearch
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103155624399.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)
**Fluentd简介**
Fluentd是一个Ruby语言开发的开源数据收集器，通过它能对数据进行统一收集和消费，能够更好地使用和理解数据。Fluentd将数据结构化为JSON，从而能够统一处理日志数据，包括：收集、过滤、缓存和输出。Fluentd是一个基于插件体系的架构，包括输入插件、输出插件、过滤插件、解析插件、格式化插件、缓存插件和存储插件，通过插件可以扩展和更好的使用Fluentd。

Fluentd 通过一组给定的数据源抓取日志数据，处理后（转换成结构化的数据格式）将它们转发给其他服务，比如 Elasticsearch、对象存储等等。Fluentd 支持超过300个日志存储和分析服务，所以在这方面是非常灵活的。主要运行步骤如下：

- 首先 Fluentd 从多个日志源获取数据
- 结构化并且标记这些数据
- 然后根据匹配的标签将数据发送到多个目标服务去
![在这里插入图片描述](https://img-blog.csdnimg.cn/202001031600510.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1aV9DdWlfNjY2,size_16,color_FFFFFF,t_70)

**Fluentd配置**
一般来说我们是通过一个配置文件来告诉 Fluentd 如何采集、处理数据的，下面简单和大家介绍下 Fluentd 的配置方法。
**日志源配置**
比如我们这里为了收集 Kubernetes 节点上的所有容器日志，就需要做如下的日志源配置：

```ruby
<source>

@id fluentd-containers.log

@type tail

path /var/log/containers/*.log

pos_file /var/log/fluentd-containers.log.pos

time_format %Y-%m-%dT%H:%M:%S.%NZ

tag raw.kubernetes.*

format json

read_from_head true

</source>
```

上面配置部分参数说明如下：

- id：表示引用该日志源的唯一标识符，该标识可用于进一步过滤和路由结构化日志数据
- type：Fluentd 内置的指令，tail表示 Fluentd 从上次读取的位置通过 tail 不断获取数据，另外一个是http表示通过一个 GET 请求来收集数据。
- path：tail类型下的特定参数，告诉 Fluentd 采集/var/log/containers目录下的所有日志，这是 docker 在 
 Kubernetes 节点上用来存储运行容器 stdout 输出日志数据的目录。
- pos_file：检查点，如果 Fluentd 程序重新启动了，它将使用此文件中的位置来恢复日志数据收集。
- tag：用来将日志源与目标或者过滤器匹配的自定义字符串，Fluentd 匹配源/目标标签来路由日志数据。
路由配置

上面是日志源的配置，接下来看看如何将日志数据发送到 Elasticsearch：

```ruby
<match **>

@id elasticsearch

@type elasticsearch

@log_level info

include_tag_key true

type_name fluentd

host "#{ENV['OUTPUT_HOST']}"

port "#{ENV['OUTPUT_PORT']}"

logstash_format true

<buffer>

@type file

path /var/log/fluentd-buffers/kubernetes.system.buffer

flush_mode interval

retry_type exponential_backoff

flush_thread_count 2

flush_interval 5s

retry_forever

retry_max_interval 30

chunk_limit_size "#{ENV['OUTPUT_BUFFER_CHUNK_LIMIT']}"

queue_limit_length "#{ENV['OUTPUT_BUFFER_QUEUE_LIMIT']}"

overflow_action block

</buffer>
```

- match：标识一个目标标签，后面是一个匹配日志源的正则表达式，我们这里想要捕获所有的日志并将它们发送给 Elasticsearch，所以需要配置成**。
- id：目标的一个唯一标识符。
- type：支持的输出插件标识符，我们这里要输出到 Elasticsearch，所以配置成 elasticsearch，这是 Fluentd 的一个内置插件。
- log_level：指定要捕获的日志级别，我们这里配置成info，表示任何该级别或者该级别以上（INFO、WARNING、ERROR）的日志都将被路由到 Elsasticsearch。
- host/port：定义 Elasticsearch 的地址，也可以配置认证信息，我们的 Elasticsearch 不需要认证，所以这里直接指定 host 和 port 即可。
- logstash_format：Elasticsearch 服务对日志数据构建反向索引进行搜索，将 logstash_format 设置为true，Fluentd 将会以 logstash 格式来转发结构化的日志数据。
- Buffer： Fluentd 允许在目标不可用时进行缓存，比如，如果网络出现故障或者 Elasticsearch 不可用的时候。缓冲区配置也有助于降低磁盘的 IO。
## Docker Image获取
所需Docker image都可以从DockerHub或者[Google镜像仓库获取](https://console.cloud.google.com/projectselector2/gcr?supportedpurview=project)获取
**目前Elasticsearch、Logstash、Kibana、Filebeat以及Fluentd都没有官方的ARM64镜像**
**但是，Elasticsearch、Logstash、Kibana有提供DockerFile，另外，*此三组件镜像版本必须一致***
- [Elasticsearch官方DockerFile](https://github.com/elastic/dockerfiles/tree/v7.5.1/elasticsearch)
- [Kibana官方DockerFile](https://github.com/elastic/dockerfiles/tree/v7.5.1/kibana)
- [Logstash官方DockerFile](https://github.com/elastic/dockerfiles/tree/v7.5.1/logstash)

**Filebeat和Fluentd的非官方镜像**：

- kasaoden/filebeat:7.2.0-arm64
- carlosedp/fluentd-elasticsearch:latest
