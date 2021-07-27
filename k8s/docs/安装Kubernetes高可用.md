# 安装Kubernetes高可用

## 一、创建API Server 的Load Balancer(私网)

* **监听端口:** 6443/TCP
* **后端资源组:** 包含m01,m02,m03
* **后端端口:** 6443
* 开启【按源地址保持会话】

> 根据每个环境的实际情况不同，实现LoadBalancer的方式不一样，这里只简单的用keepalived+Haproxy做示例，其他的选项你可以选择 nginx,haproxy和云供应商提供的负载均衡产品。

### 1.1 部署keepalived+Haproxy - apiserver 高可用

#### 1.1.1 高可用介绍

根据k8s 官方文档将HA拓扑分为两种，Stacked etcd topology（堆叠ETCD）和External etcd topology（外部ETCD）。
**堆叠ETCD：** 每个master节点上运行一个apiserver和etcd只与本节点的apiserver 通信。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/kubeadm-ha-topology-stacked-etcd.svg" width=1200 />

**外部ETCD：** etcd集群运行在单独的主机上，每个etcd都与apiserver节点通信。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/kubeadm-ha-topology-external-etcd.svg" width=1200 />

> 官方文档主要是解决了高可用场景下apiserver与etcd集群的关系，三个master节点防止单点故障，但是集群对外访问接口不可能将三个apiserver都暴露出去，一个挂掉时还是不能自动切换到其他节点。

#### 1.1.2 高可用部署架构

下图时我们在生产环境所用的部署架构：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/k8s-ha.png" width=600 />

* 由外部负载均衡器提供一个vip，流量负载到keepalived master节点上。
* 当keepalived节点出现故障，vip自动飘到其他可用节点。
* haproxy负责将流量负载到apiserver节点。
* 三个apiserver会同时工作。<font color=red>注意：k8s中controller-manager和scheduler只会有一个节点工作</font>

#### 1.1.3 部署haproxy

> haproxy 可以安装在主机上，也可以使用docker容器实现。我们使用docker容器方式。

创建/etc/haproxy/haproxy.cfg 配置文件，如下：

```config

#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/2.1/doc/configuration.txt
#   https://cbonte.github.io/haproxy-dconv/2.1/configuration.html
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

#    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
#    user        haproxy
#    group       haproxy
    # daemon

    # turn on stats unix socket
    #stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  kubernetes-apiserver
    mode tcp
    bind *:9443  ## 监听9443端口
    # bind *:443 ssl # To be completed ....

    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    default_backend             kubernetes-apiserver

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp  # 模式tcp
    balance     roundrobin  # 采用轮询的负载算法
# k8s-apiservers backend  # 配置apiserver，端口6443
 server m01 192.168.100.191:6443 check
 server m02 192.168.100.192:6443 check
 server m03 192.168.100.193:6443 check

```

分别在三个节点启动haproxy

```shell
docker run -d --name=diamond-haproxy --net=host --restart=always -v /etc/haproxy:/usr/local/etc/haproxy:ro haproxy 
```

#### 1.1.4 部署keepalived

> keepalived是以VRRP(虚拟路由冗余协议)协议为基础，包括一个master和多个backup。master劫持vip对外提供服务。master发送组播，backup节点收不到vrrp包时认为master宕机，此时选出剩余优先级最高的节点作为新的master，劫持vip。

在本机创建keepalived.conf配置文件

```config
global_defs {
   script_user root 
   enable_script_security
}

vrrp_script chk_haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 9443) ]]; then exit 0; else exit 1; fi'"  # haproxy 检测
    interval 2  # 每2秒执行一次检测
    weight 11 # 权重变化
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER # backup节点设为BACKUP
  virtual_router_id 51 # id设为相同，表示是同一个虚拟路由组
  priority 100 #初始权重
  nopreempt #可抢占

  unicast_peer {

  }

  virtual_ipaddress {
    192.168.100.190  # vip
  }

  authentication {
    auth_type PASS
    auth_pass password
  }

  track_script {
      chk_haproxy
  }

  notify "/container/service/keepalived/assets/notify.sh"
}
```

分别在三台节点启动keepalived

```shell
docker run --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host --restart=always --volume /etc/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf -d osixia/keepalived:2.0.20 --copy-service  
```

## 二、初始化第一个master节点

```shell
#只在第一个master节点执行
#替换MASTER_VIP为你的master虚拟群集IP
export MASTER_VIP=192.168.100.190
# Kubernetes 容器组所在的网段，该网段安装完成后，由kubernetes创建和管理，事先并不存在与你的物理网络中
export POD_SUBNET=172.18.0.1/16
```

