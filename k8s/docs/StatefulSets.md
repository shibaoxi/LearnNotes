# StatefulSets 详解

## 概念

StatefulSet 是用来管理有状态应用的工作负载 API 对象。
StatefulSet 用来管理某 Pod 集合的部署和扩缩， 并为这些 Pod 提供持久存储和持久标识符。
和 Deployment 类似， StatefulSet 管理基于相同容器规约的一组 Pod。但和 Deployment 不同的是， StatefulSet 为它们的每个 Pod 维护了一个有粘性的 ID。这些 Pod 是基于相同的规约来创建的， 但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。
如果希望使用存储卷为工作负载提供持久存储，可以使用 StatefulSet 作为解决方案的一部分。 尽管 StatefulSet 中的单个 Pod 仍可能出现故障， 但持久的 Pod 标识符使得将现有卷与替换已失败 Pod 的新 Pod 相匹配变得更加容易。

### Statefulset与ReplicaSet的对比

> ReplicaSet 管理的pod可以比作牛，在农场中如果一个牛生病了或者被屠宰了，可以创建一个新的实例来替换。
另外Statefulset管理的又状态的pod比作宠物，是你精心喂养和照顾的，如果一只宠物死掉，不能买一只完全一样的，若要替换掉这只宠物，需要找到一只行为举止与之完全一致的宠物。

- 提供稳定的网络标识

    一个Statefulset创建的每个pod都有一个从零开始的顺序索引，这些名称是可以预知的，有规律的。区别与replicaset生成的pod名称是随机的。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210108145210.png" width=600 />

- 控制服务

    一个Statefulset通常要求你创建一个用于记录每个pod网络标记的headlessService。通过这个Service，每个pod将拥有独立的DNS记录，这样集群里其他的小伙伴或者客户端可以通过主机名方便的找到它。

- 替换消失的pod

    当Statefulset管理的一个pod消失后，Statefulset会保证重启一个新的实例替换它，新的pod与之前的pod拥有完全一致的名称和主机名。这区别与ReplicaSet。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210108150944.png" width=600 />

- 扩缩容 Statefulset

    扩容会使用下一个还没有用到的顺序索引值创建一个新的pod实例。
    当缩容时，这里可以很明确哪个pod将要被删除。而Replicaset缩容则不同，它不知道哪个pod会被删除，完全时随机的。
    缩容Statefulset将会最先删除最高索引值的实例，所以缩容的结果时可预知的。
<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210108152419.png" width=600 />

- 为每个有状态实例提供稳定的专属存储

    有状态额pod的存储必须时持久的，并且可以与pod解耦。创建Statefulset时会在pod创建之前创建持久卷声明，然后保定到一个pod实例上。
<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210108154955.png" width=600 />

    删除pod时并不会删除持久卷声明，当扩容创建一个新的pod实例时，新的实例会使用使用绑定在持久卷上的相同声明和上面的数据。
<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210108155236.png" width=600 />

---

## 使用StatefulSets

### 准备应用和容器镜像

- 创建一个有状态的应用

    ```javascript
    const http = require('http');
    const os = require('os');
    const fs = require('fs');

    const dataFile = "/var/data/demo.txt";

    function fileExists(file) {
    try {
        fs.statSync(file);
        return true;
    } catch (e) {
        return false;
    }
    }

    var handler = function(request, response) {
    if (request.method == 'POST') {
        var file = fs.createWriteStream(dataFile);
        file.on('open', function (fd) {
        request.pipe(file);
        console.log("New data has been received and stored.");
        response.writeHead(200);
        response.end("Data stored on pod " + os.hostname() + "\n");
        });
    } else {
        var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
        response.writeHead(200);
        response.write("You've hit " + os.hostname() + "\n");
        response.end("Data stored on this pod: " + data + "\n");
    }
    };

    var www = http.createServer(handler);
    www.listen(8080);
    ```

- 创建Dockerfile文件

    ```Dockerfile
    FROM node:7
    ADD app.js /app.js
    ENTRYPOINT ["node", "app.js"]
    ```

