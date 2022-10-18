# Helm简介与安装

## Helm介绍

### 简介

很多人都使用过Ubuntu下的ap-get或者CentOS下的yum, 这两者都是Linux系统下的包管理工具。采用apt-get/yum,应用开发者可以管理应用包之间的依赖关系，发布应用；用户则可以以简单的方式查找、安装、升级、卸载应用程序。

我们可以将Helm看作Kubernetes下的apt-get/yum。Helm是Deis (<https://deis.com/>) 开发的一个用于kubernetes的包管理器。每个包称为一个Chart，一个Chart是一个目录（一般情况下会将目录进行打包压缩，形成name-version.tgz格式的单一文件，方便传输和存储）。

对于应用发布者而言，可以通过Helm打包应用，管理应用依赖关系，管理应用版本并发布应用到软件仓库。

对于使用者而言，使用Helm后不用需要了解Kubernetes的Yaml语法并编写应用部署文件，可以通过Helm下载并在kubernetes上安装需要的应用。

除此以外，Helm还提供了kubernetes上的软件部署，删除，升级，回滚应用的强大功能。

### 用途

做为 Kubernetes 的一个包管理工具，Helm具有如下功能：

- 创建新的 chart
- chart 打包成 tgz 格式
- 上传 chart 到 chart 仓库或从仓库中下载 chart
- 在Kubernetes集群中安装或卸载 chart
- 管理用Helm安装的 chart 的发布周期

### 组件与架构

<img src="https://i.loli.net/2021/08/30/tFqwk1IzyEd8avS.png" width=600 />

#### 组件介绍

- **Helm客户端**

    Helm是一个命令行下的客户端工具。主要用于Kubernetes应用程序Chart的创建、打包、发布以及创建和管理本地和远程的Chart仓库。
- **~~Tiller~~**

    ~~Tiller是Helm的服务端，部署在Kubernetes群集中。Tiller用于接收Helm的请求，并根据Chart生成Kubernetes的部署文件（Helm 称为 Release），然后提交给Kubernetes创建应用。Tiller还提供了Release的升级、删除、回滚等一系列功能。~~
    > Helm 3 中移除了Tiller，版本相关的数据直接存储在Kubernetes中。
- **Chart**
    就是Helm的一个包(package)，包含一个应用所有的Kubernetes manifest模板，类似于YUM的RPM或者APT的dpkg文件。
- **Repoistory**
    Helm的软件仓库，Repository本质上是一个web服务器，该服务器保存了一系列的Chart软件包以供用户下载，并且提供了一个该Repository的Chart包的清单文件以供查询。Helm可以同时管理多个不同的Repository。
- **Release**
    使用 ```helm install``` 命令在Kubernetes集群部署的Chart称为Release。
    可以理解成Chart部署的一个实例。通过Chart在
    > 需要注意的是：Helm 中提到的 Release 和我们通常概念中的版本有所不同，这里的 Release 可以理解为 Helm 使用 Chart 包部署的一个应用实例。
    Release名称可以在不同命名空间重用
    helm install 不再默认生成一个 Release 的名称，除非指定了 --generate-name。

## helm 安装部署

### Helm 客户端安装

Helm 的安装方式很多，这里采用二进制的方式安装。

```bash
# 从官网下载最新版的二进制安装包到本地: https://github.com/helm/helm/releases/
curl -O https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz
#解压压缩包
tar -zxvf helm-v3.6.3-linux-amd64.tar.gz
#把helm指令放到bin目录下
mv linux-amd64/helm /usr/local/bin/helm
#验证
helm version
```

## Chart 介绍

### Chart 简介

charts就是Helm要使用的包格式。它是一个描述Kubernetes相关资源的文件集合。一个charts可以用来部署一个简单的或者复杂的应用。通过创建一个特定目录树和文件来形成chart，病将他们打包到一个带有版本号的压缩包中就可以进行部署了。

### Chart 文件结构

chat是一个具有特定目录树结构和文件的集合。目录名称是chart的命名（没有版本号），比如描述为Mysql的chat会被存储在mysql目录中：

```bash
mysql/
  Chart.yaml            # 用于描述chart信息的yaml文件
  LICENSE               # 可选：用于存储关于chart的LICENSE文件
  README.md             # 可选：README 文件
  values.yaml           # 用于存储 chart 所需要的默认配置
  values.schema.json    # 可选：一个使用JSON结构的 values.yaml 文件
  charts/               # 包含chart 所以来的其他chart
  crds/                 # 自定义资源的定义
  templates/            # chart 模板文件，引入变量值后可以生成用于kubernetes的manifest文件
  templates/NOTES.txt   # 可选：包含简短使用说明的文本文件
```

### Chart.yaml 文件结构和说明

Chart.yaml 文件是chart 包中必须存在的文件，它包含以下字段

```yaml
apiVersion:             # chart API 版本信息, 通常是 "v2" (必须)
name:                   # chart 的名称 (必须)
version:                # chart 包的版本 (必须)
kubeVersion:            # 指定 Kubernetes 版本 (可选)
type:                   # chart类型 （可选）
description:            # 对项目的描述 (可选)
keywords:
  -                     # 有关于项目的一些关键字 (可选)
home:                   # 项目 HOME 页面的 URL 地址 (可选)
sources:
  -                     # 项目源码的 URL 地址 (可选)
dependencies:           # chart 必要条件列表 （可选）
  - name:               # chart名称 (nginx)
    version:            # chart版本 ("1.2.3")
    repository:         # （可选）仓库URL ("https://example.com/charts") 或别名 ("@repo-name")
    condition:          # （可选） 解析为布尔值的yaml路径，用于启用/禁用chart (e.g. subchart1.enabled )
    tags:               # （可选）
      -                 # 用于一次启用/禁用 一组chart的tag
    import-values:      # （可选）
      -                 # ImportValue 保存源值到导入父键的映射。每项可以是字符串或者一对子/父列表项
    alias:              # （可选） chart中使用的别名。当你要多次添加相同的chart时会很有用
maintainers:            # （可选）维护者信息
  - name:               # 维护者的名称
    email:              # 维护者的邮件地址
    url:                # 维护者的个人主页
engine: gotpl           # 模板引擎的名称（可选，默认为 gotpl）
icon:                   # （可选）指定 chart 图标的 SVG 或 PNG 图像的 URL
appVersion:             # 应用程序包含的版本
deprecated:             # （可选，使用布尔值）该 chart 是否被废弃
annotations:
  example:              # 按名称输入的批注列表 （可选）.
```

- **apiVersion 字段**
    对于部分仅支持helm 3的chart，apiVersion应该指定为v2，对于可以同时支持helm 3 和 helm 2版本的chart，可以将其设置为v1
- **appVersion 字段**
    appVersion 字段与 version 字段并没有直接的联系。这是指定应用版本的一种方式。比如一个 chart 可能有一个 appVersion:"6.0.1"，表示包含在 chart 中应用的版本是 6.0.1。此字段仅供参考，对 chart 版本计算没有影响。
    </br>
    > 版本号需要使用双引号括起来，否则在某些场景下，版本号不会被解析为字符串而导致其他的问题。从 Helm v3.5.0 版本开始，helm create 命令会自动将默认的 appVersion 用引号括起来
- **kubeVersion 字段**
    kubeVersion 用于指定受支持的 Kubernetes 版本，Helm 在安装的时候会验证版本信息。如果在不受支持的 Kubernetes 上安装 chart 会显示失败。</br>
    版本约束可以包括空格分隔和比较运算符，比如：>= 1.13.0 < 1.15.0
    </br>
    >版本号需要使用双引号括起来，否则在某些场景下，版本号不会被解析为字符串而导致其他的问题。从 Helm v3.5.0 版本开始，helm create 命令会自动将默认的 appVersion 用引号括起来
- **type 字段**
    type字段定义了chart的类型。有两种类型： application 和 library。 application 是默认类型，是可以完全操作的标准 chart。库类型提供针对 chart 构建的实用程序和功能。 库类型 chart 与应用类型 chart 不同，因为它不能安装，通常不包含任何资源对象.
    >应用类型chart可以作为库类型chart使用。可以通过将类型设置为library来实现。然后这个库就被渲染成一个库类型chart，所有的实用程序和功能都可以使用。所有的资源对象不会被渲染。

## 编写一个简单的Chart并发布

### 创建第一个Chart包

使用helm create 命令即可创建一个chart，其中包含完整地目录结构

```bash
 helm create myfirstchart
 # 会生成如下目录
 .
└── myfirstchart
    ├── Chart.yaml
    ├── charts
    ├── templates
    │   ├── NOTES.txt
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

删除模板文件夹下面的文件

```bash
rm -rf myfirstchart/templates/*
```

### 创建模板文件

在templates目录下面新建一个configmap.yaml的文件，输入如下内容：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myfirstchart-configmap
data:
  myvalue: "Hello World"
```

### 安装chart

```bash
helm install myconfig ./myfirstchart/
# 返回如下信息
NAME: myconfig
LAST DEPLOYED: Thu Apr 14 11:11:56 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

>需要注意的是，在3.x版本中使用helm install 的时候必须要指定release名称，或者使用--generate-name 选项来自动生成release名称。上面的命令使用myconfig作为release名称。

### 添加对象调用

在前面创建的模板中，可以看到在metadata.name中设置的值是一个固定值，这样的模板是无法在kubernetes中多次部署的，所以我们可以试着在每次安装chart时，都自动将metadata.name的值设置为release的名称。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
```

## Helm3 内置对象详解

![20220414114638](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220414114638.png)

### Release 对象

Release 对象描述了版本发布自身的一些信息。它包含以下对象：

|对象名称|描述|
| :- | :- |
|.Release.Name|release的名称|
|.Release.Namespace|release的命名空间|
|.Release.IsUpgrade|如果当前操作是升级或者回滚的话，该值为true，安装为false|
|.Release.IsInstall|如果当前操作是安装的话，该值为true|
|.Release.Revision|获取此次修订的版本号，初次安装时为1，每次升级或者回滚都会递增|
|.Release.Service|获取渲染当前模板的服务名称，一般都是Helm|

把如下命令贴到configmap.yaml文件中

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Release.Namespace }}
data:
  value1: {{ .Release.IsUpgrade }}
  value2: {{ .Release.IsInstall }}
  value3: {{ .Release.Revision }}
  value4: {{ .Release.Service }}
```

执行如下命令，并查看返回结果

```bash
helm install mydemochat2 ./myfirstchart/ --debug --dry-run
# 返回结果
# Source: myfirstchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mydemochat2-configmap
  namespace: default
data:
  value1: false
  value2: true
  value3: 1
  value4: Helm
```

### Values 对象

Values 对象描述的是Values.yaml文件中的内容，默认为空。使用Values对象客户获取到values.yaml文件中已定义的任何数值。

|Value键值对|获取方式|
|:--|:--|
|name:Jack|.Values.name|
|info:</br> &ensp; name:Jack|.Values.info.name|

清空values.yaml文件内容，然后添加name: Jack 内容

编辑configmap.yaml文件，如下内容：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Release.Namespace }}
data:
  value1: {{ .Release.IsUpgrade }}
  value2: {{ .Release.IsInstall }}
  value3: {{ .Release.Revision }}
  value4: {{ .Release.Service }}
  value5: {{ .Values.name }}
```

执行如下命令，并查看返回结果

```bash
helm install mydemochat2 ./myfirstchart/ --debug --dry-run
# 返回结果
# Source: myfirstchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mydemochat2-configmap
  namespace: default
data:
  value1: false
  value2: true
  value3: 1
  value4: Helm
  value5: Jack
```

### Chart 对象

|对象名称|描述|
|:--|:--|
|.Chart.Name|获取Chart的名称|
|.Chart.Version|获取Chart的版本|

编辑configmap文件，然后执行测试没查看返回结果

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Release.Namespace }}
data:
  value1: {{ .Release.IsUpgrade }}
  value2: {{ .Release.IsInstall }}
  value3: {{ .Release.Revision }}
  value4: {{ .Release.Service }}
  value5: {{ .Values.name }}
  value6: {{ .Chart.Name }}
  value7: {{ .Chart.Version }}

# 返回结果如下

apiVersion: v1
kind: ConfigMap
metadata:
  name: mydemochat2-configmap
  namespace: default
data:
  value1: false
  value2: true
  value3: 1
  value4: Helm
  value5: Jack
  value6: myfirstchart
  value7: 0.1.0
```

### Capabilities 对象

Capabilities 对象提供了关于Kubernetes群集相关的信息。该对象有如下方法：

|对象名称|描述|
|:--|:--|
|.Capabilities.APIVersions|返回Kubernetes集群API版本信息集合|
|.Capabilities.APIVersion.Has$version|用于检测指定的版本或资源在Kubernetes集群中是否可用，例如batch/v1或apps/v1/Deployment|
|.Capabilities.KubeVersion 和 Capabilities.KubeVersion.Version|都用于获取Kubernetes的版本号|
|.Capabilities.KubeVersion.Major|Kubernetes的主版本号|
|.Capabilities.KubeVersion.Minor|Kubernetes的小版本号|

修改configmap.yaml文件，并查看结果

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  value1: '{{ .Capabilities.APIVersions }}'
  value2: '{{ .Capabilities.KubeVersion.Version }}'
  value3: '{{ .Capabilities.KubeVersion.Major }}'
  value4: '{{ .Capabilities.KubeVersion.Minor }}'
  value5: '{{ .Capabilities.APIVersions.Has "apps/v1/Deployment" }}'
```

### Template 对象

Template 对象用于获取当前模板的信息，它包含如下两个对象

|对象名称|描述|
|--|--|
|.Template.Name|用于获取当前模板的名称和路径（例如: mychart/templates/mytemplate.yaml）|
|.Template.BasePath|用于获取当前模板的路径（例如：mychart/templates）|

## 函数

### 内置函数

#### quote和squote函数

通过向quote或squote函数中传递一个参数，既可以为这个参数添加一个双引号(quote)或单引号（squote）

#### 管道符

函数的用法是 functionName arg1 arg2...，然而实际使用中，我们更偏向于使用管道符 **|** 来将参数传递给函数，也就是这种格式：**arg1 | functionName**.这样使用可以使结构看起来更加直观，并且可以方便使用多个函数对数据进行链式处理。

示例:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  value1: {{ .Values.name | quote }}
  value2: {{ .Values.name | squote }}
```

#### upper和lower函数

使用upper和lower函数可以分别将字符串转换为答谢和小写的样式

示例：

```yaml
# 修改value.yaml文件如下
name1: jack
name2: JACK
# 修改configmap.yaml文件
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  value1: {{ .Values.name1 | quote | upper }}
  value2: {{ .Values.name2 | squote | lower }}
```

#### repeat函数

使用repeat函数可以将指定字符串重复输出指定的次数，repeat函数可以带有一个参数，用于设置重复多少次。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Release.Namespace }}
data:
  value1: {{.Values.name1 | repeat 3 | quote}}
```

#### default函数

可以使用该函数指定一个默认值，这样当引入的值不存在时，就可以使用这个默认值。

示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  value1: {{ .Values.address | default "Bejing" | quote }}
```

#### lookup函数

lookup 函数用于在当前的Kubernetes群集中获取一些资源的信息，功能有些类似于kubectl get 函数的格式如下：

```bash
lookup "apiVersion" "kind" "namespace" "name"
```

其中“namespace”和“name”都是可选的。或者可以指定为空字符串。函数执行完成后会返回特定的资源或者是资源列表。

例如在Kubernetes集群中，我们需要获取指定命名空间下的指定Pod的信息，命令应该如下：

```bash
kubectl get pods kafka-0 -n ns-kafka
```

那么在Helm中使用lookup函数的格式如下：

```bash
lookup "v1" "Pod" "ns-kafka" "kafka-0"
```

示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  value1: {{ (lookup "v1" "Namespace" "" "default").metadata.creationTimestamp | quote }}
```

#### 逻辑和流控制函数

|名称|描述|
|--|--|
|and|返回两个参数的逻辑与结果（布尔值），也就是说如果两个参数都返回为真，则结果为true。|
|or|判断两个参数的逻辑或的关系，返回第一个不为空的参数或者是返回最后一个参数|
|not|用于对参数的布尔值取反|
|ne|用于判断两个参数是否不相等，如果不等于则为true，等于则为false|
|eq|用于判断两个参数是否相当，如果等于则为true，不等于则为false|
|lt|lt函数用于判断第一个参数是否小于第二个参数，如果小于则为true，如果大于则为false|
|gt|gt函数用于判断第一个参数是否大于第二个参数，如果大于则为true。如果小于则为false|
|ge|判断第一个参数是否小于等于第二个参数，如果成立则为true，如果不成立则为false|
|le|判断第一个参数是否小于等于第二个参数，如果成立则为true，如果不成立则为false|
|default|用来设置一个默认值，在参数的值为空的情况下，则会使用默认值|
|empty|用于判断给定值是否为空，如果为空则返回true|
|coalesce|用于扫描一个给定的列表，并返回第一个非空的值|
|ternary|接受两个参数和一个test值，如果test的布尔值为true，则返回第一个参数的值|

- and 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    data1: {{ and .Values.name1 .Values.name2 | quote}}
  ```

- or 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    type1: {{ or 1 "" 3 | quote }}
    type2: {{ or 1 2 "" | quote }}
    type3: {{ or "" 2 3 | quote }}
    type4: {{ or "" "" 3 | quote }}
    type5: {{ or "" "" "" | quote }}

  ```

- not 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    type1: {{ not 1 | quote }}
    type2: {{ not "" | quote }}
  ```

- empty 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    type1: {{ 0 | empty }}
    type2: {{ 1 | empty }}
  ```

- coalesce 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    type1: {{ coalesce 0 1 2 }}
    type2: {{ coalesce "" false "Matt" }}
  ```

- ternary 函数示例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ .Release.Name }}-configmap
  data:
    type1: {{ ternary "First" "Second" true | quote }}
    type2: {{ ternary "First" "Second" false | quote }}
    type3: {{ empty "" | ternary "First" "Second" | quote }}
    type4: {{ empty hello | ternary "First" "Second" | quote }}
  ```

#### 字符串函数

##### print和println函数

print和println函数用于将所有的参数按照字符串进行输出，与print不同的是，println会在每个字符串后面添加一个空格，并且会在输出的末尾添加一个换行符。
如果参数中包含非字符串类型，那么输出的时候会转成字符串类型。当相邻两个参数不是字符串时，会在他们中间添加一个空格

```yaml
data:
  type1: {{ print "this is" "test" "message" }}
  type2: {{ println "this is" "test" "message" }}
  type3: {{ print "this is" 2 3 "message" }}
```

##### printf 函数

printf 函数用于格式化输出字符串内容，并且支持使用占位符。占位符取决于传入参数的类型。

printf 函数常用的类型-整形

- %b ：二进制
- %c ：表示普通字符
- %d ：十进制
- %o ：8进制
- %O ：带Oo前缀的8进制
- %q ：安全转义的单引号字符
- %x ：16进制。使用小写字符a-f
- %X ：16进制。使用大写字符A-F
- %U ：Unicode格式

printf 函数常用的类型-浮点数和复杂数

- %f : 无指数的小数 ，比如 123.456

printf 函数常用的类型-字符串

- %s : 未解析的二进制字符串或切片

printf 函数常用的类型-布尔值

- %t ： 输出指定额布尔值

##### trim 函数

- trim 函数可以用来去除字符串两边的空格。
- trimAll 函数用于一处字符串中指定的字符。
- trimPrefix 和 trimSuffix函数 分别用于移除字符串中指定的前缀和后缀。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  data1: {{ trim " Jack " }}
  data2: {{ trimAll "%" "%Jack%%%" }}
  data4: {{ trimPrefix "-" "-Jack" }}
  data2: {{ trimSuffix "+" "Jack+" }}
```

##### lower和upper函数

lower 和 upper 函数分别用于将字符串转换为小写或大写

示例：

- lower "JACK"
- upper "Hello"

##### title和untitle函数

title 函数用于将首字母转换为大写，untitle 函数用于将首字母大写转换为小写

示例：

- title "message"
- untitle "Hello"

##### snakecase camelcase 和 kebabcase 函数

- snakecase 函数用于将驼峰写法转换为下划线命名写法

    ```bash
    snakecase "UserName" # 返回结果 user_name
    ```

- camelcase 函数用于将下划线命名写法转换为驼峰写法

    ```bash
    camelcase "user_name" # 返回结果 UserName
    ```

- kebabcase 函数用于驼峰写法转换为中划线写法

    ```bash
    kebabcase "UserName" # 返回结果 user-name
    ```
