# 使用 Elastic 技术栈构建 K8S 全栈监控

## 搭建ElasticSearch 集群环境

- 监控指标提供系统各个组件的时间序列数据，比如 CPU、内存、磁盘、网络等信息，通常可以用来显示系统的整体状况以及检测某个时间的异常行为
- 日志为运维人员提供了一个数据来分析系统的一些错误行为，通常将系统、服务和应用的日志集中收集在同一个数据库中
- 追踪或者 APM（应用性能监控）提供了一个更加详细的应用视图，可以将服务执行的每一个请求和步骤都记录下来（比如 HTTP 调用、数据库查询等），通过追踪这些数据，我们可以检测到服务的性能，并相应地改进或修复我们的系统。

本文我们就将在 Kubernetes 集群中使用由 ElasticSearch、Kibana、Filebeat、Metricbeat 和 APM-Server 组成的 Elastic 技术栈来监控系统环境。为了更好地去了解这些组件的配置，我们这里将采用手写资源清单文件的方式来安装这些组件，当然我们也可以使用 Helm 等其他工具来快速安装配置。

![img](https://i.loli.net/2021/07/28/NzhgvQDLs27oOBi.png)

接下来我们就来学习下如何使用 Elastic 技术构建 Kubernetes 监控栈，我们这里的试验环境是 Kubernetes v1.20.0 版本的集群，为方便管理，我们将所有的资源对象都部署在一个名为 elastic 的命名空间中：

```bash
kubectl create ns elastic
```

### 创建示例应用

这里我们先部署一个使用 SpringBoot 和 MongoDB 开发的示例应用。首先部署一个 MongoDB 应用，对应的资源清单文件如下所示：

```yaml
# mongo.yml
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: elastic
  labels:
    app: mongo
spec:
  ports:
  - port: 27017
    protocol: TCP
  selector:
    app: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: elastic
  name: mongo
  labels:
    app: mongo
spec:
  serviceName: "mongo"
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: managed-nfs-storage  # 使用支持 RWO 的 StorageClass
      resources:
        requests:
          storage: 1Gi
```

这里我们使用了一个名为managed-nfs-storage的storageclass对象来自动创建PV,关于StorageClass内容可以参考 [kubernetes共享存储](k8s/docs/kubernetes共享存储.md)

```bash
kubectl apply -f mongo.yaml
#查询pod状态
kubectl get pod -n elastic
```

直到 Pod 变成 Running 状态证明 mongodb 部署成功了。接下来部署 SpringBoot 的 API 应用，这里我们通过 NodePort 类型的 Service 服务来暴露该服务，对应的资源清单文件如下所示：

```yaml
# spring-boot-simple.yml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: spring-boot-simple
  labels:
    app: spring-boot-simple
spec:
  type: NodePort
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: spring-boot-simple
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: spring-boot-simple
  labels:
    app: spring-boot-simple
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-simple
  template:
    metadata:
      labels:
        app: spring-boot-simple
    spec:
      containers:
      - image: cnych/spring-boot-simple:0.0.1-SNAPSHOT
        name: spring-boot-simple
        env:
        - name: SPRING_DATA_MONGODB_HOST  # 指定MONGODB地址
          value: mongo
        ports:
        - containerPort: 8080
```

同样直接创建上面的应用的应用即可：

```bash
kubectl apply -f spring-boot-simple.yaml 
# check pod status
kubectl get pod -n elastic
# check service status
kubectl get svc -n elastic
```

应用部署完成后通过service cluster ip地址访问应用

```bash
[root@m01 ~]# curl -X GET http://172.19.221.29:8080
Greetings from Spring Boot!
```

发送一个POST请求

```bash
[root@m01 ~]# curl -X POST http://172.19.221.29:8080/message -d "hello world 20210728"
{"id":"610103694702120001f46c24","message":"hello+world+20210728=","postedAt":"2021-07-28T07:12:41.226+0000"}
```

获取最新消息数据

```bash
[root@m01 ~]# curl -X GET http://172.19.221.29:8080/message
[{"id":"610103694702120001f46c24","message":"hello+world+20210728=","postedAt":"2021-07-28T07:12:41.226+0000"}]
```

### ElasticSearch 集群

要建立一个 Elastic 技术的监控栈，当然首先我们需要部署 ElasticSearch，它是用来存储所有的指标、日志和追踪的数据库，这里我们通过3个不同角色的可扩展的节点组成一个集群。

#### 安装 ElasticSearch 主节点

设置集群的第一个节点为 Master 主节点，来负责控制整个集群。首先创建一个 ConfigMap 对象，用来描述集群的一些配置信息，以方便将 ElasticSearch 的主节点配置到集群中并开启安全认证功能。对应的资源清单文件如下所示：

```yaml
# elasticsearch-master.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: elasticsearch-master-config
  labels:
    app: elasticsearch
    role: master
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: true
      data: false
      ingest: false

    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
---
```

然后创建一个 Service 对象，在 Master 节点下，我们只需要通过用于集群通信的 9300 端口进行通信。资源清单文件如下所示：

```yaml
# elasticsearch-master.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: elasticsearch-master
  labels:
    app: elasticsearch
    role: master
spec:
  ports:
  - port: 9300
    name: transport
  selector:
    app: elasticsearch
    role: master
---
```

最后使用一个 Deployment 对象来定义 Master 节点应用，资源清单文件如下所示：

```yaml
# elasticsearch-master.deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: elasticsearch-master
  labels:
    app: elasticsearch
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
      role: master
  template:
    metadata:
      labels:
        app: elasticsearch
        role: master
    spec:
      containers:
      - name: elasticsearch-master
        image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-master
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms512m -Xmx512m"
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: storage
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: elasticsearch-master-config
      - name: "storage"
        emptyDir:
          medium: ""
---
```

直接创建上面的资源对象即可：

```bash
kubectl apply -f elasticsearch-master.configmap.yaml \
              -f elasticsearch-master.service.yaml \
              -f elasticsearch-master.deployment.yaml
# 直到pod变成Running状态证明节点启动成功
kubectl get pods -n elastic -l app=elasticsearch
```

#### 安装 ElasticSearch 数据节点

现在我们需要安装的是集群的数据节点，它主要来负责集群的数据托管和执行查询。

和 master 节点一样，我们使用一个 ConfigMap 对象来配置我们的数据节点：

```yaml
# elasticsearch-data.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: elasticsearch-data-config
  labels:
    app: elasticsearch
    role: data
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: false
      data: true
      ingest: false

    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
---
```

可以看到和上面的 master 配置非常类似，不过需要注意的是属性 node.data=true。

同样只需要通过 9300 端口和其他节点进行通信：

```yaml
# elasticsearch-data.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: elasticsearch-data
  labels:
    app: elasticsearch
    role: data
spec:
  ports:
  - port: 9300
    name: transport
  selector:
    app: elasticsearch
    role: data
---
```

最后创建一个 StatefulSet 的控制器，因为可能会有多个数据节点，每一个节点的数据不是一样的，需要单独存储，所以也使用了一个 volumeClaimTemplates 来分别创建存储卷，对应的资源清单文件如下所示：

```yaml
# elasticsearch-data.statefulset.yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: elastic
  name: elasticsearch-data
  labels:
    app: elasticsearch
    role: data
spec:
  serviceName: "elasticsearch-data"
  selector:
    matchLabels:
      app: elasticsearch
      role: data
  template:
    metadata:
      labels:
        app: elasticsearch
        role: data
    spec:
      containers:
      - name: elasticsearch-data
        image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-data
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms1024m -Xmx1024m"
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: elasticsearch-data-persistent-storage
          mountPath: /data/db
      volumes:
      - name: config
        configMap:
          name: elasticsearch-data-config
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: managed-nfs-storage
      resources:
        requests:
          storage: 20Gi
---
```

#### 安装 ElasticSearch 客户端节点

最后来安装配置 ElasticSearch 的客户端节点，该节点主要负责暴露一个 HTTP 接口将查询数据传递给数据节点获取数据。

同样使用一个 ConfigMap 对象来配置该节点：

```yaml
# elasticsearch-client.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: elasticsearch-client-config
  labels:
    app: elasticsearch
    role: client
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: false
      data: false
      ingest: true

    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
---
```

客户端节点需要暴露两个端口，9300端口用于与集群的其他节点进行通信，9200端口用于 HTTP API。对应的 Service 对象如下所示：

```yaml
# elasticsearch-client.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: elasticsearch-client
  labels:
    app: elasticsearch
    role: client
spec:
  ports:
  - port: 9200
    name: client
  - port: 9300
    name: transport
  selector:
    app: elasticsearch
    role: client
---
```

使用一个 Deployment 对象来描述客户端节点：

```yaml
# elasticsearch-client.deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: elasticsearch-client
  labels:
    app: elasticsearch
    role: client
spec:
  selector:
    matchLabels:
      app: elasticsearch
      role: client
  template:
    metadata:
      labels:
        app: elasticsearch
        role: client
    spec:
      containers:
      - name: elasticsearch-client
        image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-client
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms256m -Xmx256m"
        ports:
        - containerPort: 9200
          name: client
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: storage
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: elasticsearch-client-config
      - name: "storage"
        emptyDir:
          medium: ""
---
```

直到所有的节点都部署成功后证明集群安装成功：

```bash
[root@cli elastic]# kubectl get pods -n elastic -l app=elasticsearch
NAME                                   READY   STATUS    RESTARTS   AGE
elasticsearch-client-5946bfc97-7wv8g   1/1     Running   0          102s
elasticsearch-data-0                   1/1     Running   7          33m
elasticsearch-master-74b55c585-4rc9p   1/1     Running   11         54m

```

可以通过如下所示的命令来查看集群的状态变化

```bash
[root@cli elastic]# kubectl logs -f -n elastic   $(kubectl get pods -n elastic | grep elasticsearch-master | sed -n 1p | awk '{print $1}')   | grep "Cluster health status changed from"
{"type": "server", "timestamp": "2021-07-28T08:51:14,967Z", "level": "INFO", "component": "o.e.c.r.a.AllocationService", "cluster.name": "elasticsearch", "node.name": "elasticsearch-master", "message": "Cluster health status changed from [YELLOW] to [GREEN] (reason: [shards started [[.monitoring-es-7-2021.07.28][0]]]).", "cluster.uuid": "vfceLLsMRmCaEQyIZer7Og", "node.id": "r5Of68kKSfCALxgxe2M-9g"  }

```

#### 生成密码

我们启用了 xpack 安全模块来保护我们的集群，所以我们需要一个初始化的密码。我们可以执行如下所示的命令，在客户端节点容器内运行 bin/elasticsearch-setup-passwords 命令来生成默认的用户名和密码：

```bash
[root@cli elastic]# kubectl exec $(kubectl get pods -n elastic | grep elasticsearch-client | sed -n 1p | awk '{print $1}') -n elastic -- bin/elasticsearch-setup-passwords auto -b
Changed password for user apm_system
PASSWORD apm_system = 7C6t9k2mSR3PRryKXkZm

Changed password for user kibana_system
PASSWORD kibana_system = T2vHJgGHsUvqDFi9c8cK

Changed password for user kibana
PASSWORD kibana = T2vHJgGHsUvqDFi9c8cK

Changed password for user logstash_system
PASSWORD logstash_system = WY4hPjxXbdNlvvwZNR2z

Changed password for user beats_system
PASSWORD beats_system = gn06fMTJdOX0nUMFOGsU

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = DvGEt0GWGEQBAFl0BC3b

Changed password for user elastic
PASSWORD elastic = B0a39le69ETFASqU8uFK

```

注意需要将 elastic 用户名和密码也添加到 Kubernetes 的 Secret 对象中

```bash
[root@cli elastic]# kubectl create secret generic elasticsearch-pw-kibanasystem -n elastic --from-literal password=T2vHJgGHsUvqDFi9c8cK
secret/elasticsearch-pw-kibanasystem created
```

#### 安装Kibana

ElasticSearch 集群安装完成后，接着我们可以来部署 Kibana，这是 ElasticSearch 的数据可视化工具，它提供了管理 ElasticSearch 集群和可视化数据的各种功能。

同样首先我们使用 ConfigMap 对象来提供一个文件文件，其中包括对 ElasticSearch 的访问（主机、用户名和密码），这些都是通过环境变量配置的。对应的资源清单文件如下所示：

```yaml
# kibana.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: kibana-config
  labels:
    app: kibana
data:
  kibana.yml: |-
    server.host: 0.0.0.0

    elasticsearch:
      hosts: ${ELASTICSEARCH_HOSTS}
      username: ${ELASTICSEARCH_USER}
      password: ${ELASTICSEARCH_PASSWORD}
---
```

然后通过一个 NodePort 类型的服务来暴露 Kibana 服务：

```yaml
# kibana.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: kibana
  labels:
    app: kibana
spec:
  type: NodePort
  ports:
  - port: 5601
    name: webinterface
  selector:
    app: kibana
---
```

最后通过 Deployment 来部署 Kibana 服务，由于需要通过环境变量提供密码，这里我们使用上面创建的 Secret 对象来引用：

```yaml
# kibana.deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: kibana
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.13.4
        ports:
        - containerPort: 5601
          name: webinterface
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch-client.elastic.svc.cluster.local:9200"
        - name: ELASTICSEARCH_USER
          value: "kibana_system"
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-pw-kibanasystem
              key: password
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          readOnly: true
          subPath: kibana.yml
      volumes:
      - name: config
        configMap:
          name: kibana-config
---
```

当状态变成 running 后，我们就可以通过 NodePort 端口 30472 去浏览器中访问 Kibana 服务了.

```bash
[root@cli elastic]# kubectl get svc kibana -n elastic 
NAME     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kibana   NodePort   172.19.219.224   <none>        5601:30472/TCP   21h
```

如下图所示，使用elastic用户名和密码即可登录

![img](https://i.loli.net/2021/07/29/K2GF6d8aAtmLh9f.png)

最后还可以通过 Management → Stack Monitoring 页面查看整个集群的健康状态：

![img](https://i.loli.net/2021/07/29/Iwniu5bD24o7Hpq.png)

到这里我们就安装成功了 ElasticSearch 与 Kibana，它们将为我们来存储和可视化我们的应用数据（监控指标、日志和追踪）服务。

我们将来学习如何安装和配置 Metricbeat，通过 Elastic Metribeat 来收集指标监控 Kubernetes 集群。

## 使用 Metricbeat 对 Kubernetes 集群进行监控

前面我们已经安装配置了 ElasticSearch 的集群，本文我们将来使用 Metricbeat 对 Kubernetes 集群进行监控。Metricbeat 是一个服务器上的轻量级采集器，用于定期收集主机和服务的监控指标。这也是我们构建 Kubernetes 全栈监控的第一个部分。

Metribeat 默认采集系统的指标，但是也包含了大量的其他模块来采集有关服务的指标，比如 Nginx、Kafka、MySQL、Redis 等等，支持的完整模块可以在 Elastic 官方网站上查看到 <https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-modules.html>。

### kube-state-metrics

首先，我们需要安装 kube-state-metrics，这个组件是一个监听 Kubernetes API 的服务，可以暴露每个资源对象状态的相关指标数据。

要安装 kube-state-metrics 也非常简单，在对应的 GitHub 仓库下就有对应的安装资源清单文件：

```bash
git clone https://github.com/kubernetes/kube-state-metrics.git
cd kube-state-metrics/
#执行部署
[root@cli kube-state-metrics]# kubectl apply -f examples/standard/
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
service/kube-state-metrics created
# 查看pod状态，当Pod变成running状态后证明安装成功
[root@cli kube-state-metrics]# kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-state-metrics
NAME                                 READY   STATUS    RESTARTS   AGE
kube-state-metrics-b967d9648-rsrl4   1/1     Running   0          3m51s
```

### Metricbeat

由于我们需要监控所有的节点，所以我们需要使用一个 DaemonSet 控制器来安装 Metricbeat。

首先，使用一个 ConfigMap 来配置 Metricbeat，然后通过 Volume 将该对象挂载到容器中的 /etc/metricbeat.yaml 中去。配置文件中包含了 ElasticSearch 的地址、用户名和密码，以及 Kibana 配置，我们要启用的模块与抓取频率等信息。

```yaml
# metricbeat.settings.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: metricbeat-config
  labels:
    app: metricbeat
data:
  metricbeat.yml: |-

    # 模块配置
    metricbeat.modules:
    - module: system
      period: ${PERIOD}
      metricsets: ["cpu", "load", "memory", "network", "process", "process_summary", "core", "diskio", "socket"]
      processes: ['.*']
      process.include_top_n:
        by_cpu: 5      # 根据 CPU 计算的前5个进程
        by_memory: 5   # 根据内存计算的前5个进程

    - module: system
      period: ${PERIOD}
      metricsets:  ["filesystem", "fsstat"]
      processors:
      - drop_event.when.regexp:
          system.filesystem.mount_point: '^/(sys|cgroup|proc|dev|etc|host|lib)($|/)'

    - module: docker
      period: ${PERIOD}
      hosts: ["unix:///var/run/docker.sock"]
      metricsets: ["container", "cpu", "diskio", "healthcheck", "info", "memory", "network"]

    - module: kubernetes  # 抓取 kubelet 监控指标
      period: ${PERIOD}
      node: ${NODE_NAME}
      hosts: ["https://${NODE_NAME}:10250"]
      metricsets: ["node", "system", "pod", "container", "volume"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
     
    - module: kubernetes  # 抓取 kube-state-metrics 数据
      period: ${PERIOD}
      node: ${NODE_NAME}
      metricsets: ["state_node", "state_deployment", "state_replicaset", "state_pod", "state_container"]
      hosts: ["kube-state-metrics.kube-system.svc.cluster.local:8080"]
    
    - module: kubernetes # Kubernetes proxy server
      period: ${PERIOD}
      node: ${NODE_NAME}
      metricsets: ["proxy"]
      hosts: ["http://localhost:10252"]

    # 根据 k8s deployment 配置具体的服务模块
    metricbeat.autodiscover:
      providers:
      - type: kubernetes
        node: ${NODE_NAME}
        templates:
        - condition.equals:
            kubernetes.labels.app: mongo
          config:
          - module: mongodb
            period: ${PERIOD}
            hosts: ["mongo.elastic:27017"]
            metricsets: ["dbstats", "status", "collstats", "metrics", "replstatus"]

    # ElasticSearch 连接配置
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}

    # 连接到 Kibana
    setup.kibana:
      host: '${KIBANA_HOST:kibana}:${KIBANA_PORT:5601}'

    # 导入已经存在的 Dashboard
    setup.dashboards.enabled: true

    # 配置 indice 生命周期
    setup.ilm:
      policy_file: /etc/indice-lifecycle.json
---
```

ElasticSearch 的 indice 生命周期表示一组规则，可以根据 indice 的大小或者时长应用到你的 indice 上。比如可以每天或者每次超过 1GB 大小的时候对 indice 进行轮转，我们也可以根据规则配置不同的阶段。由于监控会产生大量的数据，很有可能一天就超过几十G的数据，所以为了防止大量的数据存储，我们可以利用 indice 的生命周期来配置数据保留，这个在 Prometheus 中也有类似的操作。

如下所示的文件中，我们配置成每天或每次超过5GB的时候就对 indice 进行轮转，并删除所有超过10天的 indice 文件，我们这里只保留10天监控数据完全足够了。

```yaml
# metricbeat.indice-lifecycle.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: metricbeat-indice-lifecycle
  labels:
    app: metricbeat
data:
  indice-lifecycle.json: |-
    {
      "policy": {
        "phases": {
          "hot": {
            "actions": {
              "rollover": {
                "max_size": "5GB" ,
                "max_age": "1d"
              }
            }
          },
          "delete": {
            "min_age": "10d",
            "actions": {
              "delete": {}
            }
          }
        }
      }
    }
---
```

接下来就可以来编写 Metricbeat 的 DaemonSet 资源对象清单，如下所示：

```yaml
# metricbeat.daemonset.yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace: elastic
  name: metricbeat
  labels:
    app: metricbeat
spec:
  selector:
    matchLabels:
      app: metricbeat
  template:
    metadata:
      labels:
        app: metricbeat
    spec:
      serviceAccountName: metricbeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:7.13.4
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e", "-system.hostfs=/hostfs"
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-client.elastic.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-pw-elastic
              key: password
        - name: KIBANA_HOST
          value: kibana.elastic.svc.cluster.local
        - name: KIBANA_PORT
          value: "5601"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: PERIOD
          value: "10s"
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          readOnly: true
          subPath: metricbeat.yml
        - name: indice-lifecycle
          mountPath: /etc/indice-lifecycle.json
          readOnly: true
          subPath: indice-lifecycle.json
        - name: dockersock
          mountPath: /var/run/docker.sock
        - name: proc
          mountPath: /hostfs/proc
          readOnly: true
        - name: cgroup
          mountPath: /hostfs/sys/fs/cgroup
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: cgroup
        hostPath:
          path: /sys/fs/cgroup
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: config
        configMap:
          defaultMode: 0600
          name: metricbeat-config
      - name: indice-lifecycle
        configMap:
          defaultMode: 0600
          name: metricbeat-indice-lifecycle
      - name: data
        hostPath:
          path: /var/lib/metricbeat-data
          type: DirectoryOrCreate
---
```

需要注意的将上面的两个 ConfigMap 挂载到容器中去，由于需要 Metricbeat 获取宿主机的相关信息，所以我们这里也挂载了一些宿主机的文件到容器中去，比如 proc 目录，cgroup 目录以及 dockersock 文件。

由于 Metricbeat 需要去获取 Kubernetes 集群的资源对象信息，所以同样需要对应的 RBAC 权限声明，由于是全局作用域的，所以这里我们使用 ClusterRole 进行声明：

```yaml
# metricbeat.permissions.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: elastic
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metricbeat
  labels:
    app: metricbeat
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - events
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: elastic
  name: metricbeat
  labels:
    app: metricbeat
---
```

直接创建上面的几个资源对象即可：

当 Metricbeat 的 Pod 变成 Running 状态后，正常我们就可以在 Kibana 中去查看对应的监控信息了

```bash
[root@cli elastic]# kubectl get pods -n elastic -l app=metricbeat
NAME               READY   STATUS    RESTARTS   AGE
metricbeat-2qwcf   1/1     Running   0          61m
metricbeat-vkzk5   1/1     Running   0          61m

```

在 Kibana 左侧页面 Observability → Metrics 进入指标监控页面，正常就可以看到一些监控数据了：

![img](https://i.loli.net/2021/07/30/u3fOgNYJFRx1ob4.png)

也可以根据自己的需求进行筛选，比如我们可以按照 Kubernetes Namespace 进行分组作为视图查看监控信息：

![img](https://i.loli.net/2021/07/30/Zv3MeD7yjXnk2FI.png)

由于我们在配置文件中设置了属性 setup.dashboards.enabled=true，所以 Kibana 会导入预先已经存在的一些 Dashboard。我们可以在左侧菜单进入 Kibana → Dashboard 页面，我们会看到一个大约有 50 个 Metricbeat 的 Dashboard 列表，我们可以根据需要筛选 Dashboard，比如我们要查看集群节点的信息，可以查看 [Metricbeat Kubernetes] Overview ECS 这个 Dashboard：

![img](https://i.loli.net/2021/07/30/FLfmO9gBRGpJPVH.png)

我们还单独启用了 mongodb 模块，我们可以使用 [Metricbeat MongoDB] Overview ECS 这个 Dashboard 来查看监控信息：

![img](https://i.loli.net/2021/07/30/cMBlz2PGkeSDZ8v.png)

我们还启用了 docker 这个模块，也可以使用 [Metricbeat Docker] Overview ECS 这个 Dashboard 来查看监控信息：

![img](https://i.loli.net/2021/07/30/AKSUcIB1PsfxoEq.png)

到这里我们就完成了使用 Metricbeat 来监控 Kubernetes 集群信息，在下文我们再来学习如何使用 Filebeat 来收集日志以监控 Kubernetes 集群。

## 使用 Filebeat 采集 Kubernetes 集群日志

我们将要安装配置 Filebeat 来收集 Kubernetes 集群中的日志数据，然后发送到 ElasticSearch 去中，Filebeat 是一个轻量级的日志采集代理，还可以配置特定的模块来解析和可视化应用（比如数据库、Nginx 等）的日志格式。

和 Metricbeat 类似，Filebeat 也需要一个配置文件来设置和 ElasticSearch 的链接信息、和 Kibana 的连接已经日志采集和解析的方式。

如下所示的 ConfigMap 资源对象就是我们这里用于日志采集的配置信息。

```yaml
# filebeat.settings.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: filebeat-config
  labels:
    app: filebeat
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      enabled: true
      paths:
      - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          in_cluster: true
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
    
    filebeat.autodiscover:
      providers:
        - type: kubernetes
          templates:
            - condition.equals:
                kubernetes.labels.app: mongo
              config:
                - module: mongodb
                  enabled: true
                  log:
                    input:
                      type: docker
                      containers.ids:
                        - ${data.kubernetes.container.id}

    processors:
      - drop_event:
          when.or:
              - and:
                  - regexp:
                      message: '^\d+\.\d+\.\d+\.\d+ '
                  - equals:
                      fileset.name: error
              - and:
                  - not:
                      regexp:
                          message: '^\d+\.\d+\.\d+\.\d+ '
                  - equals:
                      fileset.name: access
      - add_cloud_metadata:
      - add_kubernetes_metadata:
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
      - add_docker_metadata:

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}

    setup.kibana:
      host: '${KIBANA_HOST:kibana}:${KIBANA_PORT:5601}'

    setup.dashboards.enabled: true
    setup.template.enabled: true

    setup.ilm:
      policy_file: /etc/indice-lifecycle.json
---
```

我们配置采集 /var/log/containers/ 下面的所有日志数据，并且使用 inCluster 的模式访问 Kubernetes 的 APIServer，获取日志数据的 Meta 信息，将日志直接发送到 Elasticsearch。

此外还通过 policy_file 定义了 indice 的回收策略：

```yaml
# filebeat.indice-lifecycle.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: filebeat-indice-lifecycle
  labels:
    app: filebeat
data:
  indice-lifecycle.json: |-
    {
      "policy": {
        "phases": {
          "hot": {
            "actions": {
              "rollover": {
                "max_size": "5GB" ,
                "max_age": "1d"
              }
            }
          },
          "delete": {
            "min_age": "30d",
            "actions": {
              "delete": {}
            }
          }
        }
      }
    }
---
```

同样为了采集每个节点上的日志数据，我们这里使用一个 DaemonSet 控制器，使用上面的配置来采集节点的日志。

```yaml
#filebeat.daemonset.yml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace: elastic
  name: filebeat
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.13.4
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-client.elastic.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-pw-elastic
              key: password
        - name: KIBANA_HOST
          value: kibana.elastic.svc.cluster.local
        - name: KIBANA_PORT
          value: "5601"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: filebeat-indice-lifecycle
          mountPath: /etc/indice-lifecycle.json
          readOnly: true
          subPath: indice-lifecycle.json
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: dockersock
          mountPath: /var/run/docker.sock
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: filebeat-indice-lifecycle
        configMap:
          defaultMode: 0600
          name: filebeat-indice-lifecycle
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
```

我们这里使用的是 Kubeadm 搭建的集群，默认 Master 节点是有污点的，所以如果还想采集 Master 节点的日志，还必须加上对应的容忍，我这里不采集就没有添加容忍了。

此外由于需要获取日志在 Kubernetes 集群中的 Meta 信息，比如 Pod 名称、所在的命名空间等，所以 Filebeat 需要访问 APIServer，自然就需要对应的 RBAC 权限了，所以还需要进行权限声明：

```yaml
# filebeat.permission.yml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: elastic
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    app: filebeat
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: elastic
  name: filebeat
  labels:
    app: filebeat
---
```

然后直接安装部署上面的几个资源对象即可：

```bash
[root@cli elastic]# kubectl apply -f filebeat.settings.configmap.yaml \
                                  -f filebeat.indice-lifecycle.configmap.yaml \
                                  -f filebeat.daemonset.yaml \
                                  -f filebeat.permission.yaml 
configmap/filebeat-config created
configmap/filebeat-indice-lifecycle created
daemonset.apps/filebeat created
clusterrolebinding.rbac.authorization.k8s.io/filebeat created
clusterrole.rbac.authorization.k8s.io/filebeat created
serviceaccount/filebeat created
```

当所有的 Filebeat 的 Pod 都变成 Running 状态后，证明部署完成。现在我们就可以进入到 Kibana 页面中去查看日志了。左侧菜单 Observability → Logs

![img](https://i.loli.net/2021/08/02/4VCmw6BlMu2ofQW.png)

## 使用 Elastic APM 实时监控应用性能

Elastic APM 是 Elastic Stack 上用于应用性能监控的工具，它允许我们通过收集传入请求、数据库查询、缓存调用等方式来实时监控应用性能。这可以让我们更加轻松快速定位性能问题。

Elastic APM 是兼容 OpenTracing 的，所以我们可以使用大量现有的库来跟踪应用程序性能。

比如我们可以在一个分布式环境（微服务架构）中跟踪一个请求，并轻松找到可能潜在的性能瓶颈。

![img](https://i.loli.net/2021/08/06/hHz7ArY3L2dxpVU.png)

Elastic APM 通过一个名为 APM-Server 的组件提供服务，用于收集并向 ElasticSearch 以及和应用一起运行的 agent 程序发送追踪数据。

![img](https://i.loli.net/2021/08/06/cpeFlKoZMbhstYx.png)

### 安装 APM-Server

首先我们需要在 Kubernetes 集群上安装 APM-Server 来收集 agent 的追踪数据，并转发给 ElasticSearch，这里同样我们使用一个 ConfigMap 来配置：

```yaml
# apm.configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: elastic
  name: apm-server-config
  labels:
    app: apm-server
data:
  apm-server.yml: |-
    apm-server:
      host: "0.0.0.0:8200"

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}

    setup.kibana:
      host: '${KIBANA_HOST:kibana}:${KIBANA_PORT:5601}'
---
```

APM-Server 需要暴露 8200 端口来让 agent 转发他们的追踪数据，新建一个对应的 Service 对象即可：

```yaml
# apm.service.yaml
---
apiVersion: v1
kind: Service
metadata:
  namespace: elastic
  name: apm-server
  labels:
    app: apm-server
spec:
  ports:
  - port: 8200
    name: apm-server
  selector:
    app: apm-server
---
```

然后使用一个 Deployment 资源对象管理即可：

```yaml
# apm.deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: apm-server
  labels:
    app: apm-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apm-server
  template:
    metadata:
      labels:
        app: apm-server
    spec:
      containers:
      - name: apm-server
        image: docker.elastic.co/apm/apm-server:7.13.4
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch-client.elastic.svc.cluster.local
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-pw-elastic
              key: password
        - name: KIBANA_HOST
          value: kibana.elastic.svc.cluster.local
        - name: KIBANA_PORT
          value: "5601"
        ports:
        - containerPort: 8200
          name: apm-server
        volumeMounts:
        - name: config
          mountPath: /usr/share/apm-server/apm-server.yml
          readOnly: true
          subPath: apm-server.yml
      volumes:
      - name: config
        configMap:
          name: apm-server-config
---
```

直接部署上面的几个资源对象，当Pod处于Running状态证明运行成功。

```bash
[root@cli elastic]# kubectl get pods -n elastic -l app=apm-server
NAME                         READY   STATUS    RESTARTS   AGE
apm-server-98446795d-dgnc5   1/1     Running   0          2m53s
```

### 配置Java Agent

接下来我们在示例应用程序 spring-boot-simple 上配置一个 [Elastic APM Java agent](https://repo1.maven.org/maven2/co/elastic/apm/elastic-apm-agent/)。

首先我们需要把 elastic-apm-agent-1.8.0.jar 这个 jar 包程序内置到应用容器中去，在构建镜像的 Dockerfile 文件中添加一行如下所示的命令直接下载该 JAR 包即可：

```bash
RUN wget -O /apm-agent.jar https://search.maven.org/remotecontent?filepath=co/elastic/apm/elastic-apm-agent/1.8.0/elastic-apm-agent-1.8.0.jar
```

完整的 Dockerfile 文件如下所示：

```bash
FROM openjdk:8-jdk-alpine

ENV ELASTIC_APM_VERSION "1.8.0"
RUN wget -O /apm-agent.jar https://search.maven.org/remotecontent?filepath=co/elastic/apm/elastic-apm-agent/$ELASTIC_APM_VERSION/elastic-apm-agent-$ELASTIC_APM_VERSION.jar

COPY target/spring-boot-simple.jar /app.jar

CMD java -jar /app.jar
```

然后需要修改第一篇文章中使用 Deployment 部署的 Spring-Boot 应用，需要开启 Java agent 并且要连接到 APM-Server。

```yaml
# spring-boot-simple.deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: spring-boot-simple
  labels:
    app: spring-boot-simple
spec:
  selector:
    matchLabels:
      app: spring-boot-simple
  template:
    metadata:
      labels:
        app: spring-boot-simple
    spec:
      containers:
      - image: cnych/spring-boot-simple:0.0.1-SNAPSHOT
        imagePullPolicy: Always
        name: spring-boot-simple
        command:  #修改部分
          - "java"
          - "-javaagent:/apm-agent.jar"
          - "-Delastic.apm.active=$(ELASTIC_APM_ACTIVE)"
          - "-Delastic.apm.server_urls=$(ELASTIC_APM_SERVER)"
          - "-Delastic.apm.service_name=spring-boot-simple"
          - "-jar"
          - "app.jar"
        env:
          - name: SPRING_DATA_MONGODB_HOST
            value: mongo
          - name: ELASTIC_APM_ACTIVE
            value: "true"
          - name: ELASTIC_APM_SERVER
            value: http://apm-server.elastic.svc.cluster.local:8200 #修改部分
        ports:
        - containerPort: 8080
---
```

然后重新部署上面的示例应用：

当示例应用重新部署完成后，执行如下几个请求：

get messages

获取所有发布的 messages 数据：

```bash
curl -X GET http://192.168.100.194:31847/message
```

get messages (慢请求)

使用 sleep= ms 来模拟慢请求：

```bash
curl -X GET http://192.168.100.194:31847/message?sleep=3000
```

get messages (error)

使用 error=true 来触发一异常：

```bash
curl -X GET http://192.168.100.194:31847/message?error=true
```

现在我们去到 Kibana 页面中路由到 APM 页面，我们应该就可以看到 spring-boot-simple 应用的数据了。
