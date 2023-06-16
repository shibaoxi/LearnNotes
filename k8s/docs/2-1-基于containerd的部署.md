# 使用containerd 作为容器运行时部署Kubernetes

## 一、基础环境准备

### 1. 服务器说明

|系统类型    |IP地址         | 节点角色 | CPU | Memory | Hostname       |
| --------- | ------------- | -----   | --- | ------ | --------       |
|RedHat-8.7 |192.168.100.111|  Master | 4   | 8GB    | k8smaster01    |
|RedHat-8.7 |192.168.100.112|  Master | 4   | 8GB    | k8smaster02    |
|RedHat-8.7 |192.168.100.113|  Master | 4   | 8GB    | k8smaster03    |
|RedHat-8.7 |192.168.100.114|  Worker | 4   | 8GB    | k8snode01      |
|RedHat-8.7 |192.168.100.115|  Worker | 4   | 8GB    | k8snode01      |

### 2. 系统设置(所有节点master+worker)

#### 2.1 主机名

主机名必须每个节点都不一样，并且保证所有节点之间可以通过hostname相互访问。

```bash
# 修改主机名
hostname
# 修改主机名
hostnamectl set-hostname <your host name>
# 配置host,使所有节点之间可以通过hostname相互访问
cat >> /etc/hosts <<EOF
192.168.100.111 k8smaster01
192.168.100.112 k8smaster02
192.168.100.113 k8smaster03
192.168.100.114 k8snode01
192.168.100.115 k8snode02
EOF

```

#### 2.2 安装依赖包

```bash
# 更新yum
yum update -y
# 安装工具包
yum install -y tree tc net-tools wget bash-completion telnet
# 安装依赖包
yum -y install ipvsadm ipset sysstat conntrack libseccomp

```

#### 2.3 关闭防火墙、swap，重置iptables

```bash
# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
# 重置iptables
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
# 关闭swap
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭selinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

#### 2.4 配置时间同步

```bash
# 安装 chrony Redhat 已经默认安装
yum -y install chrony
# 配置时间源为国内服务器
# 服务器端配置
sed -i -e '/^pool.*/d' -e '/^server.*/d' -e '/^# Please consider .*/a\server ntp.aliyun.com iburst\nserver time1.cloud.tencent.com iburst\nserver ntp.tuna.tsinghua.edu.cn iburst' -e 's@^#allow.*@allow 0.0.0.0/0@' -e 's@^#local.*@local stratum 10@' /etc/chrony.conf
# 客户端配置，如果企业有自己的NTP服务器，可以通过下面语句执行
# sed -i -e '/^pool.*/d' -e '/^server.*/d' -e '/^# Please consider .*/a\server '${SERVER1}' iburst\nserver '${SERVER2}' iburst' /etc/chrony.conf

# 启用服务
systemctl enable --now chronyd
systemctl restart chronyd
systemctl is-active chronyd
```

#### 2.5 设置时区

```bash
# 配置中国时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
```

#### 2.6 优化资源限制参数

```bash
ulimit -SHn 65535
​
cat >>/etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 131072
* soft nproc 65535
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF

```

#### 2.7 配置k8s 内核

master和node节点配置ipvs模块，在内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack， 4.18以下使用nf_conntrack_ipv4即可：

系统内核优化

```bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack #内核小于4.18，把这行改成nf_conntrack_ipv4
​
cat >> /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
​#内核小于4.18，把这行改成nf_conntrack_ipv4(执行时去掉注释)
# 重启服务
systemctl restart systemd-modules-load.service

```

k8s 内核优化

```bash
cat > /etc/sysctl.d/k8s.conf <<EOF 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
​
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl params without reboot
sudo sysctl --system
```

所有节点配置完内核后，重启服务器，保证重启后内核依旧加载

```bash
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

Kubernetes内核优化常用参数详解：