- 构建镜像并推送

  ```bash
  docker build -t davidshi/demopet .
  docker push davidshi/demopet
  ```

### 通过StatefulSets 部署应用

- 在部署Statefulset之前需要先部署Headless服务

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: demo-headless
  spec:
    clusterIP: None #设置为none，就标记它时一个headless service，它使得你的pod之间可以彼此发现
    selector:
      app: demo-ss
    ports:
    - name: http
      port: 80
  ```

- 部署statefulset

  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: demo-statefulset
  spec:
    selector:
      matchLabels:
        app: demo-ss
    serviceName: demo-headless
    replicas: 3
    template:
      metadata:
        labels:
          app: demo-ss
      spec:
        containers:
        - name: demoapp
          image: davidshi/demopet
          ports:
          - name: http
            containerPort: 8080
          volumeMounts:
          - name: data
            mountPath: /var/data
    volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-premium
        resources:
          requests:
            storage: 5Gi
  ```

### 验证检查

- 检查生成的持久卷声明

  ```bash
  # kubectl get pvc
  NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
  data-demo-statefulset-0   Bound    pvc-593a7b1a-2b3c-4310-9d00-43473e0583d9   5Gi        RWO            managed-premium   22h
  data-demo-statefulset-1   Bound    pvc-d06b184f-4ed2-4e28-b4be-aa443f024942   5Gi        RWO            managed-premium   22h
  data-demo-statefulset-2   Bound    pvc-a8919919-2750-4c88-9e42-ad783dfcd347   5Gi        RWO            managed-premium   22h
  ```

  > 生成的持久卷声明的名称由在volumeClaimTemplate字段中定义的名称和每个pod的名称组成。

- 使用pod
  1. 在客户端运行``` kubectl proxy ```

      ```bash
      Starting to serve on 127.0.0.1:8001
      ```

  2. 发送一个如下所示的请求到demo-statefulset-0 pod上：

      ```bash
      # curl localhost:8001/api/v1/namespaces/default/pods/demo-statefulset-0/proxy/
      You've hit demo-statefulset-0
      Data stored on this pod: No data posted yet
      ```

  3. 当应用收到一个post请求时，它把请求的主体内容保存到本地一个文件中，发送一个post请求到demo-statefulset-0

      ```bash
      # curl -X POST -d "Hey there! this greeting was submitted to demo-statefulset-0." localhost:8001/api/v1/namespaces/default/pods/demo-statefulset-0/proxy/
      Data stored on pod demo-statefulset-0
      ```

  4. 你发送的数据现在已经保存到pod中，那让我们检查一下当你再次发送一个get请求时，它是否会返回存储的数据：

      ```bash
      # curl localhost:8001/api/v1/namespaces/default/pods/demo-statefulset-0/proxy/                                                                          
      You've hit demo-statefulset-0
      Data stored on this pod: Hey there! this greeting was submitted to demo-statefulset-0.
      ```

  5. 我们看看其他节点如何

      ```bash
      # curl localhost:8001/api/v1/namespaces/default/pods/demo-statefulset-1/proxy/
      You've hit demo-statefulset-1
      Data stored on this pod: No data posted yet
      ```

      > 这个结果符合我们的期望，每个节点拥有独自的状态。

  6. 删除一个有状态的pod来检查重新调度的pod是否关联了相同的存储

      ```bash
      # kubectl delete po demo-statefulset-0
      pod "demo-statefulset-0" deleted
      ```

  7. 查看pod，可以发现新的替代pod正在被创建

      ```bash
      # kubectl get po
      NAME                              READY   STATUS              RESTARTS   AGE
      demo-statefulset-0                0/1     ContainerCreating   0          29s
      demo-statefulset-1                1/1     Running             1          24h
      demo-statefulset-2                1/1     Running             1          24h
      ```

  8. 查看新替换的pod是否拥有与之前的pod一样的标记和数据

      ```bash
      # curl localhost:8001/api/v1/namespaces/default/pods/demo-statefulset-0/proxy/
      You've hit demo-statefulset-0
      Data stored on this pod: Hey there! this greeting was submitted to demo-statefulset-0.
      ```

      > 从pod返回的信息表明它的主机名和持久化数据与之前pod是完全一致的。

