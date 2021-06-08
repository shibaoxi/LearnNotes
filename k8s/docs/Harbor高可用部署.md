# Harbor 高可用部署

## 介绍

Harbor是VMware公司开源的一个企业级Docker Registry项目，项目地址：<https://github.com/goharbor/harbor>

Harbor作为一个企业级私有Registry服务器，提供了更好的性能和安全，提升了用户使用Registry构建和运行环境传输镜像的效率。虽然Harbor和Registry都是私有镜像仓库的选择，但是Harbor的企业级特性更强，因此也是更多企业级用户的选择。

Harbor实现了基于角色的访问控制机制，并通过项目来对镜像进行组织和访问权限的控制，也常常和K8S中的namespace结合使用。此外，Harbor还提供了图形化的管理界面，我们可以通过浏览器来浏览，检索当前Docker镜像仓库，管理项目和命名空间。

## 架构

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/architecture.png" width=600 />

## 高可用架构

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210426145848.png" width=600 />

## 安装

### 1. 下载离线安装包

```shell
#下载离线安装包
wget https://github.com/goharbor/harbor/releases/download/v2.2.1/harbor-offline-installer-v2.2.1.tgz
```

### 2. 解压安装包

```shell
#解压压缩包
tar zxvf harbor-offline-installer-v2.2.1.tgz
```

### 3. 修改配置文件

```shell
# 进入解压后的文件夹
cd harbor
# 修改harbor.yml.tmpl文件
cp harbor.yml.tmpl harbor.yml
vi harbor.yml
```

修改如下配置

```shell
hostname: 192.168.100.194 #修改成自己的IP地址
#修改端口
port 8080
# 修改密码
harbor_admin_password: Harbor12345
# 数据存放目录，建议选择空间足够的路径
data_volume: /data
```

### 4. 安装docker compose

```shell
# 下载
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# 赋予执行权限
sudo chmod +x /usr/local/bin/docker-compose 
# 建立软连接
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
# 查看版本
docker-compose --version
```

### 5. 安装Harbor

```shell
# 执行install.sh
sudo ./install.sh
```

## 配置高可用

### 1. 配置nginx

> 这里我们选择一台管理节点，使用容器nginx

```shell
# 下载nginx镜像
docker pull nginx
# 创建nginx配置文件夹
mkdir /root/nginx
# 创建配置文件
vi nginx.conf
```

编辑配置文件如下：

```shell
user nginx;
worker_processes 1;

error_log /var/log/nginx/error.log warn;

pid /var/run/nginx.pid;

events {

    worker_connections 1024;
}

stream {
        upstream hub{
          server 192.168.100.194:8080;
          server 192.168.100.195:8080 backup;
        }
        server {
          listen 80;
          proxy_pass hub;
          proxy_timeout 300s;
          proxy_connect_timeout 5s;
        }
}
```

编辑启动docker脚本

```shell
vi restartharbornginx.sh 
 
```

```shell
#!/bin/bash

echo "stop the old docker harbornginx"
docker stop harbornginx
echo "delete the old docker harbornginx"
docker rm harbornginx

docker run -idt --net=host --name harbornginx -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf nginx
```

执行shell脚本

```shell
sh restartharbornginx.sh
```

### 2. 配置Harbor复制

#### 2.1 新建项目

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210507161828.png" width=600 />

#### 2.2 新建用户

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210507162320.png" width=600 />

#### 2.3 把用户添加到项目中

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210507162500.png" width=600 />

#### 2.4 上传第一个镜像

```shell
# 查看本地镜像
docker images
# 使用刚才新建的账户登录
docker login hub.contoso.com
# 打tag
docker tag nginx:latest hub.contoso.com/demorepo/nginx:latest
# 上传
docker push hub.contoso.com/demorepo/nginx:latest
# 注意，这次环境我们没有使用https，这里会报https错误，我们可以修改daemon.json 文件来解决这个问题
vi /etc/docker/daemon.json
# 添加如下内容
"insecure-registries": ["hub.contoso.com"]
# 重启docker服务
systemctl restart docker
```

#### 2.5 新建复制目标

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210507170101.png" width=600 />

#### 2.6 新建复制策略

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210507170409.png" width=600 />

#### 2.7 另一台也执行相同动作
