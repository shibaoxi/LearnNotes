# POD 操作介绍

## 命令

### 列出Pod列表

    kubectl get pods

### 查看Pod更多信息，使用如下命令

    kubectl describe pod [pod name]

### 列出pod时显示 pod IP 和 pod 的节点

    kubectl get pods -o wide

### 使用yaml创建pod

    kubectl create -f demo-manual.yaml

### 查看pod完整描述文件yaml格式

    kubectl get po demo-manual -o yaml

### 查看pod完整描述文件json格式

    kubectl get po demo-manual -o json

### 获取pod日志

    kubectl logs demo-manual

### 如果pod中包含多个容器，获取日志时可以指定单个容器

    kubectl logs demo-manual -c demo01

### 端口转发，将本地网络端口转发到pod中的端口

    kubectl port-forward demo-manual 8888:8080

### 按名称删除pod

    kubectl delete po demo01

### 通过删除整个命名空间来删除pod

    kubectl delete ns custom-namespace

### 删除命名空间中的所有pod，但保留命名空间

    kubectl delete po --all

### 标签

---

#### 查看pod标签

    kubectl get po --show-labels

#### 如果查看某些特定标签，可以使用-L选项

    kubectl get po -L creation_method,env

#### 修改现有pod标签

     kubectl label po demo-manual creation_method=manual  // 增加标签
     kubectl label po demo-manual-v2 env=debug --overwrite   // 修改现有标签

#### 使用标签选择器列出pod

```bash
# 列出creation_method=manual的pod
kubectl get po -l creation_method=manual
# 列出包含env标签的所有pod，无论其值如何
kubectl get po -l env
# 列出没有env标签的pod
kubectl get po -l '!env'
# 列出带有creation_method标签，并且值不等于manual的pod
kubectl get po -l 'creation_method!=manual'
# 选择带有env标签，且值为prod或者devel的pod
kubectl get po -l 'env in (prod,devel)'
# 选择带有env标签，但其值不是prod或者devel的pod
kubectl get po -l 'env notin (prod,devel)'
```

##### 使用标签选择器删除pod

    kubectl delete po -l creation_method=manual
