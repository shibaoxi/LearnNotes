# Kubernetes Pronetheus 概述及部署

## Prometheus架构原理

Prometheus是一种开源的监控和警报工具，用于收集和记录应用程序和系统的度量数据。它特别适用于在Kubernetes集群中监控容器化应用程序。Kubernetes集群中通常与Prometheus一起使用的组件是Prometheus Operator和Grafana。。以下是Prometheus最新版本的架构图：

![img](https://i.loli.net/2021/08/09/emxlZ385yCwWpYV.png)

以下是在Kubernetes中使用Prometheus的主要步骤：

- **安装Prometheus Operator**： Prometheus Operator是一种Kubernetes控制器，用于简化Prometheus的部署和管理。您可以通过在Kubernetes中部署Prometheus Operator来自动设置和管理Prometheus实例。

- **配置Prometheus实例**： Prometheus Operator将通过Kubernetes的自定义资源定义（CRD）创建和管理Prometheus实例。您可以使用PrometheusRule CRD定义监控规则，并使用ServiceMonitor CRD定义需要监控的目标（例如Kubernetes服务）。

- **配置和导入Dashboard**： Grafana通常与Prometheus一起使用，用于可视化监控指标。您可以在Grafana中导入Prometheus的预定义仪表板或自定义仪表板来查看和分析度量数据。

- **监控应用程序和系统**： Prometheus通过HTTP端点从目标应用程序和系统中拉取度量数据。您可以在应用程序中暴露Prometheus格式的度量数据，并在ServiceMonitor中定义用于监控的目标。

- **警报配置**： Prometheus还支持配置警报规则，以便在达到特定阈值或条件时触发警报。警报规则可以定义为PrometheusRule CRD。

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

### 常见的几款监控工具

以下这些工具可以用于在 Kubernetes 集群中实现监控和指标收集，以便于监视集群中的各种资源和应用的性能。

- **Heapster**：Heapster 是一个 Kubernetes 集群的资源监控工具，用于收集和汇总资源使用情况数据，如 CPU、内存、网络等。

- **Metrics Server**：Metrics Server 是 Kubernetes 官方提供的一个轻量级指标收集器，用于提供节点和 Pod 等资源的实时性能指标，可以用于水平自动扩展等。

- **Prometheus Operator**：Prometheus Operator 是一个 Kubernetes 控制器，用于管理和部署 Prometheus 和相关的监控组件。它可以自动创建和管理 Prometheus 实例、ServiceMonitor 和其他配置。

- **kube-prometheus 或 kube-prometheus-stack**：这是一个基于 Prometheus 的 Kubernetes 集群监控解决方案。它包含了一系列组件，用于部署和管理 Prometheus、Alertmanager、Grafana 等，以实现对 Kubernetes 集群和应用的全面监控。

#### 这几款工具之间的对比

##### kube-prometheus 和 kube-prometheus-stack 区别

"**kube-prometheus**" 和 "**kube-prometheus-stack**" 本质上是同一个项目，只是在不同的时间和版本中使用了不同的名称。"kube-prometheus-stack" 是 "kube-prometheus" 项目的更新版本，它提供了更多的功能、改进和修复。

- 最初，项目被称为 "kube-prometheus"，但随着时间的推移，项目团队对项目进行了大量的改进和扩展，并将其重命名为 "kube-prometheus-stack"，以更好地反映其提供的综合性监控解决方案。

- "kube-prometheus-stack"（或简称 "kube-prometheus"）是一个在 Kubernetes 集群中部署和管理 Prometheus 监控系统以及相关组件的综合解决方案。它集成了 Prometheus、Grafana、Alertmanager 等一系列组件，还包括预配置的监控规则和仪表盘，以及一键部署功能。用户可以通过部署 "kube-prometheus-stack" 来快速启动一个全面的 Kubernetes 集群监控系统，无需逐个配置各个组件。

总结起来，"kube-prometheus-stack" 是 "kube-prometheus" 项目的更新版本，提供更多的功能和改进，是一个便捷的综合性监控解决方案，适合在 Kubernetes 环境中快速部署和使用。

##### Prometheus Operator 和kube-prometheus 或 kube-prometheus-stack对比

"Prometheus Operator" 和 "kube-prometheus"（或 "kube-prometheus-stack"）都是用于在 Kubernetes 集群中部署和管理 Prometheus 监控系统的工具。它们有一些相似之处，但也存在一些区别。以下是它们的主要特点和区别的对比：

**Prometheus Operator：**

- 核心功能：Prometheus Operator 是一个 Kubernetes 控制器，专门用于管理 Prometheus 和相关组件的配置和部署。它自动创建和管理 Prometheus 实例、ServiceMonitor、Alertmanager、PrometheusRule 等 Kubernetes 资源。

- 声明式配置：Prometheus Operator 通过自定义资源定义（Custom Resource Definitions，CRDs）来实现声明式配置。您可以创建 Prometheus、ServiceMonitor 等资源对象来定义监控配置，Operator 会根据这些定义自动创建和维护相关的资源。

- 自动发现：Prometheus Operator 支持自动发现 Kubernetes 中的 Service、Pod、Namespace 等资源，无需手动配置每个监控目标。

- 生态系统整合：Prometheus Operator 集成了 Grafana 和 Alertmanager，并可以轻松与其他监控工具集成。

- 灵活性：Prometheus Operator 允许根据不同的需求和配置选择性地部署多个 Prometheus 实例，每个实例可以针对特定的监控任务进行配置。

**kube-prometheus 或 kube-prometheus-stack：**

- 综合解决方案：kube-prometheus（或 kube-prometheus-stack）是一个完整的监控解决方案，集成了 Prometheus、Grafana、Alertmanager 等一系列组件，以及一些预配置的监控规则和仪表盘。

- 快速启动：kube-prometheus 提供了一键式的部署方式，适合快速启动一个完整的监控系统，无需逐个配置各个组件。

- 预配置规则和仪表盘：kube-prometheus 提供了一些默认的监控规则和 Grafana 仪表盘，可以快速启用监控功能。

- 集成和扩展：由于 kube-prometheus 集成了多个组件，您可以使用这个解决方案来快速部署一个全面的监控系统，并且可以根据需要进行定制和扩展。

综合来看，Prometheus Operator 专注于 Prometheus 和相关资源的管理和自动化配置，而 kube-prometheus 或 kube-prometheus-stack 则是一个更加综合的解决方案，适合快速启动一个完整的监控系统，尤其对于刚开始使用 Prometheus 的用户来说，可以减少配置的复杂性。您可以根据实际需求和情况选择合适的工具。

#### Prometheus Operator 架构

![20230830171044](https://s2.loli.net/2023/08/30/t4rYxjNPnCA1usc.png)

Prometheus Operator 是一个用于在 Kubernetes 集群中自动化部署和管理 Prometheus 监控系统的控制器。它采用了声明式配置的方式，通过 Kubernetes 自定义资源定义（Custom Resource Definitions，CRDs）来定义和管理 Prometheus、ServiceMonitor、Alertmanager、PrometheusRule 等资源对象。以下是 Prometheus Operator 的架构说明：

- Prometheus Operator 控制器：Prometheus Operator 控制器是一个运行在 Kubernetes 集群中的控制器，负责监听 Prometheus 相关的自定义资源变化，根据变化自动执行相应的操作。

- Prometheus CRD：Prometheus Operator 引入了自定义资源定义（CRD） Prometheus，用于定义 Prometheus 实例的配置。在 Prometheus CRD 中，您可以定义监控的规则、数据存储、数据保留策略等。

- ServiceMonitor CRD：ServiceMonitor 是另一个自定义资源，用于定义要监控的应用程序。每个 ServiceMonitor 都关联到一个或多个 Kubernetes 的 Service，Prometheus Operator 将自动发现这些关联的服务，并生成适当的监控配置。

- Alertmanager CRD：类似于 Prometheus 和 ServiceMonitor，Prometheus Operator 还支持 Alertmanager 自定义资源，用于定义 Alertmanager 实例的配置。

- PrometheusRule CRD：PrometheusRule 自定义资源用于定义 Prometheus 的告警规则。通过这些规则，您可以指定应该在 Prometheus 中生成哪些告警。

- 自动发现和配置生成：Prometheus Operator 根据定义的 ServiceMonitor 和 PrometheusRule 自动发现和生成相应的监控配置。它会监听 Kubernetes 中的变化，如服务的创建、删除或标签的变更，以及规则的更新，然后自动更新 Prometheus 的配置文件。

- Prometheus 部署：Prometheus Operator 会基于 Prometheus 自定义资源的定义，在 Kubernetes 集群中部署 Prometheus 实例。Operator 负责管理配置、Pod 的生命周期、版本升级等。

- 集成 Grafana 和 Alertmanager：Prometheus Operator 通常也与 Grafana 和 Alertmanager 集成，可以配置 Grafana 和 Alertmanager 自定义资源，以便自动部署和配置这些组件。

## 快速在k8s内搭建 Prometheus 全家桶

> kube-prometheus 的github地址：
> <https://github.com/prometheus-operator/kube-prometheus/>

### 直接安装方式（kube-prometheus）

#### 下载kube-prometheus

> 下载地址：<https://github.com/prometheus-operator/kube-prometheus>

```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus
```

>【注】在 release-0.11 版本之后新增了 NetworkPolicy 默认是允许自己访问，如果了解 NetworkPolicy 可以修改一下默认的规则，可以用查看 ls manifests/*networkPolicy*，如果不修改的话则会影响到修改 NodePort 类型也无法访问，如果不会 Networkpolicy 可以直接删除就行。

#### 修改镜像源

> 国外镜像源某些镜像无法拉取，我们这里修改prometheus-operator，prometheus，alertmanager，kube-state-metrics，node-exporter，prometheus-adapter的镜像源为国内镜像源。我这里使用的是中科大的镜像源

```bash
# 查找
grep -rn 'quay.io' *
# 批量替换
sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' `grep "quay.io" -rl *`
# 再查找
grep -rn 'quay.io' *
grep -rn 'image: ' *

sed -i 's|registry.k8s.io|m.daocloud.io/registry.k8s.io|g' $(grep "registry.k8s.io" -rl *)

```

#### 修改 service 配置类型为 NodePort

> 为了可以从外部访问 prometheus，alertmanager，grafana，我们这里修改 promethes，alertmanager，grafana的 service 类型为 NodePort 类型。

##### 修改 prometheus 的 service

```bash
# 设置对外访问端口，增加如下两行，完整配置也贴出来了。
# type: NodePort
# nodePort: 30090

vim manifests/prometheus-service.yaml

```

完整配置

```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: prometheus-k8s
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    targetPort: web
    nodePort: 30090
  - name: reloader-web
    port: 8080
    targetPort: reloader-web
  selector:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
  sessionAffinity: ClientIP
```

##### 修改 grafana 的 service

```bash
# 设置对外访问端口，增加如下两行，完整配置也贴出来了。
# type: NodePort
# nodePort: 30300
vim manifests/grafana-service.yaml

```

完整配置

```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 9.5.3
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 30300
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
```

##### 修改 alertmanager 的 service

```bash
# 设置对外访问端口，增加如下两行，完整配置也贴出来了。
# type: NodePort
# nodePort: 30093
vim alertmanager-service.yaml
```

完整配置

```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.26.0
  name: alertmanager-main
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9093
    targetPort: web
    nodePort: 30093
  - name: reloader-web
    port: 8080
    targetPort: reloader-web
  selector:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
  sessionAffinity: ClientIP
```

#### 开始安装

```bash
kubectl apply --server-side -f manifests/setup
kubectl wait \
    --for condition=Established \
    --all CustomResourceDefinition \
    --namespace=monitoring
kubectl apply -f manifests/

# 查看
kubectl get all -n monitoring

```

#### 浏览器访问

Prometheus：<http://ip:30090/>

![20230831173517](https://s2.loli.net/2023/08/31/yMI1QqsxU5FpPhY.png)

Grafana ：<http://ip:30300/>
默认账号/密码：admin/admin

![20230831173716](https://s2.loli.net/2023/08/31/tQEo9yfhcGMSYmd.png)

参考：
<https://www.cnblogs.com/liugp/p/17661155.html>
