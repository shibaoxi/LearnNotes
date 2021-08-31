# Helm简介与安装

## Helm介绍

### 简介

很多人都使用过Ubuntu下的ap-get或者CentOS下的yum, 这两者都是Linux系统下的包管理工具。采用apt-get/yum,应用开发者可以管理应用包之间的依赖关系，发布应用；用户则可以以简单的方式查找、安装、升级、卸载应用程序。

我们可以将Helm看作Kubernetes下的apt-get/yum。Helm是Deis (<https://deis.com/>) 开发的一个用于kubernetes的包管理器。每个包称为一个Chart，一个Chart是一个目录（一般情况下会将目录进行打包压缩，形成name-version.tgz格式的单一文件，方便传输和存储）。

对于应用发布者而言，可以通过Helm打包应用，管理应用依赖关系，管理应用版本并发布应用到软件仓库。

对于使用者而言，使用Helm后不用需要了解Kubernetes的Yaml语法并编写应用部署文件，可以通过Helm下载并在kubernetes上安装需要的应用。

除此以外，Helm还提供了kubernetes上的软件部署，删除，升级，回滚应用的强大功能。

### 用途

做为 Kubernetes 的一个包管理工具，Helm具有如下功能：

- 创建新的 chart
- chart 打包成 tgz 格式
- 上传 chart 到 chart 仓库或从仓库中下载 chart
- 在Kubernetes集群中安装或卸载 chart
- 管理用Helm安装的 chart 的发布周期

### 组件与架构

<img src="https://i.loli.net/2021/08/30/tFqwk1IzyEd8avS.png" width=600 />

#### 组件介绍

- **Helm客户端**

    Helm是一个命令行下的客户端工具。主要用于Kubernetes应用程序Chart的创建、打包、发布以及创建和管理本地和远程的Chart仓库。
- **~~Tiller~~**

    ~~Tiller是Helm的服务端，部署在Kubernetes群集中。Tiller用于接收Helm的请求，并根据Chart生成Kubernetes的部署文件（Helm 称为 Release），然后提交给Kubernetes创建应用。Tiller还提供了Release的升级、删除、回滚等一系列功能。~~
    > Helm 3 中移除了Tiller，版本相关的数据直接存储在Kubernetes中。
- **Chart**
    Helm的软件包，采用TAR格式。类似于APT的DEV包或者YUM的RPM包，其包含了一组定义Kubernetes资源相关的YAML文件。
    > V3 中支持将 Chart 推送至 Docker 镜像仓库中
- **Repoistory**
    Helm的软件仓库，Repository本质上是一个web服务器，该服务器保存了一系列的Chart软件包以供用户下载，并且提供了一个该Repository的Chart包的清单文件以供查询。Helm可以同时管理多个不同的Repository。
- **Release**
    使用 ```helm install``` 命令在Kubernetes集群部署的Chart称为Release。
    > 需要注意的是：Helm 中提到的 Release 和我们通常概念中的版本有所不同，这里的 Release 可以理解为 Helm 使用 Chart 包部署的一个应用实例。
    Release名称可以在不同命名空间重用
    helm install 不再默认生成一个 Release 的名称，除非指定了 --generate-name。

## helm 安装部署

### Helm 客户端安装

Helm 的安装方式很多，这里采用二进制的方式安装。

```bash
# 从官网下载最新版的二进制安装包到本地: https://github.com/helm/helm/releases/
curl -O https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz
#解压压缩包
tar -zxvf helm-v3.6.3-linux-amd64.tar.gz
#把helm指令放到bin目录下
mv linux-amd64/helm /usr/local/bin/helm
#验证
helm version
```

### 配置国内的Chart仓库

- ~~微软仓库（ <http://mirror.azure.cn/kubernetes/charts/>）~~
- 阿里云仓库（ <https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts> ）推荐
- 官方仓库（ <https://hub.kubeapps.com/charts/incubator>）官方chart仓库，国内有点不好使。

添加存储库

```bash
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo update
```

查看配置的存储库

```bash
helm repo list
```
