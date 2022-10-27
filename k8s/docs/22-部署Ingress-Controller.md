# 部署Ingress Controller

## Ingress controller 资源

> ingress github 地址： <https://github.com/kubernetes/ingress-nginx>

## 部署中的注意事项

* 如果你时本地contos或者ubuntu上部署的群集，请执行Bare-metal yaml文件

* yaml文件中定义的image，如果在国内可能被墙掉，这里可以替换docker hub上的镜像链接，可参考如下：
<https://hub.docker.com/r/siriuszg/nginx-ingress-controller>

* 裸金属部署模式需要注意是使用node port 模式还是host模式
如果是host模式需要在yaml文件deployment模块添加如下信息

    ```yaml
    template:
    spec:
        hostNetwork: true
    ```

    参考链接：<https://kubernetes.github.io/ingress-nginx/deploy/baremetal/>

## 测试Ingress 功能

### 编写测试yaml文件

```yaml
#deploy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-demo
spec:
  selector:
    matchLabels:
      app: tomcat-demo
  replicas: 1
  template:
    metadata:
      labels:
        app: tomcat-demo
    spec:
      containers:
      - name: tomcat-demo
        image: registry.cn-hangzhou.aliyuncs.com/liuyi01/tomcat:8.0.51-alpine
        ports:
        - containerPort: 8080
---
#service
apiVersion: v1
kind: Service
metadata:
  name: tomcat-demo
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat-demo

---
#ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tomcat-demo
spec:
  rules:
  - host: tomcat.contoso.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-demo
          servicePort: 80
```
