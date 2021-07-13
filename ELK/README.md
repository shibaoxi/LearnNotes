# ELK 学习笔记

## ELK 简介

ELK是Elasticsearch、Logstash、Kibana三大开源框架首字母大写简称(但是后期出现的filebeat(beats中的一种)可以用来替代logstash的数据收集功能，比较轻量级)。市面上也被成为Elastic Stack。

Filebeat是用于转发和集中日志数据的轻量级传送工具。Filebeat监视您指定的日志文件或位置，收集日志事件，并将它们转发到Elasticsearch或 Logstash进行索引。Filebeat的工作方式如下：启动Filebeat时，它将启动一个或多个输入，这些输入将在为日志数据指定的位置中查找。对于Filebeat所找到的每个日志，Filebeat都会启动收集器。每个收集器都读取单个日志以获取新内容，并将新日志数据发送到libbeat，libbeat将聚集事件，并将聚集的数据发送到为Filebeat配置的输出。

　　Logstash是免费且开放的服务器端数据处理管道，能够从多个来源采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中。Logstash能够动态地采集、转换和传输数据，不受格式或复杂度的影响。利用Grok从非结构化数据中派生出结构，从IP地址解码出地理坐标，匿名化或排除敏感字段，并简化整体处理过程。

　　Elasticsearch是Elastic Stack核心的分布式搜索和分析引擎,是一个基于Lucene、分布式、通过Restful方式进行交互的近实时搜索平台框架。Elasticsearch为所有类型的数据提供近乎实时的搜索和分析。无论您是结构化文本还是非结构化文本，数字数据或地理空间数据，Elasticsearch都能以支持快速搜索的方式有效地对其进行存储和索引。

　　Kibana是一个针对Elasticsearch的开源分析及可视化平台，用来搜索、查看交互存储在Elasticsearch索引中的数据。使用Kibana，可以通过各种图表进行高级数据分析及展示。并且可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以汇总、分析和搜索重要数据日志。还可以让海量数据更容易理解。它操作简单，基于浏览器的用户界面可以快速创建仪表板（dashboard）实时显示Elasticsearch查询动态

## ELK 架构介绍

1. **beats+elasticsearch+kibana模式**

    <img src="https://i.loli.net/2021/07/13/i7JD6AEdQmO2WMv.png" width=600 />

    如上图所示，该ELK框架由beats（日志分析我们通常使用filebeat）+elasticsearch+kibana构成，这个框架比较简单，入门级的框架。其中filebeat也能通过module对日志进行简单的解析和索引。并查看预建的Kibana仪表板。

2. **beats+logstash+elasticsearch+kibana模式**

    <img src="https://i.loli.net/2021/07/13/JBrh6kLHMDqCgQ5.png" width=600 />

    该框架是在上面的框架的基础上引入了logstash，引入logstash带来的好处如下：

    * 通Logstash具有基于磁盘的自适应缓冲系统，该系统将吸收传入的吞吐量，从而减轻背压
    * 从其他数据源（例如数据库，S3或消息传递队列）中提取
    * 将数据发送到多个目的地，例如S3，HDFS或写入文件
    * 使用条件数据流逻辑组成更复杂的处理管道

    filebeat结合logstash带来的优势：
    * 水平可扩展性，高可用性和可变负载处理：filebeat和logstash可以实现节点之间的负载均衡，多个logstash可以实现logstash的高可用
    * 消息持久性与至少一次交付保证：使用Filebeat或Winlogbeat进行日志收集时，可以保证至少一次交付。从Filebeat或Winlogbeat到Logstash以及从Logstash到Elasticsearch的两种通信协议都是同步的，并且支持确认。Logstash持久队列提供跨节点故障的保护。对于Logstash中的磁盘级弹性，确保磁盘冗余非常重要。
    * 具有身份验证和有线加密的端到端安全传输：从Beats到Logstash以及从 Logstash到Elasticsearch的传输都可以使用加密方式传递 。与Elasticsearch进行通讯时，有很多安全选项，包括基本身份验证，TLS，PKI，LDAP，AD和其他自定义领域
    当然在该框架的基础上还可以引入其他的输入数据的方式：比如：TCP，UDP和HTTP协议是将数据输入Logstash的常用方法（如下图所示）：

    <img src="https://i.loli.net/2021/07/13/cVzaJDTPset6UMY.png" width=600 />

3. **beats+缓存/消息队列+logstash+elasticsearch+kibana模式**

<img src="https://i.loli.net/2021/07/13/lxKPAoOaWe9ysmI.png" width=600 />

在如上的基础上我们可以在beats和logstash中间添加一些组件redis、kafka、RabbitMQ等，添加中间件将会有如下好处：
第一，降低对日志所在机器的影响，这些机器上一般都部署着反向代理或应用服务，本身负载就很重了，所以尽可能的在这些机器上少做事；
第二，如果有很多台机器需要做日志收集，那么让每台机器都向Elasticsearch持续写入数据，必然会对Elasticsearch造成压力，因此需要对数据进行缓冲，同时，这样的缓冲也可以一定程度的保护数据不丢失；
第三，将日志数据的格式化与处理放到Indexer中统一做，可以在一处修改代码、部署，避免需要到多台机器上去修改配置
