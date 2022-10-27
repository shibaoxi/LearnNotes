# Pod健康检查机制

## 健康检查

### 健康检查概述

 应用在运行过程中难免会出现错误，如程序异常，软件异常，硬件故障，网络故障等，kubernetes提供Health Check健康检查机制，当发现应用异常时会自动重启容器，将应用从service服务中剔除，保障应用的高可用性。k8s定义了三种探针Probe：

* **readiness probes** 准备就绪检查，通过readiness是否准备接受流量，准备完毕加入到endpoint，否则剔除
* **liveness probes** 在线检查机制，检查应用是否可用，如死锁，无法响应，异常时会自动重启容器
* **startup probes** 启动检查机制，应用一些启动缓慢的业务，避免业务长时间启动而被前面的探针kill掉

每种探测机制支持三种健康检查方法，分别是命令行exec，httpGet和tcpSocket，其中exec通用性最强，适用与大部分场景，tcpSocket适用于TCP业务，httpGet适用于web业务。

* **exec**  提供命令或shell的检测，在容器中执行命令检查，返回码为0健康，非0异常
* **httpGet** http协议探测，在容器中发送http请求，根据http返回码判断业务健康情况
* **tcpSocket** tcp协议探测，向容器发送tcp建立连接，能建立则说明正常

每种探测方法能支持几个相同的检查参数，用于设置控制检查时间：

* **initialDelaySeconds** 初始第一次探测间隔，用于应用启动的时间，防止应用还没启动而健康检查失败
* **periodSeconds** 检查间隔，多久执行probe检查，默认为10s
* **timeoutSeconds** 检查超时时长，探测应用timeout后为失败
* **successThreshold** 成功探测阈值，表示探测多少次为健康正常，默认探测1次

### exec命令行健康检查

定义一个容器，启动时创建一个文件，健康检查时ls -l /tmp/liveness-probe.log返回码为0，健康检查正常，10s后将其删除，返回码为非0，健康检查异常

```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: exec-liveness-probe
    spec:
      containers:
      - name: exec-liveness-probe
        image: centos:latest
        imagePullPolicy: IfNotPresent
        args: #容器启动命令，生命周期为30s
        - /bin/sh
        - -c
        - touch /tmp/liveness-probe.log && sleep 10 && rm -f /tmp/liveness-probe.log && sleep 20
        livenessProbe:
          exec: #健康检查机制，通过ls -l /tmp/liveness-probe.log 返回码判断容器的健康状态
            command:
            - ls
            - -l
            - /tmp/liveness-probe.log
          initialDelaySeconds: 1
          periodSeconds: 5
          timeoutSeconds: 1
```

查看容器的event日志，容器启动后，10s以内容器状态正常，11s开始执行liveness健康检查，检查异常，触发容器重启

```bash
kubectl describe pods exec-liveness-probe | tail
Events:
  Type     Reason     Age               From          Message
  ----     ------     ----              ----          -------
  Normal   Scheduled  <unknown>                       Successfully assigned default/exec-liveness-probe to w01
  Normal   Pulling    2m43s             kubelet, w01  Pulling image "centos:latest"
  Normal   Pulled     63s               kubelet, w01  Successfully pulled image "centos:latest" in 1m40.0763836s
  Warning  Unhealthy  9s (x6 over 49s)  kubelet, w01  Liveness probe failed: ls: cannot access '/tmp/liveness-probe.log': No such file or directory
  Normal   Killing    9s (x2 over 39s)  kubelet, w01  Container exec-liveness-probe failed liveness probe, will be restarted
  Normal   Created    2s (x3 over 62s)  kubelet, w01  Created container exec-liveness-probe
  Normal   Started    2s (x3 over 62s)  kubelet, w01  Started container exec-liveness-probe
  Normal   Pulled     2s (x2 over 32s)  kubelet, w01  Container image "centos:latest" already present on machine

```

### httpGet健康检查