```shell
#!/bin/bash
#只在mster节点执行
#脚本出错时终止执行
set -e

if [ ${#POD_SUBNET} -eq 0 ] || [ ${#MASTER_VIP} -eq 0 ]; then
  echo -e "\033[31;1m请确保您已经设置了环境变量 POD_SUBNET 和 MASTER_VIP \033[0m"
  echo 当前POD_SUBNET=$POD_SUBNET
  echo 当前MASTER_VIP=$MASTER_VIP
  exit 1
fi

# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
cgroupDriver: systemd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "${MASTER_VIP}:6443"
networking:
  serviceSubnet: "172.19.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs
# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config


# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
rm -f calico-3.18.yaml
wget https://baoxishi.blob.core.chinacloudapi.cn/sharefile/calico-3.18.yaml
sed -i "s#192\.168\.0\.0/16#${POD_SUBNET}#" calico-3.18.yaml
# 添加如下内容：
# 如果不添加可能会导致node节点之间无法连接
- name: IP_AUTODETECTION_METHOD 
  value: "interface=eth.*" # eth.*正则表示匹配到eth开头的所有网卡,或者直接指定 insterface=eth0
#配置如下
#- name: CLUSTER_TYPE 

#  value: "k8s,bgp" 

#- name: IP_AUTODETECTION_METHOD 

#  value: "interface=eth.*" #或者 value: "interface=eth0"

#- name: IP 

#  value: "autodetect" 

#执行
kubectl apply -f calico-3.18.yaml
```

**执行结果** :
在执行结果中着重注意以下信息：保存到文本中备用

```bash
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.100.190:6443 --token plmj87.uhqlmiojvd26ifdr \
    --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 \
    --control-plane --certificate-key 79eb7999f0d23b0e570cc0e972ac04c8e3b208a3097d138b05d771283c877830

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.190:6443 --token plmj87.uhqlmiojvd26ifdr \
    --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 
```

**检查master初始化结果**：

```shell
#旨在第一个master节点上运行

#执行如下命令，等待3-10分钟，直到多有的容器组处于Running状态
watch kubectl get pod -n kube-system -o wide
#查看 master 节点初始化结果
kubectl get nodes
```

>注意：
>请等到所有容器组(大约9个)全部处于Running状态，才进行下一步

## 三、初始化第二、三个节点

>**注意：**
>
> * 初始化 master 节点的 token 有效时间为 2 小时
> * 如果之前没有添加LB，这里要添加上，join命令需指定Load Balancer IP 或者dns名称。

如果之前安装第一个master节点输出结果中给出的token没有超过2个小时，则直接使用那个token添加第二、三个节点。

如果超过了2个小时，或者之前只用了一个master节点，后面想扩容，则需要获取master token

### 3.1 获取 Master 节点的join命令

#### 3.1.1 获取certificate key

在第一台master节点上执行如下语句

```shell
# 只在第一个master节点上执行
kubeadm init phase upload-certs --upload-certs
```

输出结果如下：

```bash
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
f2bb24325fe4f0f36d6d20eb6d6c6c155d28e272255d9475883c602a964d9918
```

#### 3.1.2 获取join命令

在第一台master节点上执行如下语句

```shell
#只在第一个master节点上执行
kubeadm token create --print-join-command
```

输出结果如下：

```bash
kubeadm join 192.168.100.190:6443 --token zqifcs.f0i9c5gxdo84kam1     --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 
```

根据以上两个输出结果，第二三个节点的join命令如下：

```shell
kubeadm join 192.168.100.190:6443 --token zqifcs.f0i9c5gxdo84kam1 --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 --control-plane --certificate-key f2bb24325fe4f0f36d6d20eb6d6c6c155d28e272255d9475883c602a964d9918
```

### 3.2 初始化第二、三个master节点

在另外两个master节点上执行

```shell
kubeadm join 192.168.100.190:6443 --token zqifcs.f0i9c5gxdo84kam1 --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 --control-plane --certificate-key f2bb24325fe4f0f36d6d20eb6d6c6c155d28e272255d9475883c602a964d9918
```

检查master初始化结果

```shell
#在第一台master节点上执行
#查看 master 节点初始化结果
kubectl get nodes
```

检查haproxy日志

```shell
#查看haproxy docker的日志
docker logs 8662e2d1f85c

#输出结果如下：
[WARNING] 076/073337 (7) : Server kubernetes-apiserver/m01 is UP, reason: Layer4 check passed, check duration: 0ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
[WARNING] 077/020948 (7) : Server kubernetes-apiserver/m02 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

## 四、初始化 work 节点

### 4.1 获得join命令参数

>**有效时间**
> 该token的有效时间为2个小时，2个小时内，你可以使用此token初始化任意数量的work节点。超过2个小时的则需要重新获取。

```shell
# 在第一个master节点上执行
kubeadm token create --print-join-command
```

输出的join命令和参数如下所示：

```shell
kubeadm join 192.168.100.190:6443 --token c9h7aj.0oyzt5eupkkok526 --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 
```

### 4.2 初始化worker节点

针对所有的worker节点执行如下语句：

```shell
# 把前面输出的join命令贴进来，注意查看LB IP是否正确
kubeadm join 192.168.100.190:6443 --token c9h7aj.0oyzt5eupkkok526 --discovery-token-ca-cert-hash sha256:5e84b524d22dfff89dbba23a5e87aaaae04914cdc017e3d56197b3552ae6f9c0 
```

### 4.3 检查 worker 初始化结果

```shell
# 在第一个master节点上执行
kubectl get nodes
```

### 4.4 移除 worker 节点

> 如果你的worker节点有问题或者计划移除worker节点，可以执行如下命令：

```shell
# 在准备移除的worker节点上执行
kubeadm reset
# 在第一个master节点上执行
kubectl delete node demo-worker-*-*
```
