# Azure Kubernetes 服务介绍及操作指南

## 简介及组件介绍

### Kubernetes 介绍

Kubernetes 是一个快速发展的平台，用于管理基于容器的应用程序及其相关网络和存储组件。 Kubernetes 重点关注应用程序工作负载，而不是底层基础结构组件。 Kubernetes 提供了一种声明性的部署方法，由一组针对管理操作的强大 API 提供支持。

你可构建和运行可移植的、基于微服务的现代应用程序，从而使用 Kubernetes 安排和管理这些应用程序组件的可用性。 由于团队通过采用基于微服务的应用程序而取得进展，因此 Kubernetes 支持无状态和有状态应用程序。

作为开放平台，Kubernetes 可使用首选的编程语言、OS、库或消息总线生成应用程序。 现有的持续集成和持续交付 (CI/CD) 工具可以与 Kubernetes 集成，以计划和部署版本。

#### AKS 简介

AKS 提供一项托管 Kubernetes 服务，它可降低部署和核心管理任务（例如升级协调）的复杂性。 由 Azure 平台来管理 AKS 控制平面，你只需为运行应用程序的 AKS 节点付费。 AKS 建立在开源 Azure Kubernetes 服务引擎 aks-engine

### Kubernetes 群集体系结构

Kubernetes 群集分为两个组件：

- 控制平面：提供 Kubernetes 核心服务和应用程序工作负载的业务流程。
- 节点：运行应用程序工作负载。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926133055.png" width=600 />

#### 控制面板

创建 AKS 群集时，系统会自动创建和配置控制平面。 此控制平面作为提取自用户的 Azure 托管资源免费提供。 你只需为附加到 AKS 群集的节点付费。 控制平面及其资源仅驻留在创建群集的区域。

#### 节点和节点池

要运行应用程序和支持服务，需要 Kubernetes 节点。 一个 AKS 群集至少有一个节点，这是运行 Kubernetes 节点组件和容器运行时的 Azure 虚拟机 (VM)。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926135949.png" width=600 />

## 在Azure China 部署 AKS

目前可以在中国 东二或者北二创建AKS，你可以通过azure Portal 或者 Azure CLI 来创建AKS，本文将展示使用azure cli 创建 AKS。

> 你需要安装最新版本的 Azure CLI , 具体安装信息参考[官网](https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli?view=azure-cli-latest)。

### 1. 使用 azure cli 登录 azure china

```bash
# 设置 cli 登录环境为azure china
az cloud set --name AzureChinaCloud
# 登录
az login
# 列出 account
az account list
# 如果有多个订阅，手动指定订阅名称
az account set -s <subscription-name>
```

### 2. 获取可用的AKS版本

```bash
az aks get-versions -l chinaeast2 -o table
KubernetesVersion    Upgrades
-------------------  -----------------------
1.21.2               None available
1.21.1               1.21.2
1.20.9               1.21.1, 1.21.2
1.20.7               1.20.9, 1.21.1, 1.21.2
1.19.13              1.20.7, 1.20.9
1.19.11              1.19.13, 1.20.7, 1.20.9
```

### 3. 创建 AKS cluster

```bash
# 定义变量
$RESOURCE_GROUP_NAME="demo-aks"
$CLUSTER_NAME="demo-aks"
$LOCATION="chinaeast2"
$VERSION="1.21.1"
# 创建资源组
az group create -n $RESOURCE_GROUP_NAME -l $LOCATION
# 创建 有一个节点的 AKS Cluster
az aks create -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME --node-count 1 --node-vm-size B2s --generate-ssh-keys --kubernetes-version $VERSION -l $LOCATION --node-osdisk-size 128
# 等待 AKS 创建完成

```

### 4. 管理 AKS 群集

```bash
# 获取 AKS 认证信息
az aks get-credentials -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME
# 检查节点状态
kubectl get node
# 打开 kubernetes 面板
az aks browse --resource-group $RESOURCE_GROUP_NAME -n $CLUSTER_NAME
# 缩放节点池
az aks scale -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME --node-count=2
# 删除群集节点
az aks delete -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME
```

## 演示：准备运行应用程序到AKS上

### 1. 为AKS 准备应用程序

