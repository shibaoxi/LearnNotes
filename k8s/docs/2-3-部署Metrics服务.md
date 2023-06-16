# 部署Metrics服务

## 简介

在Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。
开源地址：<https://github.com/kubernetes-sigs/metrics-server>

## 部署

下载部署文件

```bash
# 单节点
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# 高可用
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
```

修改yaml文件,添加 --kubelet-insecure-tls 启动参数，并替换镜像地址为国内镜像源

```bash
sed -i -e 's/k8s.gcr.io\/metrics-server/registry.aliyuncs.com\/google_containers/g' high-availability.yaml
sed -i -e '/args:/a\        - --kubelet-insecure-tls' high-availability.yaml
```

```yml
...
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.1
        imagePullPolicy: IfNotPresent
...

```

![20221121133355](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221121133355.png)

## 检查

```bash
kubectl get pod -A -o wide | grep metrics*
kube-system   metrics-server-5db6fcfddc-7l5gw            1/1     Running   0                112m    172.19.190.146    k8smaster01   <none>           <none>
kube-system   metrics-server-5db6fcfddc-npc9d            1/1     Running   0                112m    172.19.153.67     k8smaster03   <none>           <none>
```

查看性能

```bash
kubectl top node
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8smaster01   234m         11%    1536Mi          42%       
k8smaster02   232m         11%    1562Mi          43%       
k8smaster03   200m         10%    1784Mi          49% 

kubectl top pod -A
NAMESPACE     NAME                                       CPU(cores)   MEMORY(bytes)   
kube-system   calico-kube-controllers-84c476996d-87gmg   3m           27Mi            
kube-system   calico-node-969pc                          31m          136Mi           
kube-system   calico-node-kppl4                          33m          172Mi           
kube-system   calico-node-rqpvj                          39m          169Mi           
kube-system   coredns-74586cf9b6-kqw4h                   2m           21Mi            
kube-system   coredns-74586cf9b6-nntmb                   2m           23Mi            
kube-system   etcd-k8smaster01                           37m          110Mi           
kube-system   etcd-k8smaster02                           45m          114Mi           
kube-system   etcd-k8smaster03                           37m          89Mi            
kube-system   kube-apiserver-k8smaster01                 54m          345Mi 
```
