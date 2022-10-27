# ConfigMap and Secret

## 在fortune-args文件夹下面修改foruneloop.sh

```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds
mkdir /var/htdocs
while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

## 修改Dockerfile

```bash
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod +x /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]
```

## 重新构建镜像并推送到Docker Hub

```bash
docker build -t davidshi/fortune:args .
docker push davidshi/fortune:args
```

## 在本地启动镜像进行测试

```bash
docker run -it davidshi/fortune:args
#添加参数
docker run -it davidshi/fortune:args 20
```

## 在Kubernetes中覆盖命令和参数

### 在Docker与Kubernetes中指定可执行程序及其参数

| Docker | Kubernetes | 描述 |
| :----: | :----: | :----: |
| ENTRYPOINT | command | 容器中运行的可执行文件 |
| CMD | args | 传给可执行文件的参数 |

### 自定义间隔值运行fortune pod

```yml
apiVersion: v1
kind: Pod
metadata:
  name: demo-fortune-args-2s
spec:
  volumes:  #一个名为html的单独emptyDir卷，挂载在上面两个容器中
  - name: html
    emptyDir: {}
  containers:
  - image: davidshi/fortune:args
    args: ["2"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - name: web-server
    image: nginx:alpine #第二个容器命名为web-server，运行nginx:alpine镜像
    volumeMounts:
    - name: html  #与上面相同的卷挂载在/usr/share/nginx/html目录上，设为只读
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
```

### 设置容器环境变量

容器化应用通常会使用环境变量作为配置源
> 通过环境变量配置fortune镜像中的间隔值
>> 修改如下脚本移除脚本中的INTERVAL初始化所在的行

```bash
#!/bin/bash
trap "exit" SIGINT
echo Configured to generate new fortune every $INTERVAL seconds
mkdir /var/htdocs
while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

>> 重建镜像然后推送到docker hub中
    docker build -t davidshi/fortune:env .
    docker push davidshi/fortune:env
>> 在pod yaml文件中定义环境变量

```yml
apiVersion: v1
kind: Pod
metadata:
  name: demo-fortune-args-2s
spec:
  volumes:  #一个名为html的单独emptyDir卷，挂载在上面两个容器中
  - name: html
    emptyDir: {}
  containers:
  - image: davidshi/fortune:env
    env:  #在环境变量列表中添加一个新的变量
    - name: INTERVAL
      value: "30"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - name: web-server
    image: nginx:alpine #第二个容器命名为web-server，运行nginx:alpine镜像
    volumeMounts:
    - name: html  #与上面相同的卷挂载在/usr/share/nginx/html目录上，设为只读
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
```

### ConfigMap

#### 介绍

kubernetes 允许将配置选项分离到单独的资源对象ConfigMap中。本质上就是一个键/值对映射，值可以时短字变量，也可以是完整的配置文件

### 使用configMap卷将条目暴露为文件

configMap卷会将ConfigMap中的每个条目均暴露成一个文件，运行在容器中的进程可以通过读取文件内容获取对应的条目值
> 这种方法主要适用于传递较大的配置文件给容器，同样可以用于传递较短的变量值

### 示例

* 创建ConfigMap
    > 开启gzip压缩的Nginx配置文件

    ``` bash
    server {
        listen 80;
        server_name www.contoso.com;
        gzip on;
        #开启对文本文件与xml文件的gzip压缩
        gzip_type text/plain application/xml;   
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
    ```

    >创建一个configmap-files文件夹，把上面的配置文件存储在该文件夹下面，另外在创建一个sleep-interval的文本文件，写入值25

    > 从文件夹创建ConfigMap

    ```bash
    kubectl create configmap fortune-config --from-file=configmap-files
    ```

* 创建pod引用ConfigMap
    > 编辑yaml文件

    ```yml
    apiVersion: v1
    kind: Pod
    metadata:
    name: demo-fortune-configmap-volume
    spec:
    volumes:  
    - name: html  #一个名为html的单独emptyDir卷，挂载在上面两个容器中
        emptyDir: {}
    - name: config  #定义一个名为config的ConfigMap卷，引用fortune-config
        configMap:
        name: fortune-config
    containers:
    - image: davidshi/fortune:env
        env:  #引入环境变量
        - name: INTERVAL  #设置环境变量INTERVAL
        valueFrom:
            configMapKeyRef:  #用ConfigMap 初始化，不设定固定值
            name: fortune-config  #引用ConfigMap名称
            key: sleep-interval #环境变量值被设置为ConfigMap下对应键的值
        name: html-generator
        volumeMounts:
        - name: html
        mountPath: /var/htdocs
    - name: web-server
        image: nginx:alpine #第二个容器命名为web-server，运行nginx:alpine镜像
        volumeMounts:
        - name: html  #与上面相同的卷挂载在/usr/share/nginx/html目录上，设为只读
        mountPath: /usr/share/nginx/html
        readOnly: true
        - name: config
        mountPath: /etc/nginx/conf.d  #挂载configMap卷至这个位置
        readOnly: true
        ports:
        - containerPort: 80
        protocol: TCP
    ```

### Secret

#### Secret介绍

Secret结构和ConfigMap类似，均是键值对的形式，Secret的使用方法也与ConfigMap相同

* 将Secret条目作为环境变量传递给容器
* 将Secret条目暴露为卷中的文件

> Secret 只会存放在主机的内存中，不会写入物理存储，这样就需要保障主节点的安全。

#### 创建Secret

1. 在本地机器上生成证书和私钥文件

    ```bash
    openssl genrsa -out https.key 2048
    openssl req  -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.contoso.com
    ```

2. 创建一个额外的字符串bar的虚拟文件foo
    > 后面这个会验证Secret和ConfigMap的区别
    ```echo bar > foo```

3. 创建Secret

    ```bash
    kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
    ```

#### Secret和ConfigMap的区别

> Secret 条目的内容会被以Base64格式编码，而ConfigMap直接以纯文本展示，因此Secret的条目可以涵盖二进制数据，另外Secret的大小限制至于1MB

#### 在pod中使用Secret

1. 修改demo-nginx-config.conf文件内容，并创建新的ConfigMap对象

    ``` bash
    server {
        listen 80;
        listen 443 ssl;
        server_name www.contoso.com;
        #/etc/nginx的相对位置
        ssl_certificate certs/https.cert;
        ssl_certificate_key certs/https.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers HIGH:!aNULL:!MD5;
        gzip on;
        #开启对文本文件与xml文件的gzip压缩
        gzip_types text/plain application/xml;   
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
    ```

2. 挂载fortune-secret 到pod，需要创建一个新的fortune-https pod，将含有证书的secret卷挂载到pod中的web-server容器。

    ```yml
    apiVersion: v1
    kind: Pod
    metadata:
    name: demo-fortune-https
    spec:
    volumes:  
    - name: html  #一个名为html的单独emptyDir卷，挂载在上面两个容器中
        emptyDir: {}
    - name: config  #定义一个名为config的ConfigMap卷，引用fortune-config
        configMap:
        name: fortune-config-https
    - name: certs
        secret:
        secretName: fortune-https #这里引用fortune-https Secret来定义secret卷
    containers:
    - image: davidshi/fortune:env
        env:  #引入环境变量
        - name: INTERVAL  #设置环境变量INTERVAL
        valueFrom:
            configMapKeyRef:  #用ConfigMap 初始化，不设定固定值
            name: fortune-config-https  #引用ConfigMap名称
            key: sleep-interval #环境变量值被设置为ConfigMap下对应键的值
        name: html-generator
        volumeMounts:
        - name: html
        mountPath: /var/htdocs
    - name: web-server
        image: nginx:alpine #第二个容器命名为web-server，运行nginx:alpine镜像
        volumeMounts:
        - name: html  #与上面相同的卷挂载在/usr/share/nginx/html目录上，设为只读
        mountPath: /usr/share/nginx/html
        readOnly: true
        - name: config
        mountPath: /etc/nginx/conf.d  #挂载configMap卷至这个位置
        readOnly: true
        - name: certs
        mountPath: /etc/nginx/certs/  #Nginx 默认从/etc/nginx/certs中读取证书和密钥文件，需要将Secret卷挂载于此。
        readOnly: true
        ports:
        - containerPort: 80
        - containerPort: 443
    ```

3. 测试Nginx是否正在使用Secret中的证书与密钥

    ```bash
    kubectl port-forward demo-fortune-https 8443:443
    curl https://localhost:8443 -k -v
    ```