若要完成本教程，需要运行 Linux 容器的本地 Docker 开发环境。 Docker 提供的包可在 [Mac](https://docs.docker.com/docker-for-mac/)、[Windows](https://docs.docker.com/docker-for-windows/) 或 [Linux](https://docs.docker.com/engine/installation/#supported-platforms) 系统上配置 Docker。

1.1 获取示例应用程序代码
> 示例应用程序是一个包含前端 Web 组件和后端 Redis 实例的基本投票应用。 Web 组件打包到自定义容器映像中。 Redis 实例使用 Docker 中心提供的未修改的映像。

```bash
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
# 进入目录
 cd .\azure-voting-app-redis\
```

1.2 目录内包含应用程序源代码、预创建的 Docker Compose 文件和 Kubernetes 清单文件。 整套教程都会使用这些文件。 目录的内容和结构如下所示：

```bash
D:.
│  .gitignore
│  azure-vote-all-in-one-redis.yaml
│  docker-compose.yaml
│  LICENSE
│  README.md
│
├─azure-vote
│  │  app_init.supervisord.conf
│  │  Dockerfile
│  │  Dockerfile-for-app-service
│  │  sshd_config
│  │
│  └─azure-vote
│      │  config_file.cfg
│      │  main.py
│      │
│      ├─static
│      │      default.css
│      │
│      └─templates
│              index.html
│
└─jenkins-tutorial
        config-jenkins.sh
        deploy-jenkins-vm.sh
```

1.3 使用示例 docker-compose.yaml 文件创建容器映像、下载 Redis 映像和启动应用程序：

```bash
docker-compose up -d
```

1.4 完成后，使用 docker images 命令查看创建的映像。 已下载或创建三个映像。 azure-vote-front 映像包含前端应用程序，并将 nginx-flask 映像用作基础映像。 redis 映像用于启动 Redis 实例。

```bash
docker images
REPOSITORY                                     TAG       IMAGE ID       CREATED         SIZE
mcr.microsoft.com/azuredocs/azure-vote-front   v1        328eb5aa2e1c   6 minutes ago   904MB
mcr.microsoft.com/oss/bitnami/redis            6.0.8     3a54a920bb6c   12 months ago   103MB
```

1.5 运行 docker ps 命令，查看正在运行的容器：

```bash
docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED         STATUS         PORTS                                            NAMES
7feee0ac037e   mcr.microsoft.com/oss/bitnami/redis:6.0.8         "/opt/bitnami/script…"   4 minutes ago   Up 4 minutes   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp        azure-vote-back
1fa21f0bbcc1   mcr.microsoft.com/azuredocs/azure-vote-front:v1   "/entrypoint.sh /sta…"   7 minutes ago   Up 2 minutes   443/tcp, 0.0.0.0:8080->80/tcp, :::8080->80/tcp   azure-vote-front
```

1.6 若要查看正在运行的应用程序，请在本地 Web 浏览器中输入 <http://localhost:8080>。 示例应用程序会加载，如以下示例所示：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926160437.png" width=600 />

### 2. 将镜像推送到ACR

2.1 若要将 azure-vote-front 容器映像与 ACR 配合使用，需使用注册表的登录服务器地址对映像进行标记。 在将容器映像推送到映像注册表时，使用此标记进行路由。

>若要获取登录服务器地址，请使用 az acr list 命令并查询是否存在 loginServer，如下所示

```bash
az acr list --resource-group Demo-ACR-RG --query "[].{acrLoginServer:loginServer}" --output table
```

2.2 现在，请使用容器注册表的 acrLoginServer 地址标记本地 azure-vote-front 映像。 若要指示映像版本，请将 :v1 添加到映像名称的末尾：

```bash
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrLoginServer>/azure-vote-front:v1
```

2.3 生成并标记映像后，将 azure-vote-front 映像推送到 ACR 实例。 使用 docker push 并提供自己的适用于映像名称的 acrLoginServer 地址，如下所示：

```bash
#推送之前需要先登录acr
az acr login --name <acrName>
#推送
docker push <acrLoginServer>/azure-vote-front:v1
```

2.4 若要返回已推送到 ACR 实例的映像列表，请使用 az acr repository list 命令。 按如下所示提供自己的 \<acrName>

```bash
az acr repository list --name <acrName> --output table
Result
----------------
azure-vote-front
```

### 3. 运行应用程序

若要部署此应用程序，必须更新 Kubernetes 清单文件中的映像名称，使之包括 ACR 登录服务器名称。

3.1 在第一步中克隆的 git 存储库中的示例清单文件使用登录服务器名称 microsoft。 确保位于所克隆的 azure-voting-app-redis 目录中，然后使用某个文本编辑器（例如 vi）打开清单文件：

```bash
vi azure-vote-all-in-one-redis.yaml
```

3.2 将 microsoft 替换为 ACR 登录服务器名称。 映像名称位于清单文件的第 60 行, 提供自己的 ACR 登录服务器名称，使清单文件如以下示例所示：

```yaml
containers:
  - name: azure-vote-front
    image: acrdemofb9caa25.azurecr.cn/azure-vote-front:v1
```

3.3 部署之前需要先把AKS 和 ACR 进行集成

```bash
az aks update -n <AKSCluster> -g <ResourceGroup> --attach-acr <acr-name>
```

3.4 部署应用程序

```bash
kubectl apply -f azure-vote-all-in-one-redis.

# 输出如下：
deployment.apps/azure-vote-back created
service/azure-vote-back created
deployment.apps/azure-vote-front created
service/azure-vote-front created
```

3.5 查看应用部署状态

```bash
kubectl get pod
kubectl get svc
```

3.6 若要查看应用程序的实际效果，请打开 Web 浏览器，以转到服务的外部 IP 地址：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926160437.png" width=600 />

### 4. 缩放应用程序

4.1 在前述教程中部署 Azure 投票前端和 Redis 实例时，创建了单个副本。 若要查看群集中 Pod 的数目和状态，请使用 kubectl get 命令，如下所示：

```bash
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
azure-vote-back-59d587dbb7-vlzxk    1/1     Running   0          6m3s
azure-vote-front-78dc4ff55b-bx4dx   1/1     Running   0          6m3s
```

4.2 若要手动更改 azure-vote-front 部署中的 Pod 数，请使用 kubectl scale 命令。 以下示例将前端 Pod 数增加到 3：

```bash
kubectl scale --replicas=3 deployment/azure-vote-front
```

4.3 再次运行 kubectl get pods，验证 AKS 是否成功创建其他 Pod。 大约一分钟后，即可在群集中使用 Pod：

```bash
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
azure-vote-back-59d587dbb7-vlzxk    1/1     Running   0          7m11s
azure-vote-front-78dc4ff55b-45pgr   1/1     Running   0          5s
azure-vote-front-78dc4ff55b-bx4dx   1/1     Running   0          7m11s
azure-vote-front-78dc4ff55b-wvskh   1/1     Running   0          5s
```

### 5. 更新应用程序

5.1 让我们更改示例应用程序，然后更新已部署到 AKS 群集的版本。 确保在克隆的 azure-voting-app-redis 目录中操作。 可在 azure-vote 目录中找到示例应用程序的源代码。 使用编辑器（例如 vi）打开 config_file.cfg 文件：

```bash
vi azure-vote/azure-vote/config_file.cfg
```

5.2 将 VOTE1VALUE 和 VOTE2VALUE 的值更改为不同的值，例如不同的颜色。 以下示例显示更新的值：

```python
# UI Configurations
TITLE = 'Azure Voting App V2'
VOTE1VALUE = 'Blue'
VOTE2VALUE = 'Purple'
SHOWHOST = 'false'
```

5.3 若要重新创建前端映像并测试更新的应用程序，请使用 docker-compose。 --build 参数用于指示 Docker Compose 重新创建应用程序映像：

```bash
docker-compose up --build -d
```

5.4 在本地打开<http://localhost:8080> 测试应用程序

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926170049.png" width=600 />

5.5 使用 docker tag 标记映像。 将\<acrLoginServer> 替换为 ACR 登录服务器名称或公共注册表主机名，并将映像版本更新为 :v2，如下所示：

```bash
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrLoginServer>/azure-vote-front:v2
```

5.6 现在，请使用 docker push 将映像上传到注册表。 将 \<acrLoginServer> 替换为 ACR 登录服务器名称。

```bash
docker push <acrLoginServer>/azure-vote-front:v2
```

5.7 若要更新应用程序，请使用 kubectl set 命令。 使用容器注册表的登录服务器或主机名更新 \<acrLoginServer>，并指定 v2 应用程序版本：

```bash
kubectl set image deployment azure-vote-front azure-vote-front=<acrLoginServer>/azure-vote-front:v2
```

5.8 若要监视部署，请使用 kubectl get pod 命令。 部署更新的应用程序时，Pod 终止运行并通过新容器映像重新创建。

```bash
kubectl get pods
```

5.9 现在，请打开 Web 浏览器并访问服务的 IP 地址：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210926170049.png" width=600 />
