# Azure Cli 记录

## 定义

Azure CLI 是 Microsoft 提供的用于管理 Azure 资源的跨平台命令行工具。 它适用于 macOS、Linux 和 Windows，或通过 Azure Cloud Shell 在浏览器中使用。

## 安装

> 参考:
><https://docs.azure.cn/zh-cn/cli/install-azure-cli?view=azure-cli-latest>

## 使用

### 登录azure

```AzureCLI
az login
```

>备注：
>请先运行 az cloud set -n AzureChinaCloud 更改云环境，然后才能在 Azure 中国中使用 Azure CLI。若要切换回 Azure 公有云，请再次运行 az cloud set -n AzureCloud。

### 使用 Azure CLI 创建虚拟机

Azure CLI 包含用于在 Azure 中使用虚拟机的 vm 命令。 可提供几个用于执行特定任务的子命令。 最常见的子命令包括：

|子命令|说明|
| --- | --- |
|create|新建虚拟机|
|deallocate|解除分配虚拟机|
|delete|删除虚拟机|
|list|列出订阅中已创建的虚拟机|
|open-port|打开特定于入站流量的网络端口|
|restart|重启虚拟机|
|show|获取虚拟机详细信息|
|start|启动已停止的虚拟机|
|stop|停止正在运行的虚拟机|
|update|更新虚拟机属性|

>详细命令信息参考：<https://docs.microsoft.com/zh-cn/cli/azure/reference-index?view=azure-cli-latest>

我们先从第一个命令 az vm create 开始。 此命令用于在资源组中创建虚拟机。 你可以传递几个参数用于配置新 VM 的所有方面。 必须提供下面四个参数：

|参数|说明|
|---|---|
|--resource-group|指定虚拟机所在的资源组|
|--name|虚拟机名称，它必须在资源组中唯一|
|--image|用于创建VM的操作系统映像|
|--location|要放置VM的区域。|
|--verbose|（可选）有助于在创建VM时查看进度|

```AzureCLI
az vm create \
  --resource-group resourcegroup \
  --location westus \
  --name SampleVM \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --verbose 
```

完成创建 VM 后，将获得 JSON 响应，其中包括虚拟机的当前状态及 Azure 分配的公共和专用 IP 地址：

```JSON
{
  "fqdns": "",
  "id": "/subscriptions/dba7b003-c301-4f1c-89e6-6f6296f5d985/resourceGroups/learn-897badeb-835e-4106-a061-6389ff603aaf/providers/Microsoft.Compute/virtualMachines/SampleVM",
  "location": "westus",
  "macAddress": "00-22-48-06-F6-D1",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "13.93.162.73",
  "resourceGroup": "learn-897badeb-835e-4106-a061-6389ff603aaf",
  "zones": ""
}
```

### 列出映像

通过运行下面命令，获取可用的VM映像列表:

```AzureCLI
az vm image list --output table
```

通过将 --all 标志添加到命令中，可获取完整列表。 由于市场中的映像列表非常大，因此最好使用 --publisher、--sku 或 –-offer 选项来筛选列表。

### 调整或者设置虚拟机大小

可用大小根据VM创建的区域而变，可以使用```vm list-size```命令获取可用大小的列表。

```AzureCLI
az vm list-sizes --location eastus --output table
```

#### 在VM创建过程中指定大小

由于我们在创建 VM 时未指定大小，因此 Azure 为我们选取了默认通用大小。 但是，我们可以通过 --size 参数在使用 vm create 命令时指定大小。 例如，可以使用以下命令创建 2 个核心的虚拟机：

```AzureCLI
az vm create \
  --resource-group resourcegroup \
  --location westus \
  --name SampleVM \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --verbose \
  --size "Standard_DS2_v2"
```

#### 重设现有VM的大小

如果工作负荷发生更改，或在创建时未正确设置 VM 大小，还可重设现有 VM 的大小。 请求重设大小之前，必须检查以查看所需大小是否在 VM 所属的群集中可用。 为此，可以使用 vm list-vm-resize-options 命令：

```AzureCLI
az vm list-vm-resize-options \
  --resource-group resourcegroup \
```