```bash
net.ipv4.ip_forward = 1 #其值为0,说明禁止进行IP转发；如果是1,则说明IP转发功能已经打开。
net.bridge.bridge-nf-call-iptables = 1 #二层的网桥在转发包时也会被iptables的FORWARD规则所过滤，这样有时会出现L3层的iptables rules去过滤L2的帧的问题
net.bridge.bridge-nf-call-ip6tables = 1 #是否在ip6tables链中过滤IPv6包 
fs.may_detach_mounts = 1 #当系统有容器运行时，需要设置为1
​
vm.overcommit_memory=1  
#0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
#1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
#2， 表示内核允许分配超过所有物理内存和交换空间总和的内存
​
vm.panic_on_oom=0 
#OOM就是out of memory的缩写，遇到内存耗尽、无法分配的状况。kernel面对OOM的时候，咱们也不能慌乱，要根据OOM参数来进行相应的处理。
#值为0：内存不足时，启动 OOM killer。
#值为1：内存不足时，有可能会触发 kernel panic（系统重启），也有可能启动 OOM killer。
#值为2：内存不足时，表示强制触发 kernel panic，内核崩溃GG（系统重启）。
​
fs.inotify.max_user_watches=89100 #表示同一用户同时可以添加的watch数目（watch一般是针对目录，决定了同时同一用户可以监控的目录数量）
​
fs.file-max=52706963 #所有进程最大的文件数
fs.nr_open=52706963 #单个进程可分配的最大文件数
net.netfilter.nf_conntrack_max=2310720 #连接跟踪表的大小，建议根据内存计算该值CONNTRACK_MAX = RAMSIZE (in bytes) / 16384 / (x / 32)，并满足nf_conntrack_max=4*nf_conntrack_buckets，默认262144
​
net.ipv4.tcp_keepalive_time = 600  #KeepAlive的空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2小时）
net.ipv4.tcp_keepalive_probes = 3 #在tcp_keepalive_time之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）
net.ipv4.tcp_keepalive_intvl =15 #KeepAlive探测包的发送间隔，默认值为75s
net.ipv4.tcp_max_tw_buckets = 36000 #Nginx 之类的中间代理一定要关注这个值，因为它对你的系统起到一个保护的作用，一旦端口全部被占用，服务就异常了。 tcp_max_tw_buckets 能帮你降低这种情况的发生概率，争取补救时间。
net.ipv4.tcp_tw_reuse = 1 #只对客户端起作用，开启后客户端在1s内回收
net.ipv4.tcp_max_orphans = 327680 #这个值表示系统所能处理不属于任何进程的socket数量，当我们需要快速建立大量连接时，就需要关注下这个值了。
​
net.ipv4.tcp_orphan_retries = 3
#出现大量fin-wait-1
#首先，fin发送之后，有可能会丢弃，那么发送多少次这样的fin包呢？fin包的重传，也会采用退避方式，在2.6.358内核中采用的是指数退避，2s，4s，最后的重试次数是由tcp_orphan_retries来限制的。
​
net.ipv4.tcp_syncookies = 1 #tcp_syncookies是一个开关，是否打开SYN Cookie功能，该功能可以防止部分SYN攻击。tcp_synack_retries和tcp_syn_retries定义SYN的重试次数。
net.ipv4.tcp_max_syn_backlog = 16384 #进入SYN包的最大请求队列.默认1024.对重负载服务器,增加该值显然有好处.
net.ipv4.ip_conntrack_max = 65536 #表明系统将对最大跟踪的TCP连接数限制默认为65536
net.ipv4.tcp_max_syn_backlog = 16384 #指定所能接受SYN同步包的最大客户端数量，即半连接上限；
net.ipv4.tcp_timestamps = 0 #在使用 iptables 做 nat 时，发现内网机器 ping 某个域名 ping 的通，而使用 curl 测试不通, 原来是 net.ipv4.tcp_timestamps 设置了为 1 ，即启用时间戳
net.core.somaxconn = 16384  #Linux中的一个kernel参数，表示socket监听（listen）的backlog上限。什么是backlog呢？backlog就是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog。而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。

```

### 3. 安装Containerd(所有节点)

> v1.24 之前的 Kubernetes 版本直接集成了 Docker Engine 的一个组件，名为 dockershim。 这种特殊的直接整合不再是 Kubernetes 的一部分 （这次删除被作为 v1.20 发行版本的一部分宣布）。
> 此句话意思就是k8s 不在依赖Docker，让底层的容器技术有了可选择的可能，目前应用比较广的就是containerd 和 docker，此篇文章我们主要写使用container容器技术部署k8s.

