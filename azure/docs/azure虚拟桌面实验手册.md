# 配置和操作 Microsoft Azure 虚拟桌面

## 练习 1：部署 Active Directory 域服务 (AD DS) 域

本练习的主要任务如下：

- [ ] 准备 Azure VM 部署
- [ ] 使用 Azure 资源管理器快速启动模板来部署运行 AD DS 域控制器的 Azure VM
- [ ] 使用 Azure 资源管理器快速启动模板部署运行 Windows 10 的 Azure VM
- [ ] 部署 Azure Bastion

### 任务 1：准备 Azure VM 部署

- [ ] 在实验室计算机上，启动 Web 浏览器，导航到 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在显示 Azure 门户的 Web 浏览器中，导航到 Azure AD 租户的“概述”边栏选项卡，并在左侧垂直菜单的“管理”部分中，单击“属性”。

- [ ] 在 Azure AD 租户的“属性”边栏选项卡的最底部，选择“管理安全默认值”链接。

- [ ] 如有需要，在“启用安全默认值”边栏选项卡上，选择“否”，选中“我的组织正在使用条件访问”复选框，然后选择“保存”。

- [ ] 在 Azure 门户中，通过选择搜索文本框右侧的“工具栏”图标，打开 “Cloud Shell” 窗格。

- [ ] 提示选择 “Bash” 或 “PowerShell” 时，选择 “PowerShell”。

> 备注： 如果这是第一次启动 Cloud Shell，并显示消息 “未装载任何存储”，请选择你将在本实验室中使用的订阅，然后选择 “创建存储”。

### 任务 2：使用 Azure 资源管理器快速启动模板来部署运行 AD DS 域控制器的 Azure VM

- [ ] 在实验室计算机上显示 Azure 门户的 Web 浏览器中，从 Cloud Shell 窗格中的 PowerShell 会话中运行以下命令，以创建资源组（将 <Azure_region> 占位符替换为你打算用于本实验室的 Azure 区域的名称，例如 eastus）：

    ```powershell
    $location = '<Azure_region>'
    $resourceGroupName = 'avd-11-RG'
    New-AzResourceGroup -Location $location -Name $resourceGroupName
    ```

- [ ] 在 Azure 门户中，关闭 “Cloud Shell” 窗格。

- [ ] 在实验室计算机的同一 Web 浏览器窗口中，打开另一个 Web 浏览器选项卡，并浏览名为[创建新的 Windows VM 并创建新的 AD 林、域和 DC](https://github.com/az140mp/azure-quickstart-templates/tree/master/application-workloads/active-directory/active-directory-new-domain) 的快速启动模板的自定义版本。

- [ ] 在 **创建新的 Windows VM 并创建新的 AD 林、域和 DC** 页面上，选择 **部署到 Azure**。这将自动将浏览器重定向到 Azure 门户中的 **使用新的 AD 林创建 Azure VM** 边栏选项卡。

- [ ] 在 “使用新的 AD 林创建 Azure VM” 边栏选项卡中，选择 “编辑参数”。

- [ ] 在 “编辑参数” 边栏选项卡中，复制并替换为如下参数,然后点击保存。

    ```json
    {
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "adminUsername": {
        "value": "avdAdmin"
        },
        "adminPassword": {
        "value": "Pa55w.rd1234"
        },
        "domainName": {
        "value": null
        },
        "vmSize": {
        "value": "Standard_B2s"
        },
        "_artifactsLocation": {
        "value": "[deployment().properties.templateLink.uri]"
        },
        "_artifactsLocationSasToken": {
        "value": null
        },
        "location": {
        "value": "[resourceGroup().location]"
        },
        "virtualMachineName": {
        "value": "avd-dc-vm11"
        },
        "virtualNetworkName": {
        "value": "avd-adds-vnet11"
        },
        "virtualNetworkAddressRange": {
        "value": "10.0.0.0/16"
        },
        "networkInterfaceName": {
        "value": "avd-dc-vm11-nic"
        },
        "privateIPAddress": {
        "value": "10.0.0.4"
        },
        "subnetName": {
        "value": "adds-subnet"
        },
        "subnetRange": {
        "value": "10.0.0.0/24"
        },
        "availabilitySetName": {
        "value": "avd-dc-advset11"
        }
    }
    }
    ```

- [ ] 在 “使用新的 AD 林创建 Azure VM” 边栏选项卡中，指定以下设置（将其他设置保留为默认值）：

    |设置   |值|
    |--|--|
    |订阅|在本实验室中使用的 Azure 订阅的名称|
    |资源组|avd-11-RG|
    |域名|adatum.com|

- [ ] 在“使用新的 AD 林创建 Azure VM”边栏选项卡中，选择“查看 + 创建”，然后选择“创建”。

> 备注：等待部署完成后，再继续下一个练习。该过程大约需要 15 分钟。

### 任务 3：使用 Azure 资源管理器快速启动模板部署运行 Windows 10 的 Azure VM

- [ ] 在显示 Azure 门户的 Web 浏览器中，在“Cloud Shell”窗格中打开 PowerShell 会话，并运行以下命令，将名为 cl-Subnet 的子网添加到在上一个任务中创建的名为 avd-adds-vnet11 的虚拟网络中：

    ```powershell
    $resourceGroupName = 'avd-11-RG'
    $vnet = Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name 'avd-adds-vnet11'
    $subnetConfig = Add-AzVirtualNetworkSubnetConfig `
    -Name 'cl-Subnet' `
    -AddressPrefix 10.0.255.0/24 `
    -VirtualNetwork $vnet
    $vnet | Set-AzVirtualNetwork
    ```


