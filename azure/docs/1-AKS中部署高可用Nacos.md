# 使用AKS群集部署高可用Nacos

## Nacos介绍

### 概念

Nacos /nɑ:kəʊs/ 是 Dynamic Naming and Configuration Service的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

### 关键特性

- **服务发现和服务健康监测**
  Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用 原生SDK、OpenAPI、或一个独立的Agent TODO注册 Service 后，服务消费者可以使用DNS TODO 或HTTP&API查找和发现服务。
  Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。Nacos 支持传输层 (PING 或 TCP)和应用层 (如 HTTP、MySQL、用户自定义）的健康检查。 对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查，Nacos 提供了 agent 上报模式和服务端主动检测2种健康检查模式。Nacos 还提供了统一的健康检查仪表盘，帮助您根据健康状态管理服务的可用性及流量。
- **动态配置服务**
    &nbsp;
    动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置.
    &nbsp;
    动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。
    &nbsp;
    配置中心化管理让实现无状态服务变得更简单，让服务按需弹性扩展变得更容易。
    &nbsp;
    Nacos 提供了一个简洁易用的UI  帮助您管理所有的服务和应用的配置。Nacos 还提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。
- **动态 DNS 服务**
    动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。动态DNS服务还能让您更容易地实现以 DNS 协议为基础的服务发现，以帮助您消除耦合到厂商私有服务发现 API 上的风险。
- **服务及其元数据管理**
    Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

## 部署

>在高级使用中,Nacos在K8S拥有自动扩容缩容和数据持久特性,请注意如果需要使用这部分功能请使用PVC持久卷,Nacos的自动扩容缩容需要依赖持久卷,以及数据持久化也是一样，这里使用Azure file 作为持久存储

### 创建Azure File 存储类

创建azure-file-sc.yaml文件，内如如下：

```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aks-azurefile
provisioner: file.csi.azure.com # replace with "kubernetes.io/azure-file" if aks version is less than 1.21
allowVolumeExpansion: true
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
  - mfsymlinks
  - cache=strict
  - actimeo=30
parameters:
  skuName: Standard_LRS
```

## 初始化数据库

初始化数据库的语句 <https://github.com/alibaba/nacos/blob/develop/distribution/conf/mysql-schema.sql>

## 集成Azure KeyVault

> 把外部数据库的连接方式：主机地址、数据库名称、数据库端口、用户名、密码等，写入Azure Key Vault

将现有的 AKS 群集升级为带有适用于机密存储 CSI 驱动程序的 Azure 密钥保管库提供程序功能，请将 az aks enable-addons 命令与加载项 azure-keyvault-secrets-provider 配合使用：

```azurecli
az aks enable-addons --addons azure-keyvault-secrets-provider --name myAKSCluster --resource-group myResourceGroup
```

通过列出在 kube-system 命名空间中具有 secrets-store-csi-driver 和 secrets-store-provider-azure 标签的所有 Pod 来验证安装是否完成，并确保输出类似于下面显示的输出：

```azurecli
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver, secrets-store-provider-azure)'
NAME                                     READY   STATUS    RESTARTS   AGE
aks-secrets-store-csi-driver-hj8wz       3/3     Running   0          3h7m
aks-secrets-store-csi-driver-w5nrq       3/3     Running   0          24h
aks-secrets-store-csi-driver-wx6l4       3/3     Running   0          24h
aks-secrets-store-provider-azure-92pnl   1/1     Running   0          24h
aks-secrets-store-provider-azure-9ctjt   1/1     Running   0          24h
aks-secrets-store-provider-azure-ffbw9   1/1     Running   0          3h7m
```

把aks 主机标识赋予Azure Key Vault 相关权限
<https://learn.microsoft.com/zh-cn/azure/aks/csi-secrets-store-identity-access>

创建并编辑secretProviderClass.yaml文件

```yml
# This is a SecretProviderClass example using system-assigned identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-appkv-system-msi
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"    # Set to true for using managed identity
    userAssignedIdentityID: ""      # If empty, then defaults to use the system assigned identity on the VM
    keyvaultName: app-KV-CN3-DEV
    cloudName: "azurechinacloud"                   # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: dbhost
          objectType: secret       
          objectVersion: "" 
        - |
          objectName: dbport
          objectType: secret       
          objectVersion: ""
        - |
          objectName: dbname
          objectType: secret       
          objectVersion: "" 
        - |
          objectName: nacosdbuser
          objectType: secret       
          objectVersion: "" 
        - |
          objectName: nacosdbpw
          objectType: secret       
          objectVersion: ""       
    tenantId: 7d281bcf-d0f7-431b-8f2f-XXXXXXX           # The tenant ID of the key vault
  secretObjects:
  - secretName: appsecrets
    type: Opaque
    data:
    - key: dbhost
      objectName: dbhost
    - key: dbport
      objectName: dbport
    - key: dbname
      objectName: dbname
    - key: nacosdbuser
      objectName: nacosdbuser
    - key: nacosdbpw
      objectName: nacosdbpw
```

## 部署Nacos

创建nacos-pvc.yaml文件并编辑

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: app-nacos-sit-headless
  # annotations:
  #   service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - port: 8848
      name: server
      targetPort: 8848
    - port: 9848
      name: client-rpc
      targetPort: 9848
    - port: 9849
      name: raft-rpc
      targetPort: 9849
    ## 兼容1.4.x版本的选举端口
    - port: 7848
      name: old-raft-rpc
      targetPort: 7848
  selector:
    app: nacos

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
spec:
  serviceName: nacos-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                      - nacos
              topologyKey: "kubernetes.io/hostname"
      initContainers:
        - name: peer-finder-plugin-install
          image: nacos/nacos-peer-finder-plugin:1.1
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: /home/nacos/plugins/peer-finder
              name: data
              subPath: peer-finder
      containers:
        - name: nacos
          imagePullPolicy: Always
          image: nacos/nacos-server:latest
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
          ports:
            - containerPort: 8848
              name: client-port
            - containerPort: 9848
              name: client-rpc
            - containerPort: 9849
              name: raft-rpc
            - containerPort: 7848
              name: old-raft-rpc
          env:
            - name: NACOS_REPLICAS
              value: "3"
            - name: SERVICE_NAME
              value: "nacos-headless"
            - name: DOMAIN_NAME
              value: "cluster.local"
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
            - name: PREFER_HOST_MODE
              value: "hostname"
            - name: MYSQL_SERVICE_DB_PARAM
              value: "characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=true&serverTimezone=Asia/Shanghai" #连接8.0需要添加，否则连不上
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                secretKeyRef:
                  name: appsecrets
                  key: dbhost
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: appsecrets
                  key: dbname
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                secretKeyRef:
                  name: appsecrets
                  key: dbport
            - name: MYSQL_SERVICE_USER
              valueFrom:
                secretKeyRef:
                  name: appsecrets
                  key: nacosdbuser
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: appsecrets
                  key: nacosdbpw 
          volumeMounts:
            - name: data
              mountPath: /home/nacos/plugins/peer-finder
              subPath: peer-finder
            - name: data
              mountPath: /home/nacos/data
              subPath: data
            - name: data
              mountPath: /home/nacos/logs
              subPath: logs
            - name: secrets-store
              mountPath: "/mnt/secrets-store"
              readOnly: true
      volumes:
        - name: secrets-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "azure-appkv-system-msi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteMany
        storageClassName: aks-azurefile
        resources:
          requests:
            storage: 20Gi
  selector:
    matchLabels:
      app: nacos

---
apiVersion: v1
kind: Service
metadata:
  name: apollo-nacos-sit-nlb
  namespace: apollo-sit
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
    - port: 8848
      name: server
      targetPort: 8848
    # - port: 9848
    #   name: client-rpc
    #   targetPort: 9848
    # - port: 9849
    #   name: raft-rpc
    #   targetPort: 9849
    # ## 兼容1.4.x版本的选举端口
    # - port: 7848
    #   name: old-raft-rpc
    #   targetPort: 7848
  selector:
    app: nacos

```

创建完成后查找负载均衡的IP地址，直接访问 <http://172.16.22.41:8848/nacos/> 默认用户名密码为nacos/nacos

更改密码

![20221025131036](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221025131036.png)
