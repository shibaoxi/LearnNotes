# NameSpace 总结

## NameSpace 简介

Kubernetes集群可以同时管理大量互不相关的工作负载，而组织通常会选择将不同团队创建的项目部署到共享集群上。随着数量的增加，部署对象常常很快就会变得难以管理，拖慢操作响应速度，并且会增加危险错误出现的概率。

Kubernetes使用命名空间的概念帮助解决集群中在管理对象时的复杂性问题。命名空间允许将对象分组到一起，便于将它们作为一个单元进行筛选和控制。无论是应用自定义的访问控制策略，还是为了测试环境而分离所有组件，命名空间都是一个按照组来处理对象、强大且灵活的概念。

<img src="https://i.loli.net/2021/07/13/h7yVnDEINjBxFLA.png" width=600 />

## 初始NameSpace介绍

|ns名称|作用说明|
|--|--|
|default|缺省的namespace，对于没有指定namespace的对象缺省会在此处进行管理|
|kube-system|Kubernetes系统所创建的对象在此namespace中进行管理|
|kube-public|此namespace对所有用户均为可读（包括通过认证的用户）。此namespace的存在实施一种约定，并非为必选项。|

## 常规操作

创建命名空间

```shell

```
