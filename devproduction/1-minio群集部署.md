# 部署Minio高可用群集

## 环境准备

### 虚拟机及操作系统基础环境

#### 服务器信息

|主机名|IP地址|操作系统|挂载磁盘|
|:--|:--|:--|:--|
|minio01|192.168.100.181|RedHat8.6|/dev/sdb1|
|minio02|192.168.100.182|RedHat8.6|/dev/sdb1|
|minio03|192.168.100.183|RedHat8.6|/dev/sdb1|
|minio04|192.168.100.184|RedHat8.6|/dev/sdb1|

#### 修改计算机名称

```bash
# 分别在每个节点上运行
hostnamectl set-hostname minio01
hostnamectl set-hostname minio02
hostnamectl set-hostname minio03
hostnamectl set-hostname minio04
# 运行完成以上命令后重启节点生效
```

修改hosts文件，实现内网互通

```bash
cat >> /etc/hosts << EOF
192.168.100.181 minio01
192.168.100.182 minio02
192.168.100.183 minio03
192.168.100.184 minio04
EOF
```

#### 初始化并挂载磁盘

```bash
# 查看磁盘情况
fdisk -l
Disk /dev/sda: 127 GiB, 136365211648 bytes, 266338304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x5997ed9a

Device     Boot   Start       End   Sectors  Size Id Type
/dev/sda1  *       2048   2099199   2097152    1G 83 Linux
/dev/sda2       2099200 266338303 264239104  126G 8e Linux LVM

Disk /dev/sdb: 200 GiB, 214748364800 bytes, 419430400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

由于MBR分区格式有一定限制我们这里使用GPT分区格式

```bash
# 下载工具gdisk
yum install -y gdisk
# 运行gdisk进行分区
gdisk /dev/sdb
```

输入？获取命令
![20221016191135](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20221016191135.png)

输入n 创建分区（一直保持默认即可）

![20221016191427](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20221016191427.png)

输入 w 保存分区

![20221016191928](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20221016191928.png)

最后使用命令lsblk查看块设备信息，此时已经可以看到刚新建的分区了

![20221016192049](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20221016192049.png)

格式化分区为xfs

```bash
mkfs -t xfs /dev/sdb1
```

创建minio数据目录，然后挂载上面的磁盘

```bash
# 创建minio目录
mkdir -p /data/minio-data
# 把上面操作的磁盘挂载到此目录
mount /dev/sdb1 /data/minio-data
# 编辑/etc/fstab文件，永久挂载此目录
cat >> /etc/fstab << EOF
/dev/sdb1 /data/minio-data xfs defaults 0 0
EOF
```

#### 系统配置

关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld
```

禁用 selinux

```bash
Edit the /etc/selinux/config file and change SELINUX=enforcing to SELINUX=disabled .
Reboot the system. $ sudo reboot.
```

修改系统最大文件数

```bash
# 查看最大连接数
ulimit -n 
ulimit -a
echo "*  soft  nofile  65535" >> /etc/security/limits.conf
echo "*  hard  nofile  65535" >> /etc/security/limits.conf
sysctl -p
reboot
```

创建minio启动脚本和配置文件目录

```bash
mkdir -p /data/minio/run && mkdir -p /etc/minio
```

下载minio到/data/minio/run目录下

```bash
cd /data/minio/run && wget https://dl.min.io/server/minio/release/linux-amd64/minio
```

## 部署minio

### 编写集群启动脚本

> 所有节点配置文件相同

启动脚本/data/minio/run/run.sh

```bash
#!/bin/bash

#其中，“MINIO_ACCESS_KEY”为用户名，“MINIO_SECRET_KEY”为密码，密码不能设置过于简单，不然minio会启动失败
#export MINIO_ACCESS_KEY=minio
#export MINIO_SECRET_KEY=miniostorage


export MINIO_ROOT_USER=minio
export MINIO_ROOT_PASSWORD=miniostorage

# 4 个节点，每个节点各有两个目录
/data/minio/run/minio server  --config-dir /etc/minio --address ":9000" --console-address ":9001" \
http://192.168.100.181/data/minio-data/data1 http://192.168.100.181/data/minio-data/data2 \
http://192.168.100.182/data/minio_data/data1 http://192.168.100.182/data/minio-data/data2 \
http://192.168.100.183/data/minio-data/data1 http://192.168.100.183/data/minio-data/data2 \
http://192.168.100.184/data/minio-data/data1 http://192.168.100.184/data/minio-data/data2
```

### 创建systemd配置文件minio.service

```bash
cat > /usr/lib/systemd/system/minio.service <<EOF
[Unit]
Description=Minio service
Documentation=https://docs.minio.io/
 
[Service]
WorkingDirectory=/data/minio/run/
ExecStart=/data/minio/run/run.sh
 
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
EOF
```

### 启动

修改权限

```bash
chmod +x /usr/lib/systemd/system/minio.service && chmod +x /data/minio/run/minio && chmod +x /data/minio/run/run.sh
```

依次启动每个服务器的minio

```bash
systemctl daemon-reload
systemctl enable minio && systemctl start minio
systemctl status minio
```

>使用 journalctl -xe 查看日志

## 使用nginx做代理

### 资源初始化配置

> 服务器IP地址：192.168.100.185

```bash
# 配置计算机名称
hostnamectl set-hostname nginx-server
# 重启生效
reboot
# 关闭防火墙
systemctl stop firewalld
systemctl status firewalld
systemctl disable firewalld
```

### 安装nginx

```bash
# 安装编译工具和库文件
yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel
# 安装 PCRE让 Nginx 支持 Rewrite 功能
cd /usr/local/src/
wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz
#解压安装包
tar -zxvf pcre-8.35.tar.gz
# 进入安装目录
cd pcre-8.35
# 编译安装
/configure
make && make install
# 安装 nginx
cd /usr/local/src/
wget http://nginx.org/download/nginx-1.20.2.tar.gz
# 解压安装包
tar -zxvf nginx-1.20.2.tar.gz
# 进入安装包目录
cd nginx-1.20.2
# 编译安装
./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.35
make
make install
```

### 修改配置文件：/usr/local/webserver/nginx/conf/nginx.conf

```bash
upstream minio{
        server 192.168.159.137:9001;
        server 192.168.159.138:9001;
        server 192.168.159.139:9001;
        server 192.168.159.140:9001;
}
server {
        listen 80;
        server_name localhost;
        location / {
                proxy_pass http://minio;
                proxy_set_header Host $http_host;
               #client_max_body_size 1000m;
        }
}
```

### 启动nginx

```bash
cd /usr/local/webserver/nginx
./sbin/nginx  -c ./conf/nginx.conf #指定配置文件，启动方式
./sbin/nginx  -c ./conf/nginx.conf -s reload #修改配置文件，重新加载
```
