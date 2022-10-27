# Kubernetes Pronetheus 概述及部署

## Prometheus架构原理

Prometheus是一款开源的监控工具，它的基本实现原理是从exporter拉取数据，或者间接地通过网关gateway拉取数据（如果在k8s内部署，可以使用服务发现的方式），它默认本地存储抓取的所有数据，并通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中，采集到的数据有两个去向，一个是报警，另一个是可视化。以下是Prometheus最新版本的架构图：

![img](https://i.loli.net/2021/08/09/emxlZ385yCwWpYV.png)

Prometheus具有以下特点：

- 提供多维度数据模型和灵活的查询语言：通过将监控指标关联多个Tag将监控数据进行任意维度的组合；提供HTTP查询接口；可以很方便的结合Grafana等组件展示数据。
- 支持服务器节点的本地存储：通过prometheus自带的时序数据库，可以完成每秒千万级的数据存储。同时在保存大量历史数据的场景中，prometheus还可以对接第三方时序数据库如OpenTSDB等。
- 定义了开放指标数据标准：支持pull和push两种方式的数据采集，以基于HTTP的Pull方式采集时序数据，只有实现了prometheus监控数据格式才可以被prometheus采集；以Push方式向中间网关推送时序数据，能更灵活地应对各种监控场景。
- 支持通过静态文件配置和动态发现机制发现监控对象，自动完成数据采集。prometheus目前已经支持Kubernetes、Consul等多种服务发现机制，可以减少运维人员的手动配置环节。

### Prometheus组件介绍

从Prometheus的架构图中可以看到，Prometheus主要有四大组件Prometheus Server、Push gateway、Exporters和Alertmanager，分别如下：

- Prometheus Server：负责从Exporter拉取和存储监控数据，根据告警规则产生告警并发送给Alertmanager，并提供一套灵活的查询语言PromQL。
- Exporters/Jobs：Prometheus的数据采集组件，负责收集目标对象（host, container…）的性能数据，并通过HTTP接口提供给Prometheus Server。支持数据库、硬件、消息中间件、存储系统、http服务器、jmx等。
- Short-lived jobs：瞬时任务的场景，无法通过pull方式拉取，需要使用push方式，与PushGateway搭配使用.
- PushGateway：应对部分push场景的组件可选组件，这部分监控数据先推送到Push Gateway上，然后再由Prometheus Server端拉取 。用于存在时间较短，可能在Prometheus来拉取之前就消失了的 jobs.
- Alertmanager：从Prometheus server端接收到alerts后，会基于PromQL的告警规则分析数据，如果满足PromQL定义的规则，则会产生一条告警，并发送告警信息到Alertmanager，Alertmanager则是根据配置处理告警信息并发送。常见的接收方式有：电子邮件，pagerduty，OpsGenie, webhook 等。
- Service Discovery：Prometheus支持多种服务发现机制：文件、DNS、Consul、Kubernetes、OpenStack、EC2等等。基于服务发现的过程是通过第三方提供的接口，Prometheus查询到需要监控的Target列表，然后轮训这些Target获取监控数据。

### Prometheus 存储机制

Prometheus以时间序列的方式将数据存储在本地硬盘，按照两个小时为一个时间窗口，将两小时内产生的数据存储在一个块(Block)中，每一个块又分为多个chunks，其中包含该时间窗口内的所有样本数据(chunks)，元数据文件(meta.json)以及索引文件(index)。

![img](https://i.loli.net/2021/08/09/OJjI2c3Z48he6Vf.png)

当前时间窗口内正在收集的样本数据会直接保存在内存当中，达到2小时后写入磁盘，这样可以提高Prometheus的查询效率。为了防止程序崩溃导致数据丢失，实现了WAL（write-ahead-log）机制，启动时会以写入日志(WAL)的方式来实现重播，从而恢复数据。此期间如果通过API删除时间序列，删除记录也会保存在单独的逻辑文件当中(tombstone)，而不是立即从chunk文件中删除。

### Prometheus中metrics类型

Prometheus中主要有以下metrics类型：

- Gauges：仪表盘类型，可增可减，如CPU使用率，内存使用率，集群节点个数，大部分监控数据都是这种类型的。
- Counters：计数器类型，只增不减，如机器的启动时间，HTTP访问量等。机器重启不会置零，在使用这种指标类型时，通常会结合rate()方法获取该指标在某个时间段的变化率。
- Histograms：柱状图，用于观察结果采样，分组及统计，如：请求持续时间，响应大小。其主要用于表示一段时间内对数据的采样，并能够对其指定区间及总数进行统计。
- Summary：类似Histogram，用于表示一段时间内数据采样结果，其直接存储quantile数据，而不是根据统计区间计算出来的。不需要计算，直接存储结果。

## 部署 kube-prometheus

> kube-prometheus 的github地址：
> <https://github.com/coreos/kube-prometheus/>

### 下载kube-prometheus

```bash
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.8.0.tar.gz

tar xf v0.8.0.tar.gz
```

### 修改配置

修改 grafana-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 7.5.4
  name: grafana
  namespace: monitoring
spec:
  type: NodePort        #添加内容
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 30100     #添加内容
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
```

修改prometheus-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.26.0
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  type: NodePort        #添加内容
  ports:
  - name: web
    port: 9090
    targetPort: web
    nodePort: 30200     #添加内容
  selector:
    app: prometheus
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
```

修改 alertmanager-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    alertmanager: main
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.21.0
  name: alertmanager-main
  namespace: monitoring
spec:
  type: NodePort        #添加内容
  ports:
  - name: web
    port: 9093
    targetPort: web
    nodePort: 30300     #添加内容
  selector:
    alertmanager: main
    app: alertmanager
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
  sessionAffinity: ClientIP
```

### kube-prometheus镜像版本查看与下载

由于镜像都在国外，因此经常会下载失败。为了快速下载镜像，这里我们下载国内的镜像，然后tag为配置文件中的国外镜像名即可。

查看kube-prometheus的镜像信息

```bash
[root@cli manifests]# grep -riE 'quay.io|k8s.gcr|grafana/' *
alertmanager-alertmanager.yaml:  image: quay.io/prometheus/alertmanager:v0.21.0
blackbox-exporter-deployment.yaml:        image: quay.io/prometheus/blackbox-exporter:v0.18.0
blackbox-exporter-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.8.0
grafana-deployment.yaml:        image: grafana/grafana:7.5.4
grafana-deployment.yaml:        - mountPath: /etc/grafana/provisioning/datasources
grafana-deployment.yaml:        - mountPath: /etc/grafana/provisioning/dashboards
kube-state-metrics-deployment.yaml:        image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.0.0
kube-state-metrics-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.8.0
kube-state-metrics-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.8.0
node-exporter-daemonset.yaml:        image: quay.io/prometheus/node-exporter:v1.1.2
node-exporter-daemonset.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.8.0
prometheus-prometheus.yaml:  image: quay.io/prometheus/prometheus:v2.26.0
setup/prometheus-operator-deployment.yaml:        - --prometheus-config-reloader=quay.io/prometheus-operator/prometheus-config-reloader:v0.47.0
setup/prometheus-operator-deployment.yaml:        image: quay.io/prometheus-operator/prometheus-operator:v0.47.0
setup/prometheus-operator-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.8.0
```

编写下载脚本：download_prometheus_image.sh

```bash
#!/bin/sh

##### 在master节点和worker节点都要执行

#加载环境变量

. /etc/profile
. /etc/bashrc

################################
#docker hub 镜像站
src_registry="davidshi"
# 镜像下载及重命名
image_name="alertmanager:v0.21.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/prometheus/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="blackbox-exporter:v0.18.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/prometheus/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="kube-rbac-proxy:v0.8.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/brancz/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="grafana:7.5.4"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} grafana/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="kube-state-metrics:v2.0.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} k8s.gcr.io/kube-state-metrics/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="node-exporter:v1.1.2"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/prometheus/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="prometheus:v2.26.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/prometheus/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="prometheus-operator:v0.47.0"
docker pull ${src_registry}/${image_name}  && docker tag ${src_registry}/${image_name} quay.io/prometheus-operator/${image_name}  && docker rmi ${src_registry}/${image_name}
echo "====================== ${image_name} download complete =============="

image_name="prometheus-config-reloader:v0.47.0"
docker pull davidshi/${image_name}  && docker tag davidshi/${image_name} quay.io/prometheus-operator/${image_name}  && docker rmi davidshi/${image_name}
echo "====================== ${image_name} download complete =============="


echo "********** prometheus docker images OK! **********"
```

镜像下载完成后执行安装

```bash
#先执行
kubectl apply -f manifests/setup/
#然后执行
kubectl apply -f manifests/
```

所有pod都正常运行后即可访问Prometheus和grafana。