若要重设 VM 大小，请运行 vm resize 命令。 例如，也许会发现虚拟机的性能不足，无法执行所需任务。 可将其提升到 D2s_v3，这时它具有 2 个虚拟核心和 8 GB 的内存

```AzureCLI
az vm resize \
    --resource-group [sandbox resource group name] \
    --name SampleVM \
    --size Standard_D2s_v3
```

### 获取IP地址

另一个有用的命令是 vm list-ip-addresses，它将列出 VM 的公共和专用 IP 地址。

```Azure CLI
az vm list-ip-addresses -n SampleVM -o table
```

### 使用JMESPath将筛选器添加到查询中

MESPath 是一个行业标准的查询语言，它围绕 JSON 对象进行构建。 最简单的查询是指定一个标识符，它在 JSON 对象中选择一个键。

例如，假定对象如下：

```JSON
{
  "people": [
    {
      "name": "Fred",
      "age": 28
    },
    {
      "name": "Barney",
      "age": 25
    },
    {
      "name": "Wilma",
      "age": 27
    }
  ]
}

```

我们可使用 people 查询针对 people 数组返回一组值。 如果我们只需要其中一个人员，则可使用索引程序。 例如，people[1] 将返回如下内容：

```JSON
{
    "name": "Barney",
    "age": 25
}
```

我们还可以添加特定限定符，这些限定符将根据某些条件返回对象的子集。 例如，添加限定符 people[?age > '25'] 将返回如下内容：

```JSON
[
  {
    "name": "Fred",
    "age": 28
  },
  {
    "name": "Wilma",
    "age": 27
  }
]
```

最后，我们可添加只返回名称的 select: people[?age > '25'].[name]，对结果进行限制：

```JSON
[
  [
    "Fred"
  ],
  [
    "Wilma"
  ]
]
```

### 筛选 Azure CLI 查询

在基本了解了 JMES 查询之后，现可向 vm show 命令之类的查询所返回的数据添加筛选器。 例如，我们可检索管理员用户名称：

```AzureCLI
az vm show \
    --resource-group learn-752ef656-acc3-47c8-8f53-bc2404df60a2 \
    --name SampleVM \
    --query "osProfile.adminUsername"
```

可获取分配给 VM 的大小：

```AzureCLI
az vm show \
    --resource-group learn-752ef656-acc3-47c8-8f53-bc2404df60a2 \
    --name SampleVM \
    --query hardwareProfile.vmSize
```

若要检索网络接口的所有 ID，可使用以下查询：

```AzureCLI
az vm show \
    --resource-group learn-752ef656-acc3-47c8-8f53-bc2404df60a2 \
    --name SampleVM \
    --query "networkProfile.networkInterfaces[].id"
```

>此查询技术适用于所有 Azure CLI 命令，并可用于在命令行上拉取出特定位的数据。 它还可用于编写脚本，例如你可从 Azure 帐户中拉取出一个值，并将其存储到环境或脚本变量中。 如果你决定按此方式使用它，可添加一个有用的标记，即 --output tsv 参数（可简写成 -o tsv）。 这将按制表符分隔的值返回结果，其中仅包含带制表分隔符的实际数据值。

### 停止VM

可以使用 vm stop 命令停止正在运行的 VM。 必须传递 VM 的名称和资源组或唯一 ID：

```AzureCLI
az vm stop \
    --name SampleVM \
    --resource-group [sandbox resource group name]
```

可以通过尝试对公共 IP 地址进行 ping 操作、使用 ssh 或通过 vm get-instance-view 命令来验证虚拟机是否已停止。 最后一种方法会返回与 vm show 相同的基本数据，但包含有关该实例本身的详细信息。 尝试将以下命令键入到 Azure Cloud Shell，以查看 VM 的当前运行状态：

```AzureCLI
az vm get-instance-view \
    --name SampleVM \
    --resource-group [sandbox resource group name] \
    --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" -o tsv
```

### 启动VM

可以通过 vm start 命令取得相反结果。

```AzureCLI
az vm start \
    --name SampleVM \
    --resource-group [sandbox resource group name]
```

此命令将启动已停止的 VM。 可以通过 vm get-instance-view 查询验证是否已启动虚拟机，该查询现在应返回 VM running。