#### 3.1 内核参数调整

>如果是安装 Docker 会自动配置以下的内核参数，而无需手动实现,但是如果安装Contanerd，还需手动配置
>允许 iptables 检查桥接流量,若要显式加载此模块，需运行 modprobe br_netfilter
>为了让 Linux 节点的 iptables 能够正确查看桥接流量，还需要确认net.bridge.bridge-nf-call-iptables 设置为 1。

配置Containerd所需的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
# 加载模块
modprobe -- overlay
modprobe -- br_netfilter
```

#### 3.2 二进制安装

> 官方下载地址 <https://github.com/containerd/containerd>

cri-containerd-xxx：包含runC，ctr、crictl、systemd 配置文件等相关文件，不包含cni插件，k8s不需要containerd的cni插件，所以选择这个二进制包安装

```bash
# 下载
wget https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-1.6.10-linux-amd64.tar.gz
# #cri-containerd-${version}-linux-amd64.tar.gz 压缩包中已经按照官方二进制部署推荐的目录结构布局好。 里面包含了 systemd 配置文件，containerd 和ctr、crictl等部署文件。 将解压缩到系统的根目录 / 中:
tar xf cri-containerd-1.6.10-linux-amd64.tar.gz -C /

```

配置Containerd的配置文件：

```bash
mkdir -p /etc/containerd
# 生成配置文件
containerd config default | tee /etc/containerd/config.toml
#将Containerd的Cgroup改为Systemd和修改containerd配置sandbox_image 镜像源设置为阿里google_containers镜像源
sed -ri -e 's/(.*SystemdCgroup = ).*/\1true/' -e 's@(.*sandbox_image = ).*@\1"registry.aliyuncs.com/google_containers/pause:latest"@' /etc/containerd/config.toml
# 设置镜像加速（可选）
sed -i '/.*registry.mirrors.*/a\        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]\n          endpoint = ["https://registry.docker-cn.com" ,"http://hub-mirror.c.163.com" ,"https://docker.mirrors.ustc.edu.cn"]' /etc/containerd/config.toml

```

配置crictl客户端连接的运行时位置：

```bash
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

启动Containerd，并配置开机自启动：

```bash
systemctl daemon-reload && systemctl enable --now containerd
```

检查相关信息

```bash
systemctl status containerd
ctr version
crictl version
crictl info
```

#### 3.3 containerd 客户端工具 nerdctl

>推荐使用 nerdctl，使用效果与 docker 命令的语法一致，github 下载链接：<https://github.com/containerd/nerdctl/releases>

- 安装 nerdctl

    ```bash
    wget https://github.com/containerd/nerdctl/releases/download/v1.0.0/nerdctl-1.0.0-linux-amd64.tar.gz
    tar xf nerdctl-1.0.0-linux-amd64.tar.gz -C /usr/local/bin/
    #配置nerdctl
    mkdir -p /etc/nerdctl/
    cat > /etc/nerdctl/nerdctl.toml <<EOF
    namespace  = "k8s.io" #设置nerdctl工具默认namespace
    insecure_registry = true #跳过安全镜像仓库检测
    EOF
    ```

- 安装 buildkit 支持构建镜像：

  >使用精简版 nerdctl 无法直接通过 containerd 构建镜像，需要与 buildkit 组全使用以实现镜像构建。当然你也可以安装上面的完整 nerdctl；buildkit 项目是 Docker 公司开源出来的一个构建工具包，支持 OCI 标准的镜像构建。它主要包含以下部分:服务端 buildkitd，当前支持 runc 和 containerd 作为 worker，默认是 runc；客户端 buildctl，负责解析 Dockerfile，并向服务端 buildkitd 发出构建请求。

  > buildkit 是典型的C/S 架构，client 和 server 可以不在一台服务器上。而 nerdctl 在构建镜像方面也可以作为 buildkitd 的客户端。

    ```bash
    # 下载地址 https://github.com/moby/buildkit
    wget https://github.com/moby/buildkit/releases/download/v0.10.6/buildkit-v0.10.6.linux-amd64.tar.gz
    tar xf buildkit-v0.10.6.linux-amd64.tar.gz -C /usr/local/
    ```

