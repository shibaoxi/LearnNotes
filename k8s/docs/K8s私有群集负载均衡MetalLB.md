# Kubernetes 私有群集负载均衡MetalLB

## k8s 服务暴露的三种方式

如果需要从集群外部访问服务，即将服务暴露给用户使用，Kubernetes Service 本身提供了两种方式，一种是 NodePort，另外一种是 LoadBalancer。另外 Ingress 也是一种常用的暴露服务的方式。

### NodePort

如果将服务的类型设置为 NodePort，kube-proxy 就会为这个服务申请一个 30000 以上的端口号（默认情况下），然后在集群所有主机上配置 IPtables 规则，这样用户就能通过集群中的任意节点加上这个分配的端口号访问服务了，如下图

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210928114007.png" width=600 />

NodePort 是最方便的暴露服务的方式，缺点也很明显：

- 基于SNAT进行访问，Pod无法看到真正的IP。
- NodePort是将群集中的一个主机作为跳板访问后端服务，所有的流量都会经过跳板机，很容易造成性能瓶颈和单点故障，难以用于生产环境。
- NodePort端口号一般都是用大端口，不容易记忆。

> NodePort 设计之初就不是用于生产环境暴露服务的方式，所以默认端口都是一些大端口。

### LoadBalancer

LoadBalancer 是 Kubernetes 提倡的将服务暴露给外部的一种方式。但是这种方式需要借助于云厂商提供的负载均衡器才能实现，这也要求了 Kubernetes 集群必须在云厂商上部署。LoadBalancer 的原理如下：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210928114633.png" width=600 />

LoadBalancer 通过云厂商的 LB 插件实现，LB 插件基于 Kubernetes.io/cloud-provider 这个包实现，这个包会自动选择合适的后端暴露给 LB 插件，然后 LB 插件由此创建对应的负载均衡器，网络流量在云服务端就会被分流，就能够避免 NodePort 方式的单点故障和性能瓶颈。

LoadBalancer 是 Kubernetes 设计的对外暴露服务的推荐方式，但是这种方式仅仅限于云厂商提供的 Kubernetes 服务上，对于物理部署或者非云环境下部署的 Kubernetes 集群，这一机制就存在局限性而无法使用。

### Ingress

Ingress 并不是 Kubernetes 服务本身提供的暴露方式，而是借助于软件实现的同时暴露多个服务的一种类似路由器的插件。Ingress 通过域名来区分不同服务，并且通过 annotation 的方式控制服务对外暴露的方式。其原理如下图：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210928134443.png" width=600 />

相比于 NodePort 和 LoadBalancer，Ingress 在企业业务场景中应该是使用的最多的，原因有：

- 相比 kube-proxy 的负载均衡，Ingress controller 能够实现更多的功能，诸如流量控制，安全策略等。

- 基于域名区分服务，更加直观。也不需要用到 NodePort 中的大端口号。

但是在实际场景中，Ingress 也需要解决下面的一些问题：

- 第一个问题，Ingress 也可以用于 L4，但是对于 L4 的应用，Ingress 配置过于复杂，最好的实现就是直接用 LB 暴露出去。

- 第二个问题，测试环境可以用 NodePort 将 Ingress Controller 暴露出去或者直接 hostnetwork，但也不可避免有单点故障和性能瓶颈，也无法很好的使用 Ingress-controller 的 HA 特性。

## MetalLB 介绍

MetalLB 是一个负载均衡器，专门解决裸金属 Kubernetes 集群中无法使用 LoadBalancer 类型服务的痛点。MetalLB 使用标准化的路由协议，以便裸金属 Kubernetes 集群上的外部服务也尽可能地工作。即 MetalLB 能够帮助你在裸金属 Kubernetes 集群中创建 LoadBalancer 类型的 Kubernetes 服务。

>项目地址：<https://github.com/metallb/metallb>

## MetalLB 工作原理

MetalLB 会在 Kubernetes 内运行，监控服务对象的变化，一旦监测到有新的 LoadBalancer 服务运行，并且没有可申请的负载均衡器之后，就会完成地址分配和外部声明两部分的工作。

### 地址分配

在云环境中，当你请求一个负载均衡器时，云平台会自动分配一个负载均衡器的 IP 地址给你，应用程序通过此 IP 来访问经过负载均衡处理的服务。

使用 MetalLB 时，MetalLB 会自己为用户的 LoadBalancer 类型 Service 分配 IP 地址，当然该 IP 地址不是凭空产生的，需要用户在配置中提供一个 IP 地址池，Metallb 将会在其中选取地址分配给服务。

### 外部声明