httpGet probe主要主要用于web场景，通过向容器发送http请求，根据返回码判断容器的健康状态，返回码小于4xx即表示健康，如下定义一个nginx应用，通过探测<http://[container>]:port/index.html的方式判断健康状态

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-httpget-probe
spec:
  containers:
  - name: nginx-httpget-probe
    image: nginx:latest
    ports:
    - name: http-80-port
      protocol: TCP
      containerPort: 80
    livenessProbe:
      httpGet:
        port: 80
        scheme: HTTP
        path: /index.html
      initialDelaySeconds: 3
      periodSeconds: 10
      timeoutSeconds: 3
```

生成pod并查看健康状态

```bash
kubectl apply -f nginx-httpget-probe.yaml
kubectl get pod -o wide
#显示信息如下
NAME                           READY   STATUS    RESTARTS   AGE    IP              NODE   NOMINATED NODE   READINESS GATES

nginx-httpget-probe            1/1     Running   0          102s   172.18.211.86   w01    <none>           <none>

```

模拟故障，将pod中的path文件所属文件删除，此时发送http请求时会健康检查异常，会触发容器自动重启

```bash
kubectl exec -it nginx-httpget-probe -- bash

root@nginx-httpget-probe:/# ls -l /usr/share/nginx/html/index.html 
-rw-r--r-- 1 root root 612 Jul  6 14:59 /usr/share/nginx/html/index.html

root@nginx-httpget-probe:/# rm -f /usr/share/nginx/html/index.html
```

查看pod的详情，观察容器重启的情况，通过Liveness 检查容器出现404错误，触发重启。

```bash
[root@cli yaml]# kubectl describe pod nginx-httpget-probe | tail
  Type     Reason     Age                From          Message
  ----     ------     ----               ----          -------
  Normal   Scheduled  <unknown>                        Successfully assigned default/nginx-httpget-probe to w01
  Normal   Pulled     11m                kubelet, w01  Successfully pulled image "nginx:latest" in 1m16.6629643s
  Normal   Pulling    71s (x2 over 12m)  kubelet, w01  Pulling image "nginx:latest"
  Warning  Unhealthy  71s (x3 over 91s)  kubelet, w01  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    71s                kubelet, w01  Container nginx-httpget-probe failed liveness probe, will be restarted
  Normal   Created    67s (x2 over 11m)  kubelet, w01  Created container nginx-httpget-probe
  Normal   Started    67s (x2 over 11m)  kubelet, w01  Started container nginx-httpget-probe
  Normal   Pulled     67s                kubelet, w01  Successfully pulled image "nginx:latest" in 3.856535s

```

### tcpSocket健康检查

tcpsocket健康检查适用于TCP业务，通过向指定容器建立一个tcp连接，可以建立连接则健康检查正常，否则健康检查异常，依旧以nignx为例使用tcp健康检查机制，探测80端口的连通性

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-tcp-liveness-probe
spec:
  containers:
  - name: nginx-tcp-liveness-probe
    image: nginx:latest
    ports:
    - name: http-80-port
      protocol: TCP
      containerPort: 80
    livenessProbe: #健康检查为tcpSocket，探测 tcp 80 端口
      tcpSocket:
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 10
      timeoutSeconds: 3
```

创建并查看容器

```bash
[root@cli yaml]# kubectl apply -f nginx-tcp-liveness.yaml 
pod/nginx-tcp-liveness-probe created
[root@cli yaml]# kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
nginx-tcp-liveness-probe       1/1     Running   0          24s

```

模拟故障，获取pod所属节点，登录到pod中，安装查看进程工具htop