- buildkit 配置文件

  > 示例地址：<https://github.com/moby/buildkit/tree/master/examples/systemd>

    ```bash
    #/usr/lib/systemd/system/buildkit.socket
    cat > /usr/lib/systemd/system/buildkit.socket <<EOF
    [Unit]
    Description=BuildKit
    Documentation=https://github.com/moby/buildkit

    [Socket]
    ListenStream=%t/buildkit/buildkitd.sock
    SocketMode=0660

    [Install]
    WantedBy=sockets.target
    EOF
    ```

    ```bash
    # /usr/lib/systemd/system/buildkit.service
    
    cat > /usr/lib/systemd/system/buildkit.service << EOF
    [Unit]
    [Unit]
    Description=BuildKit
    Requires=buildkit.socket
    After=buildkit.socket
    Documentation=https://github.com/moby/buildkit

    [Service]
    Type=notify
    ExecStart=/usr/local/bin/buildkitd --addr fd://

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

- 启动 buildkit
  
    ```bash
    systemctl daemon-reload && systemctl enable --now buildkit
    # 检查状态
    systemctl status buildkit
    nerdctl version
    buildctl --version
    nerdctl info
    # 启用命令补全
    echo "source <(nerdctl completion bash)" >> ~/.bashrc
    ```

### 4. 部署Haproxy 和 Keepalived（在Master节点上执行）

> 在master节点上执行

#### 4.1 部署Haproxy

```bash
mkdir -p /etc/haproxy
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
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
 server k8smaster01 192.168.100.111:6443 check inter 5s rise 2 fall 3
 server k8smaster02 192.168.100.112:6443 check inter 5s rise 2 fall 3
 server k8smaster03 192.168.100.113:6443 check inter 5s rise 2 fall 3


listen stats
    mode    http
    bind    0.0.0.0:1080
    stats   enable
    stats   hide-version
    stats uri /haproxyamdin?stats
    stats realm Haproxy\ Statistics
    stats auth admin:admin
    stats admin if TRUE

EOF
```

启动容器

```bash
nerdctl run -d --name=diamond-haproxy --net=host --restart=always -v /etc/haproxy:/usr/local/etc/haproxy:ro haproxy
```

#### 4.2 部署Keepalived

```bash
mkdir -p /etc/keepalived
cat > /etc/keepalived/keepalived.conf << EOF
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
    192.168.100.110  # 修改成自己的vip地址
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

EOF
```

启动容器

```bash
nerdctl run --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host --restart=always --volume /etc/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf -d osixia/keepalived --copy-service
```

### 5.安装kubeadm等组件

配置配置k8s镜像仓库为国内源

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<-EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```

安装kubeadm组件，如果安装kubernetes 1.25.3 版本，替换相应的版本号即可

```bash
yum -y install kubeadm-1.24.7 kubelet-1.24.7 kubectl-1.24.7
# 设置开机启动
systemctl daemon-reload && systemctl enable --now kubelet
```

### 6. 初始化高可用master（在第一个Master节点执行）

kubeadm init 命令参考说明

