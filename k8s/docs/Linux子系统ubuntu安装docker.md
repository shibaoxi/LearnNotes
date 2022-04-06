# 在windows 11 Linux 子系统中（Ubuntu）安装docker

## 在Linux 子系统中（Ubuntu）安装docker

```shell
# 1.由于apt官方库里的docker版本可能比较旧，所以先卸载可能存在的旧版本
sudo apt-get remove docker docker-engine docker-ce docker.io
# 2.更新apt包索引
sudo apt-get update
# 3.安装以下包以使apt可以通过HTTPS使用存储库（repository）
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
# 4.添加Docker官方的GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# 5.安装stable存储库
sudo add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
# 6.查看Docker-ce的版本
apt-cache madison docker-ce
# 7.安装docker-ce
# (选择上面与系统版本匹配的版本，我的系统是16.06，上面没有匹配的版本，顾选择低一个版本)
# 查看Ubuntu版本 cat /etc/issue
sudo apt-get install docker-ce=5:20.10.13~3-0~ubuntu-focal
# 8.启动docker
sudo service docker start
# 9.查看docker 运行状态 （running 标是正在运行）
sudo service docker status
# 10.  查看docker版本及信息(出现下图结果表示安装完成)
sudo docker version
```
