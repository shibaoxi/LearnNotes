# Azure 资源管理器模板介绍

## 定义

资源管理器模板可精确定义部署中的所有资源管理器资源。 资源管理器模板可部署到一个资源组中，作为一个操作。

资源管理器模板是 JSON 文件，使其以声明性自动化形式呈现。 声明性自动化是指用户定义所需资源，而不是定义创建方式。 换种说法，用户定义所需内容，资源管理器负责确保正确部署该资源。

可将声明性自动化看作类似于 Web 浏览器显示 HTML 文件的方式。 HTML 文件描述页面中显示的元素，而不描述相关显示方式。 “方式”由 Web 浏览器负责。

## 资源管理器模板的内容

资源管理器模板可以包含以下部分。 它们使用 JSON 表示法表示，但与 JSON 语言本身无关。

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "",
    "parameters": {  },
    "variables": {  },
    "functions": [  ],
    "resources": [  ],
    "outputs": {  }
}
```

### 参数 Parameter

通过本部分，可以在模板运行时指定哪些值可配置。 例如，可以允许模板用户指定用户名、密码或域名。
下面示例演示了两个参数 – 一个用于 VM 的用户名，另一个用于它的密码。

```json
"parameters": {
  "adminUsername": {
    "type": "string",
    "metadata": {
      "description": "Username for the Virtual Machine."
    }
  },
  "adminPassword": {
    "type": "securestring",
    "metadata": {
      "description": "Password for the Virtual Machine."
    }
  }
}
```

### 函数 Functions

通过本部分，可以定义整个模板中不想重复的过程。 与变量一样，函数也可简化模板的维护。 下面的示例创建了一个函数，以创建可在创建具有全局唯一命名要求的资源时使用的唯一名称。

```json
"functions": [
  {
    "namespace": "contoso",
    "members": {
      "uniqueName": {
        "parameters": [
          {
            "name": "namePrefix",
            "type": "string"
          }
        ],
        "output": {
          "type": "string",
          "value": "[concat(toLower(parameters('namePrefix')), uniqueString(resourceGroup().id))]"
        }
      }
    }
  }
],
```

### 资源 Resource

通过本部分，可以定义组成部署的 Azure 资源。

下面的示例将创建一个公共 IP 地址资源。

```json
"resources": [
{
  "type": "Microsoft.Network/publicIPAddresses",
  "name": "[variables('publicIPAddressName')]",
  "location": "[parameters('location')]",
  "apiVersion": "2018-08-01",
  "properties": {
    "publicIPAllocationMethod": "Dynamic",
    "dnsSettings": {
      "domainNameLabel": "[parameters('dnsLabelPrefix')]"
    }
  }
}
],
```

>由于资源类型可能会随着时间推移而发生变化，因此 apiVersion 指的是要使用的资源类型的版本。 随着资源类型发展变化，你可以修改模板，以便准备就绪后使用最新功能.
>最新apiversion地址：<https://docs.microsoft.com/en-us/azure/templates/microsoft.resources/allversions>

### 输出 Output

通过本部分，可以定义模板运行时想要收到的信息。 例如，你可能想要收到 VM 的 IP 地址或 FQDN – 即部署运行前不知道的信息。

下面的示例演示了名为“主机名”的输出。 从 VM 的公共 IP 地址设置中读取 FQDN 值。

```json
"outputs": {
  "hostname": {
    "type": "string",
    "value": "[reference(variables('publicIPAddressName')).dnsSettings.fqdn]"
  }
}
```

### 使用Azure Template 部署VM

基于上面提到的资源管理器模板所包含的部分，我们可以修改模板相应的参数和值，以便满足我们的需求。

在模板开头附近，将看到名为 parameters 的部分。 本部分将定义以下参数：

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20210423103635.png" width=600 />

* adminUsername
* authenticationType
* adminPasswordOrKey
* dnsLabelPrefix
* ubuntuOSVersion
* location

authenticationType 参数指定是使用密码身份验证还是基于密钥的身份验证来连接到虚拟机。 adminPasswordOrKey 参数指定密码或 SSH 密钥。
某些参数（例如 ubuntuOSVersion 和 location）具有默认值。 ubuntuOSVersion 的默认值是“18.04-LTS”，而 location 的默认值是父资源组的位置。

让我们继续使用这些参数的默认值。 对于剩下的参数，有两个选择：

1. 提供 JSON 文件中的值。
2. 提供值，作为命令行参数。

在CLI中执行如下命令定义相关参数：

```shell
#定义资源组名称
RESOURCEGROUP=MYRG
#定义位置
LOCATION=eastus
#定义用户名
USERNAME=azureuser
#运行openssl，生成一个随机密码。
PASSWORD=$(openssl rand -base64 32)
#生成唯一的DNS标签前缀
DNS_LABEL_PREFIX=mydeployment-$RANDOM

```
