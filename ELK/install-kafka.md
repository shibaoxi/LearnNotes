# 安装kafka

## 简介

>Kafka 是一种高吞吐的分布式发布订阅消息系统，能够替代传统的消息队列用于解耦合数据处理，缓存未处理消息等，同时具有更高的吞吐率，支持分区、多副本、冗余，因此被广泛用于大规模消息数据处理应用。Kafka 支持Java 及多种其它语言客户端，可与Hadoop、Storm、Spark等其它大数据工具结合使用。

## 安装步骤

### 安装JDK1.8

安装

```shell
yum install -y java-1.8.0-openjdk.x86_64
```

验证

```shell
java -version

openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
```

### 安装Zookeeper

Zookeeper的下载地址：<https://www.apache.org/dyn/closer.cgi/zookeeper/>

下载并解压到/opt/zookeeper目录下

```shell
curl -L -O https://mirror-hk.koddos.net/apache/zookeeper/stable/apache-zookeeper-3.6.3.tar.gz
tar -zxvf apache-zookeeper-3.6.3.tar.gz
mv apache-zookeeper-3.6.3 /opt/zookeeper
```

编辑配置文件

```shell
# 复制一份zoo_sample.cfg 文件并改名为zoo.cfg
cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
# cat /opt/zookeeper/conf/zoo.cfg | egrep -v "^$|#"
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper/data
clientPort=2181
```

配置环境变量

```shell
vim /etc/profile
#在profile末尾追加
export ZK_HOME=/opt/zookeeper
export PATH=$ZK_HOME/bin:$PATH
```

启动Zookeeper

```shell
cd /opt/zookeeper/bin
./zkServer.sh start
```

### 安装Kafka

下载地址： <https://kafka.apache.org/downloads.html>
