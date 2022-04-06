# 手动更新Kubernetes证书

> kubernetes证书过期错误提示：Unable to connect to the server: x509: certificate has expired or is not yet valid

## 查看证书状态

查看证书文件状态

```bash
# 登录到master节点，然后进入/etc/kubernetes/pki目录
cd /etc/kubernetes/pki
# 运行如下命令查看证书文件状态
openssl x509 -in apiserver.crt -noout -text | grep 'Not'
```

输出如下状态：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220324111117.png" width=600 />

可以看到证书文件已经过期

另外我们也可以通过 ```check-expiration``` 命令来检查证书是否过期

```bash
kubeadm certs check-expiration
```

输出如下信息：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220324114524.png" width=600 />

可以看到证书的到期时间和剩余时间

>**最佳的升级证书方法是，经常更新群集，在更新群集时会自动更新证书**

## 手动更新证书

登录到master节点

```bash
kubeadm certs renew all
```

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220324115027.png" width=600 />

## 将admin.conf文件替换掉config文件

```bash
# 备份config文件
mv ~/.kube/config ~/.kube/config.old
# copy admin.conf文件到.kube目录下
cp -i admin.conf ~/.kube/config
```

## 重启相关pod

按照提示需要重启 kube-apiserver, kube-controller-manager, kube-scheduler 和 etcd

```bash
ns=kube-system
# 查看pod
kubectl get pod -n $ns
# 删除pod，对应etcd, kube-apiserver,kubect-controller-manager,kube-schedule
kubectl delete -n $ns pod  etcd-m02
kubectl delete -n $ns pod kube-apiserver-m02
kubectl delete -n $ns pod kube-controller-manager-m02
kubectl delete -n $ns pod kube-scheduler-m02
```

## 再次检查证书状态

```bash
kubeadm certs check-expiration
```

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220324120634.png" width=600 />

> 如果有其它主节点，在其他节点执行同样动作