```bash
--kubernetes-version：#kubernetes程序组件的版本号，它必须要与安装的kubelet程序包的版本号相同
--control-plane-endpoint：#多主节点必选项,用于指定控制平面的固定访问地址，可是IP地址或DNS名称，会被用于集群管理员及集群组件的kubeconfig配置文件的API Server的访问地址,如果是单主节点的控制平面部署时不使用该选项,注意:kubeadm 不支持将没有 --control-plane-endpoint 参数的单个控制平面集群转换为高可用性集群。
--pod-network-cidr：#Pod网络的地址范围，其值为CIDR格式的网络地址，通常情况下Flannel网络插件的默认为10.244.0.0/16，Calico网络插件的默认值为192.168.0.0/16
--service-cidr：#Service的网络地址范围，其值为CIDR格式的网络地址，默认为10.96.0.0/12；通常，仅Flannel一类的网络插件需要手动指定该地址
--service-dns-domain string #指定k8s集群域名，默认为cluster.local，会自动通过相应的DNS服务实现解析
--apiserver-advertise-address：#API 服务器所公布的其正在监听的 IP 地址。如果未设置，则使用默认网络接口。apiserver通告给其他组件的IP地址，一般应该为Master节点的用于集群内部通信的IP地址，0.0.0.0表示此节点上所有可用地址,非必选项
--image-repository string #设置镜像仓库地址，默认为 k8s.gcr.io,此地址国内可能无法访问,可以指向国内的镜像地址
--token-ttl #共享令牌（token）的过期时长，默认为24小时，0表示永不过期；为防止不安全存储等原因导致的令牌泄露危及集群安全，建议为其设定过期时长。未设定该选项时，在token过期后，若期望再向集群中加入其它节点，可以使用如下命令重新创建token，并生成节点加入命令。kubeadm token create --print-join-command
--ignore-preflight-errors=Swap” #若各节点未禁用Swap设备，还需附加选项“从而让kubeadm忽略该错误
--upload-certs #将控制平面证书上传到 kubeadm-certs Secret
--cri-socket  #v1.24版之后指定连接cri的socket文件路径,注意;不同的CRI连接文件不同
#如果是cRI是containerd，则使用--cri-socket unix:///run/containerd/containerd.sock #如果是cRI是docker，则使用--cri-socket unix:///var/run/cri-dockerd.sock
#如果是CRI是CRI-o，则使用--cri-socket unix:///var/run/crio/crio.sock
#注意:CRI-o与containerd的容器管理机制不一样，所以镜像文件不能通用。

```

执行如下命令

```bash
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --control-plane-endpoint=192.168.100.110:9443 \
 --kubernetes-version v1.24.7 \
 --service-cidr=10.1.0.0/16 \
 --pod-network-cidr=172.19.0.0/16 \
 --service-dns-domain cluster.local \
 --v=5
```

成功后输入如下结果

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.100.110:6443 --token qqnnf1.vt374daxb885gvda \
        --discovery-token-ca-cert-hash sha256:fa7fbe51fd14657987cb2b7d533fc8ffdb934538c8a0417aef81f6574725dd96 \
        --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.110:6443 --token qqnnf1.vt374daxb885gvda \
        --discovery-token-ca-cert-hash sha256:fa7fbe51fd14657987cb2b7d533fc8ffdb934538c8a0417aef81f6574725dd96 
```

### 7. 生成 kubectl 命令的授权文件

kubectl是kube-apiserver的命令行客户端程序，实现了除系统部署之外的几乎全部的管理操作，是kubernetes管理员使用最多的命令之一。kubectl需经由API server认证及授权后方能执行相应的管理操作，kubeadm部署的集群为其生成了一个具有管理员权限的认证配置文件/etc/kubernetes/admin.conf，它可由kubectl通过默认的“$HOME/.kube/config”的路径进行加载。当然，用户也可在kubectl命令上使用--kubeconfig选项指定一个别的位置。

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

实现 kubectl 命令补全

```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

### 8. 部署 calico 网络插件（在第一个Master节点执行）

```bash
# 参考文档 https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
rm -f calico.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml 
#执行
kubectl apply -f calico.yaml
# 查看pod状态
kubectl get pod -n kube-system
```

### 9. Kube-proxy改为ipvs模式

因为初始化群集的时候默认是iptables模式，需要手动修改一下

在第一个master节点上执行如下命令

```bash
curl 127.0.0.1:10249/proxyMode
kubectl edit cm kube-proxy -n kube-system
# 修改如下字段，然后保存退出
mode: "ipvs"
```

更新Kube-Proxy的Pod：

```bash
kubectl patch daemonset kube-proxy -p '{"spec": {"template": {"metadata": {"annotations": {"date":"`date +'%s'`"}}}}}' -n kube-system
```

验证

```bash
curl 127.0.0.1:10249/proxyMode
ipvs

ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr
  -> 192.168.100.111:6443         Masq    1      1          0         
TCP  10.1.0.10:53 rr
  -> 172.19.190.132:53            Masq    1      0          0         
  -> 172.19.190.133:53            Masq    1      0          0         
TCP  10.1.0.10:9153 rr
  -> 172.19.190.132:9153          Masq    1      0          0         
  -> 172.19.190.133:9153          Masq    1      0          0         
UDP  10.1.0.10:53 rr
  -> 172.19.190.132:53            Masq    1      0          0         
  -> 172.19.190.133:53            Masq    1      0          0  
```