### 扩缩容Statefulset

**扩缩容有以下几点需要注意：**

- 缩容一个Statefulsset，只会删除对应的pod，留下卸载后的持久卷声明；

- 缩容超过一个实例的时候，会首先删除最高索引的值；

- 扩缩容都是顺序执行的，只有上一个被完全创建/删除，下一个才会创建/删除

### 在Statefulset中发现伙伴节点

Kubernetes通过一个headless service 创建SRV记录来指向pod的主机名。
> SRV 记录用来指向提供指定服务的服务器的主机名和端口号。

我们接下来部署一个简单应用来了解这一特点：

- 修改你的应用代码如下：

  ```javascript

  const http = require('http');
  const os = require('os');
  const fs = require('fs');
  const dns = require('dns');

  const dataFile = "/var/data/kubia.txt";
  const serviceName = "kubia.default.svc.cluster.local";
  const port = 8080;


  function fileExists(file) {
    try {
      fs.statSync(file);
      return true;
    } catch (e) {
      return false;
    }
  }

  function httpGet(reqOptions, callback) {
    return http.get(reqOptions, function(response) {
      var body = '';
      response.on('data', function(d) { body += d; });
      response.on('end', function() { callback(body); });
    }).on('error', function(e) {
      callback("Error: " + e.message);
    });
  }

  var handler = function(request, response) {
    if (request.method == 'POST') {
      var file = fs.createWriteStream(dataFile);
      file.on('open', function (fd) {
        request.pipe(file);
        response.writeHead(200);
        response.end("Data stored on pod " + os.hostname() + "\n");
      });
    } else {
      response.writeHead(200);
      if (request.url == '/data') {
        var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
        response.end(data);
      } else {
        response.write("You've hit " + os.hostname() + "\n");
        response.write("Data stored in the cluster:\n");
        //通过DNS查询获取SRV记录
        dns.resolveSrv(serviceName, function (err, addresses) {
          if (err) {
            response.end("Could not look up DNS SRV records: " + err);
            return;
          }
          var numResponses = 0;
          if (addresses.length == 0) {
            response.end("No peers discovered.");
          } else {
              //与SRV记录对应的每个pod通信获取其数据
            addresses.forEach(function (item) {
              var requestOptions = {
                host: item.name,
                port: port,
                path: '/data'
              };
              httpGet(requestOptions, function (returnedData) {
                numResponses++;
                response.write("- " + item.name + ": " + returnedData + "\n");
                if (numResponses == addresses.length) {
                  response.end();
                }
              });
            });
          }
        });
      }
    }
  };

  var www = http.createServer(handler);
  www.listen(port);
  
  ```

- 更新 Statefulset

  ```yaml
  spec:
      containers:
      - name: demoapp
        image: davidshi/demo-pet-peers #修改镜像为上面更新的镜像名称
  ```

- 通过service 向集群数据存储中写入数据

  ```bash
  # curl -X POST -d "The sun is shinning" localhost:8001/api/v1/namespaces/default/services/demoapp-public/proxy/
  Data stored on pod demo-statefulset-1
  # curl -X POST -d "The weather is sweet" localhost:8001/api/v1/namespaces/default/services/demoapp-public/proxy/                
  Data stored on pod demo-statefulset-2
  # curl -X POST -d "I decide who I am" localhost:8001/api/v1/namespaces/default/services/demoapp-public/proxy/                     
  Data stored on pod demo-statefulset-0
  ```

- 从数据存储中读取数据

  ```bash
  # curl localhost:8001/api/v1/namespaces/default/services/demoapp-public/proxy/
  You've hit demo-statefulset-0
  Data stored in the cluster:
  - demo-statefulset-0.demo-headless.default.svc.cluster.local: The weather is sweet
  - demo-statefulset-1.demo-headless.default.svc.cluster.local: The sun is shinning
  - demo-statefulset-2.demo-headless.default.svc.cluster.local: I decide who I am
  ```
