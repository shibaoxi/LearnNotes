# 自动横向伸缩pod与群集节点

## pod的横向自动伸缩

> 横向pod自动伸缩是指由控制器管理的pod副本数量的自动伸缩。它由Horizontal控制器执行，我们通过创建一个Horizontal pod autoscaler （HPA）资源来启用和配置Horizontal控制器。该控制器周期性检查pod度量，计算满足HPA资源所配置的目标数值所需的副本数量，进而调整目标资源（如：Deployment、ReplicaSet、StatefulSet等）的replicas字段。

### 自动伸缩过程

自动伸缩的过程可以分为三个步骤：

* 获取被伸缩资源对象所管理的所有pod度量
* 计算使度量数值到达（或接近）所指定目标数值所需的pod数量。
* 更新被伸缩资源的replicas字段。

**1. 获取pod度量:**

pod度量数据 由cAdvisor的agent采集的，这些数据将由集群级的组件Heapster聚合，HPA控制器向Heapster发起REST调用来获取所有pod度量数据。

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210120114712.png)

**2. 计算所需pod数量：**

一旦Autoscaler获得了它所调整的资源所辖pod的全部度量，它便可以利用这些度量计算出所需副本数量。

* 单个度量

    将所有的pod的度量求和后除以HPA资源上配置的目标值，再向上取整即可。

* 多个度量

    Autoscaler单独计算每个度量的副本数，然后取最大值（比如：如果需要4个pod达到目标CPU使用率，以及需要3个pod来达到目标QPS，那么Autocaler将扩展到4个pod）

    如图所示：

    ![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210125110615.png)

**3. 更新被伸缩资源的副本数：**

自动伸缩操作的最后一步是更新被伸缩资源对象（比如ReplicaSet）上的副本数字段，然后让ReplicaSet控制器负责启动更多pod或者删除多余的pod。

Autoscaler控制器通过Scale子资源来修改被伸缩资源的replicas字段。

![img](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210128152239.png)
