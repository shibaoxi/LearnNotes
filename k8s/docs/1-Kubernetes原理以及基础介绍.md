# Kubernetes原理以及基础介绍

## Kubernetes群集组件

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/Kubernetes%20cluster2.png)

云上的kubernetes群集组件（Azure）

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/control-plane-and-nodes.png)

下面分别去介绍他们

### Kubernetes控制平面

#### 1. etcd分布式持久化存储

> etcd 是一个相应快、分布式、一致的key-value存储。唯一能直接和etcd通信的是Kubernetes的API服务器。
为了保证高可用性，常常会运行多个etcd实例。etcd使用RAFT一致性算法来保障分布式系统需要对系统的实际状态达成一致。

#### 2. API服务器

> Kubernetes API 服务器作为中心组件，其他组件或者客户端(kubectl)都会去调用它，并且以RESTful API的形式提供了可以查询、修改集群状态的CRUD(Create、Read、Update、Delete)接口，并将状态存储到etcd中。

下图展示，API服务器接到kubectl请求后内部发生了什么：

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210117120112.png)

#### 3. 调度器

> 调度器就是利用API服务器的监听机制等待新的创建pod，然后给每个新的、没有节点集的pod分配节点。

**默认的调度算法：**

* 过滤所有的节点，找出能分配给pod的可用节点列表
* 对可用节点按优先级排序，找出最优节点。如果多个节点都有最高的优先级分数，那么则循环分配，确保平均分配给pod。

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210117123908.png)

#### 4. 控制器

控制器包含如下：

* Replication管理器（ReplicationController 资源管理器）
* ReplicaSet、DaemonSet和Job控制器
* Deployment控制器
* StatefulSet控制器
* Node控制器
* Endpoints控制器
* Namespace控制器
* PersistentVolume控制器
* 其他

#### 5. Kubelet

> Kubelet以及Service Proxy都运行在工作节点。

* 它的第一个任务是在API服务器中创建一个Node资源来注册该节点，然后持续监控API服务器是否把该节点分配给pod，然后启动pod容器。

* 它的第二个任务是运行容器存活探针，当探针报错时它会重启容器。

* 最后一点时当pod从API服务器删除时，kubelet终止容器，并通知服务器pod已经被终止。

#### 6. Kubernetes Service Proxy

> 每个工作节点还会运行kube-proxy，用于确保客户端可以通过Kubernetes API 连接到你定义的服务。

#### 7. Deployment资源提交到API服务器的事件链

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210118140131.png)
