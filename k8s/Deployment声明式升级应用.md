## Deployment 声明式升级应用

> ###### 概念
>
> 一个Deployment控制器为pods和ReplicaSets提供声明式更新能力

###### 以下是Deployments的常用场景

- 创建 Deployment 以将 ReplicaSet 上线。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
- 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新 Deployment 的修订版本。
- 扩大 Deployment 规模以承担更多负载。
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
- 使用 Deployment 状态 来判定上线过程是否出现停滞。
- 清理较旧的不再需要的 ReplicaSet

---

#### 创建Deployment

###### 准备镜像
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
###### 创建Deployment 
* 创建一个demo-deployment-v1.yaml
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: demoDeployment
    spec:
    replicas: 3
    template:
        metadata:
        name: demoapp
        labels:
            app: demoapp
        spec:
        containers:
        - image: davidshi/demoapp:v1
            name: nodejs
    ```