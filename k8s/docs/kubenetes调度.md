# kubenetes 调度介绍

## 简介

在开始前，先来看看Kubernetes的架构示意图，其中控制平面包含以下三大组件：kube-scheduler、kube-apiserver、kube-controller-manager。

<img src="https://i.loli.net/2021/07/22/9hKSMxzYwnm8R3l.png" width=600 />

>如上所述，kube-scheduler是K8S系统的核心组件之一，其主要负责Pod的调度，其监听kube-apiserver，查询未分配 Node的Pod（未分配、分配失败及尝试多次无法分配），根据配置的调度策略，将Pod调度到最优的工作节点上，从而高效、合理的利用集群的资源，该特性是用户选择K8S系统的关键因素之一，帮助用户提升效率、降低能耗。

kube-scheduler通过汇集节点资源、节点地域、节点镜像、Pod调度等信息综合决策，确保Pod分配到最佳节点，以下为kube-scheduler的主要目标：

* 公平性：在调度Pod时需要公平的决策，每个节点都有被分配的机会，调度器需要针对不同节点作出平衡决策。
* 资源高效：最大化提升所有可调度资源的利用率，使有限的CPU、内存等资源服务尽可能更多的Pod。
* 性能：能快速的完成对大规模Pod的调度工作，在集群规模扩增的情况下，依然能确保调度的性能。
* 灵活性：在实际生产中，用户希望Pod的调度策略是可扩展的，从而可以定制化调度算法以处理复杂的实际问题。因此平台要允许多种调度器并行工作，并支持自定义调度器。

## 调度器

### 默认 scheduler 实现步骤

#### 第一步：预选(Predicate)

假如我们这里有一堆节点，在这堆节点中先排除那些完全不符合此pod运行法则的节点，在k8s中pod中运行容器可以定义三个维度的资源限制

* 第一叫资源需求,意思是说每一个节点只有满足我的最低资源需求他才能够运行.
* 第二个维度我们称为资源的上限，也叫资源的限额。超出这个限额我们也就不再给他分配任何内存等资源。
* 第三个维度我们称为当前占用：而pod当前用了多少，我们虽然说他需求两个g内存，他也不一定一启动一运行就一定有两个g内存，所以这就叫当前占用。

#### 第二步：优选(Priority)

基于一系列的算法函数把每一个节点的属性输入进去，然后去计算每一个节点的优先级，计算完以后再对他们进行排序。可以认为叫逆序排序，然后取得分最高那一个，而后得分最高的那一个就是我们选择的节点。即基于优先级找出最佳匹配的节点。

#### 第三步：选定(Select)

优选完成后我们将pod绑定至选定好的节点即可。如果说出现一种极端情况，最后计算得分，最高得分者有好几个，这个时候就没有任何偏向性了。就随机选择一个即可。

然而有些有特殊偏好的pod资源需要运行在特定的节点上，因为这个pod对节点有一些特殊偏好，比如说有些应用必须要运行在ssd硬盘的节点上。我们不一定所有节点都有ssd，有些pod需要运行在GPU服务器节点上，说不定要挖矿，因此在这种情况下我们需要对节点添加标签分类的，这种分类有地理位置上的分类，有特性上的分类，拥有的硬件设备之类的，我们要给他加上很多的标签，而后我们的pod再去定义他要运行在哪个或哪些节点时是可以完全额外定义自己的特有倾向性的。在我们pod中有一个属性叫做nodeselector，意思是我只对某一些特殊的类型的节点有倾向。哪些特殊类型呢？

我们使用nodeselector去匹配节点标签，如果节点拥有这个标签或节点这个标签的值符合我们定义的选择标准那么他就会被分配到这个节点。因此可以理解为这就是一种调度条件的限定。这个时候在预选过程中哪些节点符合条件？因此我们所定义的很多Pod的属性都有可能会影响到预选甚至是优选中的步骤。

### 常用的特殊的调度方式

#### 节点亲和性调度

>通过nodeselector来完成调度

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-node
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app-node
  template:
    metadata:
      labels:
        app: nginx-app-node
    spec:
      containers:
      - name: nginx-app-node
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   #必须要满足的条件才能调度
            nodeSelectorTerms:  # 这级中多个数组表示或的关系
            - matchExpressions: #这级中多个数组条件，表示并且的关系
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
          preferredDuringSchedulingIgnoredDuringExecution:  #倾向于，不是必须条件，如果满足更好
          - weight: 1   #权重，权重越高优先选择
            preference:
              matchExpressions:
              - key: disktype
                operator: NotIn
                values:
                - ssd

```

#### pod的亲和性和pod的反亲和性调度

有时候我们期望某些个Pod运行在同一或相邻的节点上。这个相邻有可能是网络通信带宽可用量更大一些这样的节点。

如果我们期望两个pod运行在同一个节点上这个我们就称之为他们具有亲和性。即某些个pod倾向运行在同一位置就表示他们具有亲和性。而倾向于不要运行在同一位置他们就有反亲和性。

亲和性调度：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-pod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app-pod
  template:
    metadata:
      labels:
        app: nginx-app-pod
    spec:
      containers:
      - name: nginx-app-pod
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   #必须要满足的条件才能调度
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx-ds
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:  #倾向于，不是必须条件，如果满足更好
          - weight: 100   #权重，权重越高优先选择
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - nginx-app-node
              topologyKey: kubernetes.io/hostname
```

反亲和性调度：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-pod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app-pod
  template:
    metadata:
      labels:
        app: nginx-app-pod
    spec:
      containers:
      - name: nginx-app-pod
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAntiAffinity:    #反亲和性调度
          requiredDuringSchedulingIgnoredDuringExecution:   #必须要满足的条件才能调度
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx-app-pod #这里改成自己的label 可以把多个pod分散在不同节点
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:  #倾向于，不是必须条件，如果满足更好
          - weight: 100   #权重，权重越高优先选择
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - nginx-app-node
              topologyKey: kubernetes.io/hostname
```

#### 污点和污点容忍调度

>Taints【定义在节点之上】，Tolerations【定义在pod之上】

它是一种反其道而行之的做法，所谓反其道而行之可以认为说刚刚我们一直讲pod怎么选定哪些节点，但是我们也允许节点可以不让pod来运行，即我们可以给一些所谓的节点打上污点标识说明这个节点是拥有不光彩，见不得人的一些事情存在的，因此一个pod是否能够运行在这些节点上就取决于他是否能容忍这些污点。

给node打污点

```bash
#添加污点
kubectl taint nodes w02 gpu=true:NoSchedule
#删除污点
kubectl taint nodes w02 gpu=true:NoSchedule-
#另外两种
#PerferNoSchedule:最好不要调度到这个节点
#NoExecute:不能拉起，马上被驱逐
```

配置pod 容忍

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-taint
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app-taint
  template:
    metadata:
      labels:
        app: nginx-app-taint
    spec:
      containers:
      - name: nginx-app-taint
        image: nginx
        ports:
        - containerPort: 80
      tolerations:
      - key: "gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```