![img](https://i.loli.net/2021/07/21/54Nj2WFckwo3vmH.png)

容器进程通常为1，kill掉进程观察容器状态，观察重启次数

```bash
root@nginx-tcp-liveness-probe:/# kill 1
root@nginx-tcp-liveness-probe:/# command terminated with exit code 137

[root@cli yaml]# kubectl get pod 
NAME                           READY   STATUS    RESTARTS   AGE
nginx-tcp-liveness-probe       1/1     Running   1          14m

```

### readiness 健康就绪

就绪检查用于应用接入到service的场景，用于判断应用是否已经就绪完毕，即是否可以接受外部转发的流量，健康检查正常则将pod加入到service的endpoints中，健康检查异常则从service的endpoints中删除，避免影响业务的访问。

创建一个pod，使用httpGet的健康检查机制，定义readiness就绪检查探针检查路径/test.html

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-tcp-liveness-readiness-probe
  labels:
    app: nginx-probe
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-probe
  template:
    metadata:
      labels:
        app: nginx-probe
    spec:
      containers:
      - name: nginx-tcp-liveness-readiness-probe
        image: nginx:latest
        ports:
        - name: http-80-port
          protocol: TCP
          containerPort: 80
        livenessProbe: #存活检查针
          httpGet:
            port: 80
            path: /index.html
            scheme: HTTP
          initialDelaySeconds: 3
          periodSeconds: 10
          timeoutSeconds: 3
        readinessProbe: #就绪检查探针
          httpGet:
            port: 80
            path: /test.html
            scheme: HTTP
          initialDelaySeconds: 3
          periodSeconds: 10
          timeoutSeconds: 3
---
#service
apiVersion: v1
kind: Service
metadata:
  name: nginx-tcp-liveness-service
  labels:
    app: nginx-probe
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-probe
```

检查pod状态，以及service endpoint状态

```bash
[root@cli yaml]# kubectl get pod
NAME                                                  READY   STATUS    RESTARTS   AGE
nginx-tcp-liveness-readiness-probe-744cc579d4-64tx4   0/1     Running   0          4m45s

[root@cli yaml]# kubectl describe pod nginx-tcp-liveness-readiness-probe-744cc579d4-64tx4 | tail
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                    From          Message
  ----     ------     ----                   ----          -------
  Normal   Scheduled  <unknown>                            Successfully assigned default/nginx-tcp-liveness-readiness-probe-744cc579d4-64tx4 to w01
  Normal   Pulling    7m                     kubelet, w01  Pulling image "nginx:latest"
  Normal   Pulled     6m56s                  kubelet, w01  Successfully pulled image "nginx:latest" in 3.8209648s
  Normal   Created    6m56s                  kubelet, w01  Created container nginx-tcp-liveness-readiness-probe
  Normal   Started    6m56s                  kubelet, w01  Started container nginx-tcp-liveness-readiness-probe
  Warning  Unhealthy  118s (x30 over 6m48s)  kubelet, w01  Readiness probe failed: HTTP probe failed with statuscode: 404

[root@cli yaml]# kubectl describe services nginx-tcp-liveness-service 
Name:              nginx-tcp-liveness-service
Namespace:         default
Labels:            app=nginx-probe
Annotations:       Selector:  app=nginx-probe
Type:              ClusterIP
IP:                172.19.86.228
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         
Session Affinity:  None
Events:            <none>

#查看services的endpoints，发现此时endpoints为空,因为readiness就绪检查异常，kubelet认为此时pod并未就绪，因此并未将其加入到endpoints中。
```

进入到pod中手动创建网站文件，使readiness健康检查正常

```bash
[root@cli yaml]# kubectl exec -it nginx-tcp-liveness-readiness-probe-744cc579d4-64tx4 -- bash
root@nginx-tcp-liveness-readiness-probe-744cc579d4-64tx4:/# echo "hello word" > /usr/share/nginx/html/test.html

```

测试readiness健康检查正常，kubelet检测到pod就绪会将其加入到endpoint中

```bash
[root@cli yaml]# kubectl describe services nginx-tcp-liveness-service 
Name:              nginx-tcp-liveness-service
Namespace:         default
Labels:            app=nginx-probe
Annotations:       Selector:  app=nginx-probe
Type:              ClusterIP
IP:                172.19.86.228
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         172.18.211.87:80
Session Affinity:  None
Events:            <none>

```