MetalLB 将 IP 分配给某个服务后，它需要对外宣告此 IP 地址，并让外部主机可以路由到此 IP。

MetalLB 支持两种声明模式：Layer 2（ ARP / NDP ）模式或者 BGP 模式。

#### Layer 2 模式

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210928135805.png" width=600 />

在任何以太网环境均可使用该模式。当在第二层工作时，将有一台机器获得 IP 地址（即服务的所有权）。MetalLB 使用标准的地址发现协议（对于 IPv4 是 ARP，对于 IPv6 是 NDP）宣告 IP 地址，使其在本地网路中可达。从 LAN 的角度来看，仅仅是某台机器多配置了一个 IP 地址。

Layer 2 模式下，每个 Service 会有集群中的一个 Node 来负责。服务的入口流量全部经由单个节点，然后该节点的 Kube-Proxy 会把流量再转发给服务的 Pods。也就是说，该模式下 MetalLB 并没有真正提供负载均衡器。尽管如此，MetalLB 提供了故障转移功能，如果持有 IP 的节点出现故障，则默认 10 秒后即发生故障转移，IP 会被分配给其它健康的节点。

Layer 2 模式的优缺点：

- Layer 2 模式更为通用，不需要用户有额外的设备；
- Layer 2 模式下存在单点问题，服务的所有入口流量经由单点，其网络带宽可能成为瓶颈；
- 由于 Layer 2 模式需要 ARP/NDP 客户端配合，当故障转移发生时，MetalLB 会发送 ARP 包来宣告 MAC 地址和 IP 映射关系的变化，地址分配略为繁琐。

#### BGP 模式

当在第三层工作时，集群中所有机器都和你控制的最接近的路由器建立 BGP 会话，此会话让路由器能学习到如何转发针对 K8S 服务 IP 的数据包。

通过使用 BGP，可以实现真正的跨多节点负载均衡（需要路由器支持 multipath），还可以基于 BGP 的策略机制实现细粒度的流量控制。

具体的负载均衡行为和路由器有关，可保证的共同行为是：每个连接（TCP 或 UDP 会话）的数据包总是路由到同一个节点上。

BGP 模式的优缺点：

- 不能优雅处理故障转移，当持有服务的节点宕掉后，所有活动连接的客户端将收到 Connection reset by peer ；

- BGP 路由器对数据包的源 IP、目的 IP、协议类型进行简单的哈希，并依据哈希值决定发给哪个 K8S 节点。问题是 K8S 节点集是不稳定的，一旦（参与 BGP）的节点宕掉，很大部分的活动连接都会因为 rehash 而坏掉。

BGP 模式问题的缓和措施：

- 将服务绑定到一部分固定的节点上，降低 rehash 的概率。

- 在流量低的时段改变服务的部署。

- 客户端添加透明重试逻辑，当发现连接 TCP 层错误时自动重试。

## 部署MetalLB

使用YAML文件部署

```bash
# 目前 MetalLB 最新版本为 0.10.2
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/manifests/metallb.yaml
```

部署完成后，将在 metallb-system 命名空间下将 MetalLB 部署到集群。YAML 文件中主要包含以下一些组件：

- metallb-system/controller，这是处理 IP 地址分配的控制器。
- metallb-system/speakerdaemonset 这是支持你选择协议以使服务可达的组件。
- Controller 和 Speaker 的 Service Accounts，以及组件需要运行的 RBAC 权限。

>通过 YAML 安装文件部署并不包含 MetalLB 配置文件，但 MetalLB 的组件仍能启动，但在你定义和部署 configmap 之前将保持空闲状态 。

## 配置MetalLB

MetalLB 安装完成后，我们还需要根据具体的地址和通告方式配置名为 metallb-system/config 的 ConfigMap。Controller 会读取该 ConfigMap，并重新加载配置。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: my-ip-space
      protocol: layer2
      addresses:
      - 192.168.100.171-192.168.100.175
```

> 注意： 这里的IP地址范围需要跟你实际群集情况对应

首先，我们使用 kubectl apply -f MetalLB-Layer2-Config.yaml 命令使配置生效。如果你想看到详细配置更新过程，可以使用以下类似命令查看。

```bash
kubectl logs -l component=speaker -n metallb-system
```

接下来，我们来创建一个服务类型为 LoadBalancer 的 mysql 服务来验证下。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-lb
  namespace: dev
  labels:
    app: mysql
spec:
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
  selector:
    app: mysql
  type: LoadBalancer
```

我们查看service状态

```bash
kubectl get svc -n dev
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
mysql-lb        LoadBalancer   172.19.55.116   192.168.100.171   3306:31575/TCP   7s
```
