# Deployment 声明式升级应用

> ## 概念
>
> 一个Deployment控制器为pods和ReplicaSets提供声明式更新能力

## 以下是Deployments的常用场景

- 创建 Deployment 以将 ReplicaSet 上线。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
- 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新 Deployment 的修订版本。
- 扩大 Deployment 规模以承担更多负载。
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
- 使用 Deployment 状态 来判定上线过程是否出现停滞。
- 清理较旧的不再需要的 ReplicaSet

---

## 准备Deployment

### 准备镜像

1. 创建app.js

    ```javascript
    const http = require('http');
    const os = require('os');

    console.log('demo server starting ....');

    var handler = function(request, response){
        console.log("Received request from" + request.connection.remoteAddress);
        response.writeHead(200);
        response.end("This is v1 running in pod " + os.hostname() + "\n");
    };

    var www = http.createServer(handler);
    www.listen(8080);
    ```
  
2. 创建Dockerfile

    ```dockerfile
    FROM node:7
    ADD app.js /app.js
    ENTRYPOINT ["node", "app.js"]
    ```

3. 推送镜像

    ```bash
    docker build -t davidshi/demoapp:v1 .
    docker push davidshi/demoapp:v1
    ```

### 创建Deployment 对象

- 创建一个webapp-deployment-v1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-recreate
  labels:
    app: webapp-recreate
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-recreate
  template:
    metadata:
      name: webapp-recreate
      labels:
        app: webapp-recreate
    spec:
      containers:
      - image: davidshi/webapp202107:v1
        name: webapp-recreate
        ports:
        - containerPort: 8080
        livenessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 10
          timeoutSeconds: 3
---
#service
apiVersion: v1
kind: Service
metadata:
  name: webapp-recreate-service
  namespace: dev
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: webapp-recreate
---
#ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-recreate-ingress
  namespace: dev
spec:
  defaultBackend:
    service:
      name: webapp-recreate-service
      port:
        number: 80
        
```

- 应用yaml文件创建资源

  ``` kubectl apply -f demo-deployment-v1.yaml --record ```

    > 注意： 确保在创建时使用了 --record选项，这个选项会记录历史版本号，在之后的操作中非常有用

- 运行``` kubectl get deployments ```检查Deployment是否已经创建

  ```bash
  NAME             READY   UP-TO-DATE   AVAILABLE   AGE
  demodeployment   3/3     3            3           16h
  ```

- 运行``` kubectl rollout status deployment demodeployment ```检查deployment上线状态，输入类似于：

  ```bash
  deployment "demodeployment" successfully rolled out
  ```

---

### 升级Deployment

Deplyment的升级策略:

- Recreate策略：在删除旧的pod之后才开始创建新的pod。如果你的应用程序不支持多个版本同时对外提供服务，需要在启动新版本之前完全停用旧版本，那么需要使用这种策略。（会造成应用短暂不可用）
- RollingUpdate策略: 会渐进的删除旧的pod，与此同时创建新的pod，使应用程序在整个升级过程中都处于可用状态。如果应用支持多个版本同时对外提供服务，则推荐使用这个策略。(Deployment默认使用策略)
- 蓝绿部署： 利用service selector 选择不同版本的服务
- 金丝雀部署：通过ingress不停的访问不同的服务

> 注意：仅当Deployment Pod模板（.spec.template）发生改变时，例如模板的标签或者容器镜像被更新，才会触发Deployment上线。其他更新（如对Deployment执行扩缩容的操作）不会触发上线动作。

### 演示更新的操作

1. 更新镜像版本，把镜像的应用编辑更新，然后推送新的镜像版本

    - 更新app.js脚本

        ```javascript
        const http = require('http');
        const os = require('os');

        console.log('demo server starting ....');

        var handler = function(request, response){
            console.log("Received request from" + request.connection.remoteAddress);
            response.writeHead(200);
            response.end("This is v2 running in pod " + os.hostname() + "\n");
        };

        var www = http.createServer(handler);
        www.listen(8080);
        ```

    - 推送新的版本镜像

        ```bash
        docker build -t davidshi/demoapp:v2
        docker push davidshi/demoapp:v2
        ```

2. 减慢滚动升级速度
    > 我们将执行如下命令来更改Deployment 上的minReadySeconds属性来实现

    ```bash
    kubectl patch deployments demodeployment -p '{"spec":{"minReadySeconds": 10}}'
    ```

3. 触发滚动升级

    如果想要跟踪更新过程中应用的运行状况，需要现在另外一个终端中再次运行curl循环，以查看请求返回的情况

    ```bash
    while true; do curl http://40.73.8.50; done
    ```

    修改pod镜像版本以触发滚动升级

      ```bash
      kubectl set image deployment demodeployment nodejs=davidshi/demoapp:v2
      ```

---

### 修改Deployment或其他资源的不同方式

|方法|作用|
|----|----|
|<div style="width: 100pt">  kubectl edit </div>|使用默认编辑器打开资源配置。修改保存并退出编辑器，资源对象会被更新。例如：``` kubectl edit deployment demodeployment ```
|kubectl patch |修改单个资源属性。例如：``` kubectl patch deployment demodeployment -p '{"spec":{"template": {"spec": {"containers": [{"name": "nodejs", "image": "davidshi/demoapp:v2"}]}}}}' ```|
|kubectl apply|通过一个完整的yaml或者json文件，应用其中新的值来修改对象。如果yaml/json中指定的对象不存在，则被创建。该文件需要包含资源的完整定义(不能像kubectl patch那样只包含想要更新的字段)。例如: ``` kubectl apply -f demo-deployment-v2.yaml ```|
|kubectl replace|将原有对象替换为yaml/json文件中定义的新的对象，与apply命令相反，运行这个命令前要求对象必须存在，否则会报错。例如: ``` kubectl replace -f demo-deployment-v2.yaml ```|
|kubectl set image|修改Pod、RC、Deployment、DemonSet、Job或者ReplicaSet内的镜像。例如: ``` kubectl set image deployment demodeployment nodejs=davidshi/demoapp:v2 ```|

---

### 回滚 Deployment

1. 创建一个v3版本的应用程序

    ```javascript
    const http = require('http');
    const os = require('os');
    var requestCount = 0;
    console.log('demo server starting ....');

    var handler = function(request, response){
        console.log("Received request from" + request.connection.remoteAddress);
        if (++requestCount >=5) {
            response.writeHead(500);
            response.end("Some internal error has occurred! This is pod " + os.hostname() + "\n");
            return;
        }
        response.writeHead(200);
        response.end("This is v2 running in pod " + os.hostname() + "\n");
    };

    var www = http.createServer(handler);
    www.listen(8080);
    ```

2. 部署v3 版本

    ```bash
    kubectl set image deployment demodeployment nodejs=davidshi/demoapp:v3
    ```

3. 同时执行如下命令进行监视

    ```bash
    while true; do curl http://40.73.8.50; done
    ```

    ![](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210107145835.png)

4. 回滚升级

    ```bash
    kubectl rollout undo deployment demodeployment
    ```

5. 显示Deployment的滚动升级历史

    ```bash
    kubectl rollout history deployment demodeployment 
    ```

    ![](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210107150633.png)

    >还记得上面创建Deployment时用的--record参数吗？如果不给这个参数，版本历史中的CHANGE-CAUSE这栏会为空，这也会是我们很难辨别每次更新做了哪些更改。
