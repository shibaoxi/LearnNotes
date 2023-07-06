# Grafana Loki 教程

## Loki 介绍

Loki 是 Grafana 团队开源的一款高可用、高拓展、多租户的日志聚合系统，和 ELK 的组件功能一样，Loki 有负责日志存储查询的主服务，有在客户端负责收集日志并推送的代理服务，还有 Grafana 最拿手的可视化面板展示。

不同的是，Loki 不再根据日志内容去建立大量的索引，而是借鉴了 Prometheus 核心的思想，使用标签去对日志进行特征标记，然后归集统计。这样的话，能避免大量的内存资源占用，转向廉价的硬盘存储。当然 Loki 会对日志进行分块存储并压缩，保留少量的元数据索引，兼顾硬盘的查询效率。

除此之外，Loki 还有以下特性：

- 一个 Loki 实例允许面向多个租户，不同租户的数据完全与其他租户隔离。
- LogQL：Loki 自己的日志查询语言，很容易上手使用的。
- 高拓展性，Loki 的所有组件可以跑在同一个程序里，也可以按微服务的方式去部署它们。
- 支持市面上许多流行的日志客户端插件，能较好的集合在一起。

## Loki架构

下面，我们来认识下 Loki 的总体架构。就像前面提及到的，Loki 主要分为了三部分：

- agent client：日志代理客户端，负责收集日志发送到主服务 Loki，目前官方有自己的 client： Promtail，也支持主流的组件，如 Fluentd、Logstash、Fluent Bit 等。
- loki：日志主服务，负责存储收集到的日志以及对日志查询解析。
- grafana：日志数据展示面板。

![20230706153004](https://s2.loli.net/2023/07/06/x2QeLj7faVFclpE.png)

可以看到，核心的组件其实是 Loki 这个主服务，关于它的内部组成，其实还可以细分为几部分：

- Distributor：负责处理客户端发送过来的日志数据，检验正确性后将其分发给后续组件。
- Ingester：负责将日志按块存储。
- Query frontend：可选服务，对外提供查询 API，负责一些查询调整和整合。

![20230706153144](https://s2.loli.net/2023/07/06/finFxvgEpXoY521.png)

## 使用Helm部署Promtail+Loki+Grafana

### 先决条件

- 一个具有控制平面和至少两个工作节点的Kubernetes集群，以了解Loki如何在可扩展模式下工作
- 在本地机器上安装并配置了[kubectl](https://kubernetes.io/docs/tasks/tools/)和[Helm](https://helm.sh/docs/intro/install/)

### 使用helm部署

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

配置helm国内镜像加速（可选）

```bash

# 查看当前配置的仓库地址
$ helm repo list
# 删除默认仓库，默认在国外pull很慢
$ helm repo remove stable
# 添加几个常用的仓库,可自定义名字
$ helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
$ helm repo add kaiyuanshe http://mirror.kaiyuanshe.cn/kubernetes/charts
$ helm repo add azure http://mirror.azure.cn/kubernetes/charts
$ helm repo add dandydev https://dandydeveloper.github.io/charts
$ helm repo add bitnami https://charts.bitnami.com/bitnami

```

#### 安装Loki

在继续之前，花点时间回顾一下Loki可用的图表:

```bash
helm search repo loki
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
grafana/loki                    5.8.8           2.8.2           Helm chart for Grafana Loki in simple, scalable...
grafana/loki-canary             0.12.0          2.8.2           Helm chart for Grafana Loki Canary                
grafana/loki-distributed        0.69.16         2.8.2           Helm chart for Grafana Loki in microservices mode 
grafana/loki-simple-scalable    1.8.11          2.6.1           Helm chart for Grafana Loki in simple, scalable...
grafana/loki-stack              2.9.10          v2.6.1          Loki: like Prometheus, but for logs.              
grafana/fluent-bit              2.5.0           v2.1.0          Uses fluent-bit Loki go plugin for gathering lo...
grafana/promtail                6.11.5          2.8.2           Promtail is an agent which ships the contents o...
```

创建命名空间

```bash
kubectl create ns loki
```

Loki需要一个兼容s3的对象存储解决方案来存储Kubernetes集群日志。为简单起见，本教程将使用MinIO。要启用它，您需要编辑Loki的Helm Chart值文件。

创建一个values.yml 文件，并添加如下内容

```bash
minio:
  enabled: true
```

运行如下命令安装loki

```bash
helm upgrade --install --namespace loki logging grafana/loki -f values.yml --set loki.auth_enabled=false --debug
```

#### 安装Promtail

获取values 文件

```bash
helm show values grafana/promtail > promtail-values.yaml
```

在文件中找到config 关键字，配置如下所示

```yml
config:
 logLevel: info
 serverPort: 3101
 lokiAddress: http://loki-gateway/loki/api/v1/push
```

运行如下命令安装promtail

```bash
helm upgrade --install promtail grafana/promtail --namespace=loki -f promtail-values.yaml
```

#### 安装grafana

运行如下命令安装grafana

```bash
helm upgrade --install --namespace=loki loki-grafana grafana/grafana --debug
```

部署lb service 或者 ingress

```yml
apiVersion: v1
kind: Service
metadata:
  name: loki-grafana-lb
  namespace: loki
spec:
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
  selector:
    app.kubernetes.io/instance: loki-grafana
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: LoadBalancer
```

#### 配置

安装完成后运行如下命令查看服务和pod状态

```bash
kubectl get all -n loki
```

获取grafana的密码

```bash
kubectl get secret --namespace loki loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

添加data source

![20230706160921](https://s2.loli.net/2023/07/06/YFlmi1gNrwte4a6.png)

向下滚动，直到找到Loki并单击它以将其添加为数据源。

![20230706160954](https://s2.loli.net/2023/07/06/3BmnogwLI6H4qcl.png)

在URL中输入 loki gateway的服务地址，如下：<http://loki-gateway.loki.svc.cluster.local>

![20230706161146](https://s2.loli.net/2023/07/06/MBh6yJoWS741gws.png)

在下面页面查询日志

![20230706161448](https://s2.loli.net/2023/07/06/Ebf1lXY4KAr8uqR.png)
