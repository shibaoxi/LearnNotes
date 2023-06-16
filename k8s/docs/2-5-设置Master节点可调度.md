# 设置Master节点可调度与不可调度

## 语法

```bash
kubectl taint node [node] key=value[effect]
[effect] 可取值: [ NoSchedule | PreferNoSchedule | NoExecute ]
# NoSchedule: 一定不能被调度
# PreferNoSchedule: 尽量不要调度
# NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod
```

## 查看Master节点状态

```bash
kubectl describe node k8smaster01

......
CreationTimestamp:  Thu, 24 Nov 2022 15:57:19 +0800
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
                    node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
......
```

## 设置Master节点可调度

```bash
kubectl taint node k8smaster01 node-role.kubernetes.io/master-
kubectl taint node k8smaster01 node-role.kubernetes.io/control-plane-
```

## 设置master节点不可调度

```bash
kubectl taint node k8smaster01 node-role.kubernetes.io/master="":NoSchedule
kubectl taint node k8smaster01 node-role.kubernetes.io/control-plane="":NoSchedule
```
