# SonarQube 部署和配置

## SonarQube 介绍

>SonarQube 是一款用于代码质量管理的开源工具，它主要用于管理源代码的质量。 通过插件形式，可以支持众多计算机语言，比如 java, C#, go，C/C++, PL/SQL, Cobol, JavaScrip, Groovy 等。sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具来检测你的代码，帮助你发现代码的漏洞，Bug，异味等信息。
Sonar 不仅提供了对 IDE 的支持，可以在 Eclipse和 IntelliJ IDEA 这些工具里联机查看结果；同时 Sonar 还对大量的持续集成工具提供了接口支持，可以很方便地在持续集成中使用 Sonar。

## SonarQube 能带来什么？

1. 糟糕的复杂度分布
   文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员难以理解它们，且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试。
   ![20221018115202](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221018115202.png)
2. 重复
   显然程序中包含大量复制粘贴的代码是质量低下的，sonar 可以展示源码中重复严重的地方。
   ![20221018115448](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221018115448.png)
3. 缺乏单元测试
   sonar 可以很方便地统计并展示单元测试覆盖率。
   ![20221018115610](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221018115610.png)
4. 没有代码标准
   sonar 可以通过 PMD,CheckStyle,Findbugs 等等代码规则检测工具规范代码编写。
5. 没有足够的或者过多的注释
   没有注释将使代码可读性变差，特别是当不可避免地出现人员变动时，程序的可读性将大幅下降。而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷。
6. 潜在的 bug
   sonar 可以通过 PMD,CheckStyle,Findbugs 等等代码规则检测工具检测出潜在的 bug。
   ![20221018115756](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221018115756.png)
7. 糟糕的设计
   通过sonar可以找出循环，展示包与包、类与类之间的相互依赖关系，可以检测自定义的架构规则 通过sonar可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。

## sonarqube架构简介

![20221018115944](https://raw.githubusercontent.com/shibaoxi/shareimg/master/20221018115944.png)

>CS架构

- sonarqube scanner
- sonarqube server

1. SonarQube Scanner 扫描仪在本地执行代码扫描任务
2. 执行完后，将分析报告被发送到SonarQube服务器进行处理
3. SonarQube服务器处理和存储分析报告导致SonarQube数据库，并显示结果在UI中

## 部署sonarqube

>本篇文章介绍在k8s中部署sonarqube

>SonarQube需要依赖数据库存储数据，且SonarQube7.9及其以后版本将不再支持Mysql，所以这里推荐设置PostgreSQL作为SonarQube的数据库。

创建命名空间

```bash
kubectl create ns sonarqube
```

### 部署PostgreSQL

在k8s集群部署PostgreSQL，需要将数据库的数据文件持久化，因此需要创建对应的pv，本次安装通过storageclass创建pv。由于postgre只需要集群内部连接，因此采用Headless service来创建数据库对应的svc，数据库的端口是5432，最终的yaml如下

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-sonar
  namespace: sonarqube
  labels:
    app: postgres-sonar
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-sonar
  template:
    metadata:
      labels:
        app: postgres-sonar
    spec:
      containers:
      - name: postgres-sonar
        image: postgres:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "sonarDB"
        - name: POSTGRES_USER
          value: "sonarUser"
        - name: POSTGRES_PASSWORD 
          value: "XZD12T7DPV4F"
        resources:
          limits:
            cpu: 1000m
            memory: 2048Mi
          requests:
            cpu: 500m
            memory: 1024Mi
        volumeMounts:
          - name: data
            mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data 
  namespace: sonarqube
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "aazurefile-csi-nfs" # Azure Files does not support hard links by default, which is required by postgres. You will need to create a NFS backed Azure File storage for it to work
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-sonar
  namespace: sonarqube
  labels:
    app: postgres-sonar
spec:
  clusterIP: None
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432
  selector:
    app: postgres-sonar
```

### 部署SonarQube

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube
  namespace: sonarqube
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: sonarqube:lts
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          value: "sonarUser"
        - name: SONARQUBE_JDBC_PASSWORD
          value: "XZD12T7DPV4F"
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres-sonar:5432/sonarDB"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 1000m
            memory: 1024Mi
          requests:
            cpu: 500m
            memory: 500Mi
        volumeMounts:
        - mountPath: /opt/sonarqube/conf
          name: data
          subPath: conf
        - mountPath: /opt/sonarqube/data
          name: data
          subPath: data
        - mountPath: /opt/sonarqube/extensions
          name: data
          subPath: extensions
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: sonarqube-data  

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-data 
  namespace: sonarqube
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "aks-azurefile"
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: sonarqube
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 9000
  selector:
    app: sonarqube
```

- 通过官方的sonar镜像部署，通过环境变量指定连接数据库的地址信息，同样通过storageclass来提供存储卷，通过LoadBalancer方式暴露服务。
- 与常规部署不同的是，这里对sonar通过init container进行了初始化，执行修改了容器的vm.max_map_count大小.

https://www.iblog.zone/archives/%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E6%9E%84%E5%BB%BA%E5%9F%BA%E4%BA%8Ekubernetes%E7%9A%84devops%E5%B9%B3%E5%8F%B0/#sonarqube-on-kubernetes%e7%8e%af%e5%a2%83%e6%90%ad%e5%bb%ba