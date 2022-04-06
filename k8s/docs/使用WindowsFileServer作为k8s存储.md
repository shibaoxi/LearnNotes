# 使用Windows File Server 作为k8s集群存储

>如何使用开源的 SMB CSI Driver for Kubernetes 访问 Windows 文件服务器，并把windows 文件服务器作为存储给pod使用。

## SMB CSI Driver 介绍

该驱动程序允许 Kubernetes 访问Linux 和 Windows 节点上的SMBsmb.csi.k8s.io服务器，csi 插件名称： . 该驱动程序需要现有且已配置的 SMB 服务器，它通过在 SMB 服务器下创建新的子目录来支持通过持久卷声明动态配置持久卷。

参考：
> <https://github.com/kubernetes-csi/csi-driver-smb>

## 部署

### 安装SMB CSI Driver

使用kubectl进行安装

```bash
git clone https://github.com/kubernetes-csi/csi-driver-smb.git
cd csi-driver-smb
git checkout master
./deploy/install-driver.sh master local
```

检查pod状态

```bash
kubectl -n kube-system get pod
```

显示结果如下
<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220402110907.png" width=640 height=360 />

![PNG](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220402110907.png)

> **注意**:如果要删除CSI驱动，请执行如下语句 ```./deploy/uninstall-driver.sh master local```

### 创建Secrets

访问windows file server 共享需要用户凭据，这时我们可以把凭据存入 k8s中的secret。

```bash
kubectl create secret generic smbsec --from-literal username=USERNAME --from-literal password="PASSWORD" --from-literal domain=DOMAIN-NAME
```

请替换成自己的用户名和密码，如果你的文件服务器加入域了，可以添加--from-literal domain=DOMAIN-NAME，如果是工作组，则去掉后面的参数。

### 使用Storage Class（可选）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: smb
provisioner: smb.csi.k8s.io
parameters:
  # On Windows, "*.default.svc.cluster.local" could not be recognized by csi-proxy
  source: "//smb-server-address/sharename"
  # if csi.storage.k8s.io/provisioner-secret is provided, will create a sub directory
  # with PV name under source
  #csi.storage.k8s.io/provisioner-secret-name: "smbsec"
  #csi.storage.k8s.io/provisioner-secret-namespace: "default"
  csi.storage.k8s.io/node-stage-secret-name: "smbsec"
  csi.storage.k8s.io/node-stage-secret-namespace: "default"
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1001
  - gid=1001
  - noperm
  - mfsymlinks
  - cache=strict
  - noserverino # required to prevent data corruption
```

创建

```bash
kubectl apply -f sc-smb.yaml
```

创建一个statefulset 挂载smb共享

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-smb
  labels:
    app: nginx
spec:
  serviceName: statefulset-smb
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: statefulset-smb
          image: mcr.microsoft.com/oss/nginx/nginx:1.19.5
          command:
            - "/bin/bash"
            - "-c"
            - set -euo pipefail; while true; do echo $(date) >> /mnt/smb/outfile1; sleep 1; done
          volumeMounts:
            - name: persistent-storage
              mountPath: /mnt/smb
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: nginx
  volumeClaimTemplates:
    - metadata:
        name: persistent-storage
        annotations:
          volume.beta.kubernetes.io/storage-class: smb
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

执行如下命令，进入容器查看smb共享挂载情况

```bash
# 创建资源
kubectl apply -f statefulset.yaml
# 执行如下命令查看容器挂载
kubectl exec -it statefulset-smb-0 sh
```

结果如图所示：
<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220402200627.png" width=800 />

### PV/PVC使用方法

创建绑定SMB 共享的PV/PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-smb
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - vers=3.0
  csi:
    driver: smb.csi.k8s.io
    readOnly: false
    volumeHandle: unique-volumeid  # make sure it's a unique id in the cluster
    volumeAttributes:
      source: "//smb-server-address/sharename"
    nodeStageSecretRef:
      name: smbcreds
      namespace: default
```

创建资源

```bash
kubectl apply -f pv-smb.yaml
```

创建PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-smb
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-smb
  storageClassName: ""
```

确保pvc被创建并在一段时间后处于绑定状态

```bash
watch kubectl describe pvc pvc-smb
```

创建deployment资源测试

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: deployment-smb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      name: deployment-smb
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: deployment-smb
          image: mcr.microsoft.com/oss/nginx/nginx:1.19.5
          command:
            - "/bin/bash"
            - "-c"
            - set -euo pipefail; while true; do echo $(date) >> /mnt/smb/outfile; sleep 1; done
          volumeMounts:
            - name: smb
              mountPath: "/mnt/smb"
              readOnly: false
      volumes:
        - name: smb
          persistentVolumeClaim:
            claimName: pvc-smb
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
```
