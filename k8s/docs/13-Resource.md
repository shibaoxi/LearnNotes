# kubernetes 资源管理概述

## 资源简介

在 kubernetes 中，有两个基础但是非常重要的概念：node 和 pod。node 翻译成节点，是对集群资源的抽象；pod 是对容器的封装，是应用运行的实体。node 提供资源，而 pod 使用资源，这里的资源分为计算（cpu、memory、gpu）、存储（disk、ssd）、网络（network bandwidth、ip、ports）。这些资源提供了应用运行的基础，正确理解这些资源以及集群调度如何使用这些资源，对于大规模的 kubernetes 集群来说至关重要，不仅能保证应用的稳定性，也可以提高资源的利用率。

在 kubernetes 中，有两个基础但是非常重要的概念：node 和 pod。node 翻译成节点，是对集群资源的抽象；pod 是对容器的封装，是应用运行的实体。node 提供资源，而 pod 使用资源，这里的资源分为计算（cpu、memory、gpu）、存储（disk、ssd）、网络（network bandwidth、ip、ports）。这些资源提供了应用运行的基础，正确理解这些资源以及集群调度如何使用这些资源，对于大规模的 kubernetes 集群来说至关重要，不仅能保证应用的稳定性，也可以提高资源的利用率。

![img](https://i.loli.net/2021/07/19/ORHh7l6FK2Lb5nM.png)

## 节点可用资源

理想情况下，我们希望节点上所有的资源都可以分配给 pod 使用，但实际上节点上除了运行 pods 之外，还会运行其他的很多进程：系统相关的进程（比如 sshd、udev等），以及 kubernetes 集群的组件（kubelet、docker等）。我们在分配资源的时候，需要给这些进程预留一些资源，剩下的才能给 pod 使用。预留的资源可以通过下面的参数控制：

* ```--kube-reserved=[cpu=100m][,][memory=100Mi][,][ephemeral-storage=1Gi]```：控制预留给 kubernetes 集群组件的 CPU、memory 和存储资源

* ```--system-reserved=[cpu=100mi][,][memory=100Mi][,][ephemeral-storage=1Gi]```：预留给系统的 CPU、memory 和存储资源

这两块预留之后的资源才是 pod 真正能使用的，不过考虑到 eviction 机制，kubelet 会保证节点上的资源使用率不会真正到 100%，因此 pod 的实际可使用资源会稍微再少一点。主机上的资源逻辑分配图如下所示：

![img](https://i.loli.net/2021/07/19/Dvi7OjBEN583Wfl.png)

## kubernetes 资源对象

### 请求(requests)和上限(limits)

前面说过用户在创建 pod 的时候，可以指定每个容器的 Requests 和 Limits 两个字段，下面是一个实例：

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

> **NOTE**:如果 limit 没有配置，则表明没有资源的上限，只要节点上有对应的资源，pod 就可以使用。NOTE：如果 limit 没有配置，则表明没有资源的上限，只要节点上有对应的资源，pod 就可以使用。

### limit range (默认资源配置)

为每个 pod 都手动配置这些参数是挺麻烦的事情，kubernetes 提供了 LimitRange 资源，可以让我们配置某个 namespace 默认的 request 和 limit 值，比如下面的示例：

```yaml
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: you-shall-have-limits
spec:
  limits:
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
      default:
        cpu: "500m"
        memory: "200Mi"
      defaultRequest:
        cpu: "200m"
        memory: "100Mi"
```

关联namespace方式

```bash
kubectl apply -f limitrange.yaml --namespace=demo-ns
```

如果对应 namespace 创建的 pod 没有写资源的 requests 和 limits 字段，那么它会自动拥有下面的配置信息：

* 内存请求是 100Mi，上限是 200Mi
* CPU 请求是 200m，上限是 500m

当然，如果 pod 自己配置了对应的参数，kubernetes 会使用 pod 中的配置。使用 LimitRange 能够让 namespace 中的 pod 资源规范化，便于统一的资源管理。

### 资源配额(resource quota)

基于 namespace，kubernetes 还能够对资源进行隔离和限制，这就是 resource quota 的概念，翻译成资源配额，它限制了某个 namespace 可以使用的资源总额度。这里的资源包括 cpu、memory 的总量，也包括 kubernetes 自身对象（比如 pod、services 等）的数量。通过 resource quota，kubernetes 可以防止某个 namespace 下的用户不加限制地使用超过期望的资源，比如说不对资源进行评估就大量申请 16核 CPU 32G内存的 pod。

下面是一个资源配额的示例，它限制了 namespace 只能使用 20核 CPU 和 1G 内存，并且能创建 10 个 pod、20个 rc、5个 service，可能适用于某个测试场景。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    cpu: "20"
    memory: 1Gi
    pods: "10"
    replicationcontrollers: "20"
    resourcequotas: "1"
    services: "5"
```

### QoS (服务质量)

Requests 和 limits 的配置除了表明资源情况和限制资源使用之外，还有一个隐藏的作用：它决定了 pod 的 QoS 等级。
kubernetes 把 pod 分成了三个 QoS 等级：

* **Guaranteed**：优先级最高，可以考虑数据库应用或者一些重要的业务应用。除非 pods 使用超过了它们的 limits，或者节点的内存压力很大而且没有 QoS 更低的 pod，否则不会被杀死
* **Burstable**：这种类型的 pod 可以多于自己请求的资源（上限有 limit 指定，如果 limit 没有配置，则可以使用主机的任意可用资源），但是重要性认为比较低，可以是一般性的应用或者批处理任务
* **Best Effort**：优先级最低，集群不知道 pod 的资源请求情况，调度不考虑资源，可以运行到任意节点上（从资源角度来说），可以是一些临时性的不重要应用。pod 可以使用节点上任何可用资源，但在资源不足时也会被优先杀死

Pod 的 requests 和 limits 是如何对应到这三个 QoS 等级上的，可以用下面一张表格概括：

![img](https://i.loli.net/2021/07/19/wGp6ZMoVuFCTQ18.png)

## 参考配置文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: demo-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
#service
apiVersion: v1
kind: Service
metadata:
  name: demo-nginx-service
  namespace: demo-ns
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx

---
#ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: demo-nginx-service
      port:
        number: 80

```
