# 在Linux上安装elasticsearch 和 kibana

## 前提条件

### 新建用户

>出于安全考虑，es默认不允许以root账号运行

```bash
#创建用户
useradd esuser
#设置密码
passwd esuser
#添加sudo权限
visudo
## Allows people in group wheel to run all commands

esuser        ALL=(ALL)       ALL
```

### 修改/etc/security/limits.conf文件

```bash
#打开文件
vim /etc/security/limits.conf
#在文件最后，增加如下配置
* soft nofile 65536
* hard nofile 65536
```

### 修改/etc/sysctl.conf

```bash
#打开文件
vim /etc/sysctl.conf
#在文件最后添加一行
vm.max_map_count=655360
#保存退出后执行如下命令
sysctl -p
```

## 安装

### 下载es

下载地址：<https://www.elastic.co/cn/elasticsearch/>

```bash
#在/home下面新建文件夹
mkdir -p /home/es
#下载elasticsearch
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz
#目录重命名
mv elasticsearch-7.10.2 elasticsearch

```

### 目录介绍

```bash
$ES_HOME: /home/es/elasticsearch
bin: $ES_HOME/bin  #es启动命令和插件安装命令
conf：$ES_HOME/conf #elasticsearch.yml配置文件目录
data：$ES_HOME/data  #对应的参数path.data，用于存放索引分片数据文件
logs：$ES_HOME/logs  #对应的参数path.logs，用于存放日志
jdk：$ES_HOME/jdk  #自带支持该es版本的jdk
plugins： $ES_HOME/jplugins #插件存放目录
lib： $ES_HOME/lib #存放依赖包，比如java类库
modules： $ES_HOME/modules #包含所有的es模块
```

### 配置自带的java环境

```bash
vim ~/.bashrc
#在后面追加如下内容
export JAVA_HOME=/home/es/elasticsearch/jdk
export PATH=JAVAHOME/bin:PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar\:/lib/tools.jar
```

### 配置加密通信证书

```bash
./bin/elasticsearch-certutil ca  #创建集群认证机构，需要交互输入密码
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12  #为节点颁发证书，与上面密码一样
执行./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password 并输入第一步输入的密码 
执行./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password 并输入第一步输入的密码 
将生成的elastic-certificates.p12、elastic-stack-ca.p12文件移动到config目录下
```

### 配置elasticsearch配置文件

进入es的根目录，然后创建logs 和 data

```bash
mkdir data
mkdir logs
```

打开elasticsearch.yml文件

```bash
vim config/elasticsearch.yml
```

配置elasticsearch.yml 文件

```bash
$ cat config/elasticsearch.yml | egrep -v '^&|#'

cluster.name: my-application
node.name: node-1
node.data: true
node.master: true
path.data: /home/es/data
path.logs: /home/es/logs
network.host: 192.168.100.123
http.port: 9200
transport.tcp.port: 9300
discovery.seed_hosts: ["192.168.100.123"]
cluster.initial_master_nodes: ["node-1"]
# 下面的是配置x-pack和tsl/ssl加密通信的
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

### 运行elasticsearch并初始化密码

先将es文件夹下的所有目录的所有权限迭代给esuser用户

```bash
chgrp -R esuser ./es
chown -R esuser ./es
chmod 777 es
```

使用./bin/elasticsearch -d 后台启动elasticsearch，去掉-d则是前端启动elasticsearch

然后./bin/elasticsearch-setup-passwords interactive 配置默认用户的密码：（有如下的交互），可以使用auto自动生成。

```bash
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [apm_system]: 
Reenter password for [apm_system]: 
Enter password for [kibana_system]: 
Reenter password for [kibana_system]: 
Enter password for [logstash_system]: 
Reenter password for [logstash_system]: 
Enter password for [beats_system]: 
Reenter password for [beats_system]: 
Enter password for [remote_monitoring_user]: 
Reenter password for [remote_monitoring_user]: 
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

然后可以登录<http://192.168.100.123:9200/> 需要输入密码，输入elastic/passwd即可登录

## kibana安装

下载kibana文件

```bash
#在home下面新建kibana文件夹
mkdir -p /home/kibana
# 修改文件夹权限为esuser
sudo chgrp -R esuser ./kibana
sudo chown -R esuser ./kibana
sudo chmod 777 kibana
# 下载文件
curl -L -O https://artifacts.elastic.co/downloads/kibana/kibana-7.10.2-linux-x86_64.tar.gz
# 解压文件
tar -xzvf kibana-7.10.2-linux-x86_64.tar.gz
```

修改kibana.yml

```bash
$ cat config/kibana.yml | egrep  -v "^$|#"

server.port: 5601
server.host: "0.0.0.0"
server.name: "my-kibana"
elasticsearch.hosts: ["http://192.168.100.123:9200"]
kibana.index: ".kibana"
elasticsearch.username: "elastic"
```

./bin/kibana  启动

访问网址：<http://192.168.100.123:5601/>  并使用elastic/password 登录

下载并解压并将文件移动到/opt/kafka目录下

```bash
# 下载
curl -L -O https://archive.apache.org/dist/kafka/2.1.0/kafka_2.12-2.1.0.tgz

tar -zxvf kafka_2.12-2.1.0.tgz

mv kafka_2.12-2.1.0 /opt/kafka

```

配置kafka

```bash
#创建日志存放目录
cd /opt/kafka
mkdir -p logs
#修改配置文件
vim /opt/kafka/config/server.properties

cat config/server.properties | egrep -v "^$|#"
broker.id=180
listeners=PLAINTEXT://192.168.100.124:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/opt/kafka/logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
```

配置环境变量

```bash
vi /etc/profile  #追加以下信息
export KAFKA_HOME=/opt/kafka
export PATH=$KAFKA_HOME/bin:$PATH
```

启动

```bash
cd /opt/kafka/bin
kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```
