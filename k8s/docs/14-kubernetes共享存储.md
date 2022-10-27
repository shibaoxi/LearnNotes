# Kubernetes 共享存储

## 共享存储机制

k8s对有状态的容器应用或者需要对数据进行持久化的应用，可以将容器内的目录挂载到宿主机的容器目录或者emptyDir临时存储卷。另外，k8s还开放了两个资源，分别是PersistentVolume(PV)和PersistentVolumeClaim(PVC)，这两个资源对象可允许k8s使用外部的存储设备。

>比如在生产环境中有一个专门的文件服务器，那么就可以使用PV对文件服务器的资源进行定义，比如总共有多少容量等，然后用PVC对PV资源进行申请，申请多少容量，然后再容器里引用PVC即可。

![img](https://i.loli.net/2021/07/27/fQlsVIcbrPSyAdu.png)

## 概念

- **PV** ：是对底层网络共享存储的抽象，将共享存储定义为一种“资源”，比如Node也是容器应用可以消费的资源。PV由管理员创建和配置，与共享存储的具体实现直接相关。

- **PVC**：则是用户对存储资源的一个“申请”，就像Pod消费Node资源一样，PVC能够消费PV资源。PVC可以申请特定的存储空间和访问模式。

- **StorageClass**：用于标记存储资源的特性和性能，管理员可以将存储资源定义为某种类别，正如存储设备对于自身的配置描述（Profile）。根据StorageClass的描述可以直观的得知各种存储资源的特性，就可以根据应用对存储资源的需求去申请存储资源了。存储卷可以按需创建。

## PV介绍

> PV作为存储资源，主要包括存储能力、访问模式、存储类型、回收策略、后端存储类型等关键信息的设置。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    name: mynfs # name can be anything
spec:
  storageClassName: manual # same storage class as pvc
  capacity: #容量
    storage: 15Gi
  accessModes:  #访问模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle #回收策略
  nfs:
    server: 192.168.100.196 # ip addres of nfs server
    path: "/home/data" # path to directory

```

### kubernetes支持的PV类型

|类型|描述|
|---|---|
|AWSElasticBlockStore|AWS公有云提供的ElasticBlockStore|
|AzureFile|Azure公有云提供的Azure File|
|AzureDisk|Azure公有云提供的Disk|
|CephFS|一种开源共享存储系统|
|FC(Fibre Channel)|光纤存储设备|
|FlexVolume|一种插件式的存储机制|
|Flocker|一种开源共享存储系统|
|GCEPersistentDisk|GCE公有云提供的PersistentDisk|
|Glusterfs|一种开源共享存储系统|
|HostPath|宿主机目录，仅用于单机测试|
|iSCSI|iSCSI存储设备|
|Local|本地存储设备，目前可以通过指定块（Block）设备提供Local PV，或通过社区开发的sig-storage-local-static-provisioner插件（ <https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner> ）来管理Local PV的生命周期|
|NFS|网络文件系统|
|Portworx Volumes|Portworx提供的存储服务|
|Quobyte Volumes|Quobyte提供的存储服务|
|RBD（Ceph Block Device）|Ceph块存储|
|ScaleIO Volumes|DellEMC的存储设备|
|StorageOS|StorageOS提供的存储服务|
|VsphereVolume|VMWare提供的存储系统|

### 配置文件关键配置参数

1. **存储能力(Capacity)**：描述存储设备具备的能力，支持对存储空间的设置（storage=xx）
2. **存储卷模式(Volume Mode)**：volumeMode=xx，可选项包括Filesystem（文件系统）和Block（块设备），默认值是FileSystem。

    > 目前有以下PV类型支持块设备类型：
    > AWSElasticBlockStore、AzureDisk、FC、GCEPersistentDisk、iSCSI、Local volume、RBD（Ceph Block Device）、VsphereVolume（alpha）

3. **访问模式(Acess Modes)**：用于描述应用对存储资源的访问权限
    - ReadWriteOnce（RWO）：读写权限，并且只能被单个Node挂载。
    - ReadOnlyMany（ROX）：只读权限，允许被多个Node挂载。
    - ReadWriteMany（RWX）：读写权限，允许被多个Node挂载。

    ![img](https://i.loli.net/2021/07/27/JeBG83vHrgM2oDA.png)

4. **存储类别(Class)**：设定存储的类别，通过storageClassName参数指定给一个StorageClass资源对象的名称，具有特定类别的PV只能与请求了该类别的PVC进行绑定。未绑定类别的PV则只能与不请求任何类别的PVC进行绑定。

5. **回收策略（Reclaim Policy）**：通过persistentVolumeReclaimPolicy字段设置
    - Retain 保留：保留数据，需要手工处理。
    - Recycle 回收空间：简单清除文件的操作（例如执行rm -rf /thevolume/* 命令）。
    - Delete 删除：与PV相连的后端存储完成Volume的删除操作。
    > EBS、GCE PD、Azure Disk、OpenStack Cinder等设备的内部Volume清理）。
6. **挂载参数（Mount Options）**： 在将PV挂载到一个Node上时，根据后端存储的特点，可能需要设置额外的挂载参数，可根据PV定义中的mountOptions字段进行设置。
7. **节点亲和性（Node Affinity）**：限制只能通过某些Node来访问Volume，可在nodeAffinity字段中设置。使用这些Volume的Pod将被调度到满足条件的Node上。

    > 此参数仅用于Local存储卷上

### PV生命周期的各个阶段

- Available：可用状态，还未与某个PVC绑定。
- Bound：已与某个PVC绑定。
- Released：绑定的PVC已经删除，资源已释放，但没有被集群回收。
-Failed：自动资源回收失败。

## PVC介绍

PVC作为用户对存储资源的需求申请,主要包括存储空间请求、访问模式、PV选择条件和存储类别等信息的设置。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:  #访问模式
  - ReadWriteOnce
  resources: #申请资源，8Gi存储空间
    requests:
      storage: 8Gi
  storageClassName: manual #存储类别
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
    - {key: environment, operator: In, values: [dev]}
```

### 配置文件关键配置

1. **资源请求(Resources)**：描述对存储资源的请求，目前仅支持request.storage的设置，即是存储空间的大小。
2. **访问模式(AccessModes)**：用于描述对存储资源的访问权限，与PV设置相同
3. **存储卷模式（Volume Modes）**：用于描述希望使用的PV存储卷模式，包括文件系统和块设备。
4. **PV选择条件（Selector）**：通过对Label Selector的设置，可使PVC对于系统中已存在的各种PV进行筛选。
    >选择条件可以使用matchLabels和matchExpressions进行设置，如果两个字段都设置了，则Selector的逻辑将是两组条件同时满足才能完成匹配
5. **存储类别（Class）**：PVC 在定义时可以设定需要的后端存储的类别（通过storageClassName字段指定），以减少对后端存储特性的详细信息的依赖。只有设置了该Class的PV才能被系统选出，并与该PVC进行绑定。
    >PVC也可以不设置Class需求。如果storageClassName字段的值被设置为空（storageClassName=""），则表示该PVC不要求特定的Class，系统将只选择未设定Class的PV与之匹配和绑定。PVC也可以完全不设置storageClassName字段，此时将根据系统是否启用了名为DefaultStorageClass的admission controller进行相应的操作

>PVC和PV都受限于Namespace，PVC在选择PV时受到Namespace的限制，只有相同Namespace中的PV才可能与PVC绑定。Pod在引用PVC时同样受Namespace的限制，只有相同Namespace中的PVC才能挂载到Pod内。

## PV和PVC的生命周期

![img](https://i.loli.net/2021/07/27/ki41PsTGNJOMwQX.png)

1. **资源供应**

    k8s支持两种资源的供应模式：静态模式（Static）和动态模式（Dynamic）。资源供应的结果就是创建好的PV。

    - 静态模式：集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置。
    - 动态模式：集群管理员无需手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式。
2. **资源绑定**
    在定义好PVC之后，系统将根据PVC对存储资源的要求（存储空间和访问模式）在已存在的PV中选择一个满足PVC要求的PV，一旦找到，就将该PV与定义的PVC进行绑定，应用就可以使用这个PVC了。如果系统中没有这个PV，则PVC则会一直处理Pending状态，直到系统中有符合条件的PV。PV一旦绑定到PVC上，就会被PVC独占，不能再与其他PVC进行绑定。当PVC申请的存储空间比PV的少时，整个PV的空间就都能够为PVC所用，可能会造成资源的浪费。如果资源供应使用的是动态模式，则系统在为PVC找到合适的StorageClass后，将自动创建一个PV并完成与PVC的绑定。
3. **资源使用**
    Pod使用Volume定义，将PVC挂载到容器内的某个路径进行使用。Volume的类型为Persistent VolumeClaim，在容器挂载了一个PVC后，就能被持续独占使用。多个Pod可以挂载到同一个PVC上。
4. **资源释放**
    当存储资源使用完毕后，可以删除PVC，与该PVC绑定的PV会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被保留在存储设备上，只有在清除之后该PV才能被再次使用。
5. **资源回收**
    对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用。

## StorageClass

StorageClass作为对存储资源的抽象定义，对用户设置的PVC申请屏蔽后端存储的细节，一方面减少了用户对存储资源细节的关注，另一方面减少了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。

StorageClass的定义主要包括名称、后端存储的提供者（privisioner）和后端存储的相关参数配置。StorageClass一旦被创建，就无法修改，如需修改，只能删除重建。

下例定义了一个storageclass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

### 关键配置

- 提供者(provisioner): 描述存储资源的提供者，也可以看作后端存储驱动
- 参数（Parameters）: 后端存储i元提供者的参数设置，不同的Provisioner包括不同的参数设置。某些参数可以不显示设定，Provisioner将使用其默认值。

### 设置默认的StorageClass

首先需要启用名为DefaultStorageClass的admission controller，即在kube-apiserver的命令行参数--admission-control中增加

```bash
--admission-control=...,DefaultStorageClass
```

然后，在StorageClass的定义中设置一个annotation

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
  annotations:
    storageclass.beta.kubernetes.io/is-default-class="true"
privisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

## 利用NFS作为共享存储的示例

### 创建NFS

参考链接：<https://qizhanming.com/blog/2018/08/08/how-to-install-nfs-on-centos-7>

### kubernetes 使用nfs存储卷

参考链接：<https://juejin.cn/post/6850418111150686221>

### 利用NFS动态提供Kubernetes后端存储卷

参考链接：<https://www.cnblogs.com/panwenbin-logs/p/12196286.html>

<https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client>

### Kubernetes v1.20.0 报"unexpected error getting claim reference: selfLink was empty, can't make reference"

解决方法：<https://www.orchome.com/10024>
