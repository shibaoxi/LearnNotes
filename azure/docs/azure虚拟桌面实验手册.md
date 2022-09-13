# 配置和操作 Microsoft Azure 虚拟桌面

## 准备部署 Azure 虚拟桌面 (AD DS)

### 练习 1：部署 Active Directory 域服务 (AD DS) 域

本练习的主要任务如下：

- [ ] 准备 Azure VM 部署
- [ ] 使用 Azure 资源管理器快速启动模板来部署运行 AD DS 域控制器的 Azure VM
- [ ] 使用 Azure 资源管理器快速启动模板部署运行 Windows 10 的 Azure VM
- [ ] 部署 Azure Bastion

#### 任务 1：准备 Azure VM 部署

- [ ] 在实验室计算机上，启动 Web 浏览器，导航到 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在显示 Azure 门户的 Web 浏览器中，导航到 Azure AD 租户的**概述**边栏选项卡，并在左侧垂直菜单的**管理**部分中，单击**属性**。

- [ ] 在 Azure AD 租户的**属性**边栏选项卡的最底部，选择**管理安全默认值**链接。

- [ ] 如有需要，在**启用安全默认值**边栏选项卡上，选择**否**，选中**我的组织正在使用条件访问**复选框，然后选择**保存**。

- [ ] 在 Azure 门户中，通过选择搜索文本框右侧的**工具栏**图标，打开 **Cloud Shell** 窗格。

- [ ] 提示选择 **Bash** 或 **PowerShell** 时，选择 **PowerShell**。

> 备注： 如果这是第一次启动 Cloud Shell，并显示消息 **未装载任何存储**，请选择你将在本实验室中使用的订阅，然后选择 **创建存储**。

#### 任务 2：使用 Azure 资源管理器快速启动模板来部署运行 AD DS 域控制器的 Azure VM

- [ ] 在实验室计算机上显示 Azure 门户的 Web 浏览器中，从 Cloud Shell 窗格中的 PowerShell 会话中运行以下命令，以创建资源组（将 <Azure_region> 占位符替换为你打算用于本实验室的 Azure 区域的名称，例如 eastus）：

    ```powershell
    $location = '<Azure_region>'
    $resourceGroupName = 'avd-11-RG'
    New-AzResourceGroup -Location $location -Name $resourceGroupName
    ```

- [ ] 在 Azure 门户中，关闭 **Cloud Shell** 窗格。

- [ ] 在实验室计算机的同一 Web 浏览器窗口中，打开另一个 Web 浏览器选项卡，并浏览名为[创建新的 Windows VM 并创建新的 AD 林、域和 DC](https://github.com/az140mp/azure-quickstart-templates/tree/master/application-workloads/active-directory/active-directory-new-domain) 的快速启动模板的自定义版本。

- [ ] 在 **创建新的 Windows VM 并创建新的 AD 林、域和 DC** 页面上，选择 **部署到 Azure**。这将自动将浏览器重定向到 Azure 门户中的 **使用新的 AD 林创建 Azure VM** 边栏选项卡。

- [ ] 在 **使用新的 AD 林创建 Azure VM** 边栏选项卡中，选择 **编辑参数**。

- [ ] 在 **编辑参数** 边栏选项卡中，复制并替换为如下参数,然后点击保存。

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

- [ ] 在 **使用新的 AD 林创建 Azure VM** 边栏选项卡中，指定以下设置（将其他设置保留为默认值）：

    |设置   |值|
    |--|--|
    |订阅|在本实验室中使用的 Azure 订阅的名称|
    |资源组|avd-11-RG|
    |域名|adatum.com|

- [ ] 在**使用新的 AD 林创建 Azure VM**边栏选项卡中，选择**查看 + 创建**，然后选择**创建**。

> 备注：等待部署完成后，再继续下一个练习。该过程大约需要 15 分钟。

#### 任务 3：使用 Azure 资源管理器快速启动模板部署运行 Windows 10 的 Azure VM

- [ ] 在显示 Azure 门户的 Web 浏览器中，在**Cloud Shell**窗格中打开 PowerShell 会话，并运行以下命令，将名为 cl-Subnet 的子网添加到在上一个任务中创建的名为 avd-adds-vnet11 的虚拟网络中：

    ```powershell
    $resourceGroupName = 'avd-11-RG'
    $vnet = Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name 'avd-adds-vnet11'
    $subnetConfig = Add-AzVirtualNetworkSubnetConfig `
    -Name 'cl-Subnet' `
    -AddressPrefix 10.0.255.0/24 `
    -VirtualNetwork $vnet
    $vnet | Set-AzVirtualNetwork
    ```

- [ ] 从**Cloud Shell**窗格中的 PowerShell 会话，运行以下命令将运行 Windows 10 的 Azure VM（该 VM 将用作客户端）部署到新建的子网：

    ```powershell
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeploycl11.json
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeploycl11.parameters.json
    $resourceGroupName = 'avd-11-RG'
    $location = (Get-AzResourceGroup -ResourceGroupName $resourceGroupName).Location
    New-AzResourceGroupDeployment `
    -ResourceGroupName $resourceGroupName `
    -Location $location `
    -Name avdlab0101vmDeployment `
    -TemplateFile $HOME/azuredeploycl11.json `
    -TemplateParameterFile $HOME/azuredeploycl11.parameters.json
    ```

>备注：请勿等待部署完成，而是继续进行下一项任务。部署可能需要大约 10 分钟。

#### 任务 4：部署 Azure Bastion

>备注：通过 Azure Bastion，可以连接到你在本练习的上一个任务中部署的没有公共终结点的 Azure VM，同时提供针对操作系统级别凭据的暴力攻击防护

- [ ] 在显示 Azure 门户的浏览器窗口中，打开另一个选项卡，然后在该浏览器选项卡中导航到 Azure 门户。

- [ ] 在 Azure 门户中，通过选择搜索文本框右侧的**工具栏**图标，打开 Cloud Shell 窗格。

- [ ] 在 Cloud Shell 窗格中的 PowerShell 会话中运行以下命令，将名为 AzureBastionSubnet 的子网添加到你在本练习前面部分创建的名为 avd-adds-vnet11 的虚拟网络中：

    ```powershell
    $resourceGroupName = 'avd-11-RG'
    $vnet = Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name 'avd-adds-vnet11'
    $subnetConfig = Add-AzVirtualNetworkSubnetConfig `
    -Name 'AzureBastionSubnet' `
    -AddressPrefix 10.0.254.0/24 `
    -VirtualNetwork $vnet
    $vnet | Set-AzVirtualNetwork
    ```

- [ ] 关闭 Cloud Shell 窗格。

- [ ] 在 Azure 门户中搜索并选择**Bastion**，然后在**Bastion**边栏选项卡中，选择**+ 创建**。

- [ ] 在**创建 Bastion**边栏选项卡的**基本信息**选项卡中，指定以下设置并选择**查看 + 创建**：

|设置|值|
|--|--|
|订阅|在本实验室中使用的 Azure 订阅的名称|
|资源组|avd-11-RG|
|名称|avd-11-bastion|
|区域|在本练习前面的任务中向其中部署了资源的同一 Azure 区域|
|层级|基本
|虚拟网络|avd-adds-vnet11
|子网|AzureBastionSubnet (10.0.254.0/24)
|公共 IP 地址|新建
|公共 IP 名称|avd-adds-vnet11-ip

- [ ] 在**创建 Bastion**边栏选项卡的**查看 + 创建**选项卡中，选择**创建**：

> 备注：等待部署完成后，再继续下一个练习。部署可能需要大约 5 分钟。

### 练习 2：将 AD DS 林与 Azure AD 租户集成

本练习的主要任务如下：

- [ ] 创建将同步到 Azure AD 的 AD DS 用户和组
- [ ] 配置 AD DS UPN 后缀
- [ ] 创建将用于配置与 Azure AD 同步的 Azure AD 用户
- [ ] 安装 Azure AD Connect
- [ ] 配置混合 Azure AD 联接

#### 任务 1：创建将同步到 Azure AD 的 AD DS 用户和组

- [ ] 在显示 Azure 门户的 Web 浏览器中，搜索并选择 **虚拟机**，然后在 **虚拟机** 边栏选项卡中选择 **avd-dc-vm11**。

- [ ] 在**avd-dc-vm11**边栏选项卡上，选择**连接**，在下拉菜单中选择**Bastion**，在**avd-dc-vm11 | 连接**边栏选项卡的**Bastion**选项卡上，选择**使用 Bastion**。

- [ ] 系统出现提示时，请提供以下凭据并选择**连接**：

    |设置|值
    |--|--|
    |用户名|avdAdmin
    |密码|Pa55w.rd1234

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中，运行以下命令禁用针对管理员的 Internet Explorer 增强安全性：

    ```powershell
    $adminRegEntry = 'HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}'
    Set-ItemProperty -Path $AdminRegEntry -Name 'IsInstalled' -Value 0
    Stop-Process -Name Explorer
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中，运行以下命令以创建一个 AD DS 组织单位，该组织单位将包含同步到本实验室中使用的 Azure AD 租户的范围内的对象：

    ```powershell
    New-ADOrganizationalUnit 'ToSync' -path 'DC=adatum,DC=com' -ProtectedFromAccidentalDeletion $false
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中，运行以下命令以创建一个 AD DS 组织单位，该组织单位将包含已加入域的 Windows 10 客户端计算机的计算机对象：

    ```powershell
    New-ADOrganizationalUnit 'WVDClients' -path 'DC=adatum,DC=com' -ProtectedFromAccidentalDeletion $false
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中，运行以下命令创建 AD DS 用户帐户，这些帐户将同步到本实验室中使用的 Azure AD 租户（将 \<password > 占位符替换为随机的复杂密码）

    ```powershell
    $ouName = 'ToSync'
    $ouPath = "OU=$ouName,DC=adatum,DC=com"
    $adUserNamePrefix = 'aduser'
    $adUPNSuffix = 'adatum.com'
    $userCount = 1..9
    foreach ($counter in $userCount) {
    New-AdUser -Name $adUserNamePrefix$counter -Path $ouPath -Enabled $True `
    -ChangePasswordAtLogon $false -userPrincipalName $adUserNamePrefix$counter@$adUPNSuffix `
    -AccountPassword (ConvertTo-SecureString <password> -AsPlainText -Force) -passThru
    } 

    $adUserNamePrefix = 'wvdadmin1'
    $adUPNSuffix = 'adatum.com'
    New-AdUser -Name $adUserNamePrefix -Path $ouPath -Enabled $True `
    -ChangePasswordAtLogon $false -userPrincipalName $adUserNamePrefix@$adUPNSuffix `
    -AccountPassword (ConvertTo-SecureString <password> -AsPlainText -Force) -passThru

    Get-ADGroup -Identity 'Domain Admins' | Add-AdGroupMember -Members 'wvdadmin1'
    ```

    >备注：该脚本创建了九个名为 aduser1 - aduser9 的非特权用户帐户和一个名为 wvdadmin1 的 ADATUM\Domain Admins 组成员的特权帐户。

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中，运行以下命令以创建将同步到本实验室中使用的 Azure AD 租户的 AD DS 组对象：

    ```powershell
    New-ADGroup -Name 'wvd-pooled' -GroupScope 'Global' -GroupCategory Security -Path $ouPath
    New-ADGroup -Name 'wvd-remote-app' -GroupScope 'Global' -GroupCategory Security -Path $ouPath
    New-ADGroup -Name 'wvd-personal' -GroupScope 'Global' -GroupCategory Security -Path $ouPath
    New-ADGroup -Name 'wvd-users' -GroupScope 'Global' -GroupCategory Security -Path $ouPath
    New-ADGroup -Name 'wvd-admins' -GroupScope 'Global' -GroupCategory Security -Path $ouPath
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以向你在前一个步骤中创建的组添加成员：

    ```powershell
    Get-ADGroup -Identity 'wvd-pooled' | Add-AdGroupMember -Members 'aduser1','aduser2','aduser3','aduser4'
    Get-ADGroup -Identity 'wvd-remote-app' | Add-AdGroupMember -Members 'aduser1','aduser5','aduser6'
    Get-ADGroup -Identity 'wvd-personal' | Add-AdGroupMember -Members 'aduser7','aduser8','aduser9'
    Get-ADGroup -Identity 'wvd-users' | Add-AdGroupMember -Members 'aduser1','aduser2','aduser3','aduser4','aduser5','aduser6','aduser7','aduser8','aduser9'
    Get-ADGroup -Identity 'wvd-admins' | Add-AdGroupMember -Members 'wvdadmin1'
    ```

#### 任务 2：配置 AD DS UPN 后缀

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 **管理员: Windows PowerShell ISE** 脚本窗格运行以下脚本，以安装最新版本的 PowerShellGet 模块（如果提示确认，请选择 **是**）：

    ```powershell
    Install-Module -Name PowerShellGet -Force -SkipPublisherCheck
    Install-Module -Name Az -AllowClobber -SkipPublisherCheck
    Connect-AzAccount
    ```

- [ ] 如果出现提示，请提供具有本实验室所用订阅的所有者角色的用户帐户的凭据。

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以检索与 Azure 订阅关联的 Azure AD 租户的 ID 属性：

    ```powershell
    $tenantId = (Get-AzContext).Tenant.Id
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以安装最新版本的 Azure AD PowerShell 模块：

    ```powershell
    Install-Module -Name AzureAD -Force
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以对 Azure AD 租户进行身份验证：

    ```powershell
    Connect-AzureAD -TenantId $tenantId
    ```

- [ ] 如果出现提示，请使用之前在此任务中使用的相同凭据登录。

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以检索与 Azure 订阅关联的 Azure AD 租户的主 DNS 域名：

    ```powershell
    $aadDomainName = ((Get-AzureAdTenantDetail).VerifiedDomains)[0].Name
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以将与 Azure 订阅关联的 Azure AD 租户的主 DNS 域名添加到 AD DS 林的 UPN 后缀列表中：

    ```powershell
    Get-ADForest|Set-ADForest -UPNSuffixes @{add="$aadDomainName"}
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中运行以下命令，以将与 Azure 订阅关联的 Azure AD 租户的主 DNS 域名分配为 AD DS 域中所有用户的 UPN 后缀：

    ```powershell
    $domainUsers = Get-ADUser -Filter {UserPrincipalName -like '*adatum.com'} -Properties userPrincipalName -ResultSetSize $null
    $domainUsers | foreach {$newUpn = $_.UserPrincipalName.Replace('adatum.com',$aadDomainName); $_ | Set-ADUser -UserPrincipalName $newUpn}
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 控制台中运行以下命令，以将 adatum.com UPN 后缀分配给avdAdmin域用户：

    ```powershell
    $domainAdminUser = Get-ADUser -Filter {sAMAccountName -eq 'avdAdmin'} -Properties userPrincipalName
    $domainAdminUser | Set-ADUser -UserPrincipalName 'avdAdmin@adatum.com'
    ```

#### 任务 3：创建将用于配置目录同步的 Azure AD 用户

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 **管理员: Windows PowerShell ISE** 脚本窗格中运行以下命令，以创建新的 Azure AD 用户（将 \<password> 占位符替换为随机的复杂密码）：

    ```powershell
    $userName = 'aadsyncuser'
    $passwordProfile = New-Object -TypeName Microsoft.Open.AzureAD.Model.PasswordProfile
    $passwordProfile.Password = '<password>'
    $passwordProfile.ForceChangePasswordNextLogin = $false
    New-AzureADUser -AccountEnabled $true -DisplayName $userName -PasswordProfile $passwordProfile -MailNickName $userName -UserPrincipalName "$userName@$aadDomainName"
    ```

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中运行以下命令，以将全局管理员角色分配给新创建的 Azure AD 用户：

    ```powershell
    $aadUser = Get-AzureADUser -ObjectId "$userName@$aadDomainName"
    $aadRole = Get-AzureADDirectoryRole | Where-Object {$_.displayName -eq 'Global administrator'} 
    Add-AzureADDirectoryRoleMember -ObjectId $aadRole.ObjectId -RefObjectId $aadUser.ObjectId
    ```

    >备注： Azure AD PowerShell 模块将全局管理员角色称为公司管理员。

- [ ] 从 **管理员: Windows PowerShell ISE** 脚本窗格中运行以下命令，以标识新创建的 Azure AD 用户的用户主体名称：

    ```powershell
    (Get-AzureADUser -Filter "MailNickName eq '$userName'").UserPrincipalName
    ```

    >备注： 记录用户主体名称。你在本练习后面需要用到它。

#### 任务 4：安装 Azure AD Connect

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从**管理员:Windows PowerShell ISE**脚本窗格中运行以下代码，以启用 TLS 1.2：

    ```powershell
    New-Item 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -name 'SystemDefaultTlsVersions' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -name 'SchUseStrongCrypto' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-Item 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -name 'SystemDefaultTlsVersions' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -name 'SchUseStrongCrypto' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -name 'Enabled' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -name 'DisabledByDefault' -value 0 -PropertyType 'DWord' -Force | Out-Null
    New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client' -name 'Enabled' -value '1' -PropertyType 'DWord' -Force | Out-Null
    New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client' -name 'DisabledByDefault' -value 0 -PropertyType 'DWord' -Force | Out-Null
    Write-Host 'TLS 1.2 has been enabled.'
    ```

- [ ] 在 Azure 门户中，使用 Azure 门户页顶部的 **搜索资源、服务和文档** 文本框，搜索并导航到 **Azure Active Directory** 边栏选项卡，在**Azure AD 租户**选项卡上，在中心菜单的 **管理** 部分，选择 **Azure AD Connect**。

- [ ] 在 **Azure AD Connect** 边栏选项卡上，选择 **下载 Azure AD Connect** 链接。这将自动打开新的浏览器标签页，其中显示了 **Microsoft Azure Active Directory Connect** 下载页。

- [ ] 在 **Microsoft Azure Active Directory Connect** 下载页，选择 **下载**。

- [ ] 把下载的程序复制到avd-dc-vm11中

- [ ] 双击 “运行” 以启动 Microsoft Azure Active Directory Connect 向导。

- [ ] 在 “Microsoft Azure Active Directory Connect” 向导的 “欢迎使用 Azure AD Connect” 页面中，选中复选框 “我同意许可条款和隐私声明”，然后选择 “继续”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 向导的 “快速设置” 页面中，选择 “自定义” 选项。

- [ ] 在 “安装所需组件” 页面中，取消选择所有可选配置选项并选择 “安装”。

- [ ] 在 “用户登录” 页面中，确保仅启用 “密码哈希同步”，然后选择 “下一步”。

- [ ] 在 “连接到 Azure AD” 页面中，使用你在上一个练习中创建的 “aadsyncuser” 用户帐户凭据进行身份验证，然后选择 “下一步”。

    >提供在本练习前面记录的 aadsyncuser 帐户的 userPrincipalName 属性，并指定你之前在本实验室中设置的密码作为其密码。

- [ ] 在 “连接目录” 页面上，选择 “adatum.com” 林条目右边的 “添加目录” 按钮。

- [ ] 在 “AD 林帐户” 窗口中，确保选择 “新建 AD 帐户” 选项，指定以下凭据，然后选择 “确定”：

    |设置|值|
    |--|--|
    |用户名|ADATUM\avdAdmin
    |密码|Pa55w.rd1234

- [ ] 返回 “连接目录” 页面，确保 “adatum.com” 条目显示为已配置的目录，然后选择 “下一步”

- [ ] 在 “Azure AD 登录配置” 页面中，看到警告说明 “如果 UPN 后缀与已验证域名不匹配，用户将无法使用本地凭据登录 Azure AD”，启用复选框 “继续，不将所有 UPN 后缀与已验证域匹配”，然后选择 “下一步”。

    >备注： 这是意料之中的，因为 Azure AD 租户没有与 adatum.com AD DS 的其中一个 UPN 后缀匹配的经过验证的自定义 DNS 域。

- [ ] 在 “域和 OU 筛选” 页面上，在选择选项 “同步选定的域和 OU”，展开 adatum.com 节点，清除所有复选框，仅选中 “ToSync” OU 附近的复选框，然后选择 “下一步”。

- [ ] 在 “唯一标识你的用户” 页面中，接受默认设置，然后选择 “下一步”。

- [ ] 在 “筛选用户和设备” 页面中接受默认设置，然后选择 “下一步”。

- [ ] 在 “可选功能” 页面中接受默认设置，然后选择 “下一步”。

- [ ] 在 “准备配置” 页面在确保选中 “配置完成后启动同步过程” 复选框，然后选择 “安装”。

    >备注： 安装需要约 2 分钟。

- [ ] 请在 “配置完成” 页面中查看此信息，然后选择 “退出” 关闭 “Microsoft Azure Active Directory Connect” 窗口。

- [ ] 在显示 Azure 门户的窗口中，导航到 Adatum 实验室 Azure AD 租户的 “用户”-“所有用户” 边栏选项卡。

- [ ] 在 “用户 | 所有用户” 边栏选项卡中，请注意，用户对象列表包括在本实验室前面创建的 AD DS 用户帐户列表，其中 “是” 条目显示在 “同步的目录” 列中。

    >备注： 可能需要等待几分钟，然后刷新浏览器页面，AD DS 用户帐户才会出现。

## 使用 Azure 门户部署主机池和会话主机 (AD DS)

实验室场景
你需要在 Active Directory 域服务 (AD DS) 环境中创建和配置主机池和会话主机。

目标
完成本实验室后，你将能够：

- 在 AD DS 域中实现 Azure 虚拟桌面环境
- 在 AD DS 域中验证 Azure 虚拟桌面环境

### 练习 1：在 AD DS 域中实现 Azure 虚拟桌面环境

本练习的主要任务如下：

- [ ] 准备 AD DS 域和 Azure 订阅，以部署 Azure 虚拟桌面主机池
- [ ] 部署 Azure 虚拟桌面主机池
- [ ] 管理 Azure 虚拟桌面主机池会话主机
- [ ] 配置 Azure 虚拟桌面应用程序组
- [ ] 配置 Azure 虚拟桌面工作区

#### 任务 1：准备 AD DS 域和 Azure 订阅，以部署 Azure 虚拟桌面主机池

- [ ] 启动 Web 浏览器，导航到 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在 Azure 门户中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡中，选择 “avd-dc-vm11”。

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    |设置|值|
    |--|--|
    |用户名|avdAdmin|
    |密码|Pa55w.rd1234|

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台运行以下命令，创建一个组织单位，该组织单位将托管 Azure 虚拟桌面主机的计算机对象：

    ```powershell
    New-ADOrganizationalUnit 'WVDInfra' –path 'DC=adatum,DC=com' -ProtectedFromAccidentalDeletion $false
    ```

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台运行以下命令，以登录 Azure 订阅：

    ```powershell
    Connect-AzAccount
    ```

- [ ] 如果出现提示，请提供具有本实验室所用订阅的所有者角色的用户帐户的凭据。

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令，以标识 aduser1 帐户的用户主体名称：

    ```powershell
    (Get-AzADUser -DisplayName 'aduser1').UserPrincipalName
    ```

    >备注： 请记录你在此步骤中标识的用户主体名称。稍后将在本实验室用到它。

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台运行以下命令，以注册 Microsoft.DesktopVirtualization 资源提供程序：

    ```powershell
    Register-AzResourceProvider -ProviderNamespace Microsoft.DesktopVirtualization
    ```

- [ ] 启动 Microsoft Edge，导航到 Azure 门户。如果出现提示，请使用在本实验室使用的订阅中具有所有者角色的用户帐户的 Azure AD 凭据登录。

- [ ] 在 Azure 门户中，使用 Azure 门户页顶部的 “搜索资源、服务和文档” 文本框，搜索并导航到 “虚拟网络”，然后在 “虚拟网络” 边栏选项卡中选择 avd-adds-vnet11”。

- [ ] 在 avd-adds-vnet11” 边栏选项卡上，选择 “子网”，在 “子网” 边栏选项卡中选择 “+ 子网”，在 “添加子网” 边栏选项卡上，指定以下设置（所有其他设置保留默认值），并单击 “保存”：

|设置        |值
|:--         |:--
|名称        |hp1-Subnet
|子网地址范围 |10.0.1.0/24

#### 任务 2：部署 Azure 虚拟桌面主机池

- [ ] 在显示 Azure 门户的 Microsoft Edge 窗口，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “主机池”，在 “Azure 虚拟桌面 | 主机池” 边栏选项卡中，选择 “+ 添加”。
- [ ] 在“创建主机池”边栏选项卡的“基本信息”选项卡中，指定以下设置并选择“下一步: 虚拟机 >”（其他设置保留默认值）：

    | 设置 | 值 |
    |:-|:-|
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | 新资源组名称 avd-21-RG |
    | 主机池名称 | avd-21-hp1 |
    | 负载均衡算法 | 广度优先 |
    | 最大会话限制 | 50 |
    | 位置 | 在本实验室的第一个练习中将资源部署到其中的 Azure 区域的名称，或一个靠近该区域的区域的名称 |
    | 验证环境 | 否 |
    | 主机池类型 | 共用 |

- [ ] 在“创建主机池”边栏选项卡的“虚拟机”选项卡中，指定以下设置并选择“下一步: 工作区 >”（其他设置保留默认值）：

    |设置|值|
    |:--|:--|
    |添加虚拟机|是
    |资源组|默认与主机池相同
    |名称前缀|avd-21-p1
    |虚拟机位置|在本实验室的第一个练习中，将资源部署到其中的 Azure 区域的名称
    |可用性选项|不需要基础结构冗余
    |映像类型|库
    |映像|Windows 10 企业版多会话版本 20H2 + Microsoft 365 应用
    |虚拟机大小|标准 B2s
    |VM 数|2
    |OS 磁盘类型|标准 SSD
    |虚拟网络|avd-adds-vnet11
    |子网|hp1-Subnet (10.0.1.0/24)
    |网络安全组|基本
    |公共入站端口|否
    |选择要加入的目录|Active Directory
    |AD 域加入 UPN|avdAdmin@adatum.com
    |密码|Pa55w.rd1234
    |指定域或单元|是
    |要加入的域|adatum.com
    |组织单位路径|OU=WVDInfra,DC=adatum,DC=com
    |用户名|avdAdmin
    |密码|Pa55w.rd1234
    |确认密码|Pa55w.rd1234

- [ ] 在 “创建主机池” 边栏选项卡的 “工作区” 选项卡中，指定以下设置并选择 “查看 + 创建”：

    设置|值
    |:--|:--|
    注册桌面应用组|否

- [ ] 在 “创建主机池” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”。

    >备注：请等待部署完成。这需要约 10 分钟时间。

#### 任务 3：管理 Azure 虚拟桌面主机池会话主机

- [ ] 在显示 Azure 门户的 Web 浏览器窗口中，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，在垂直菜单栏的 “管理部分” 中，选择 “主机池”。
- [ ] 在 “Azure 虚拟桌面 | 主机池” 边栏选项卡的主机池列表中，选择 avd-21-hp1”。
- [ ] 在 avd-21-hp1” 边栏选项卡上，在垂直菜单栏的 “管理部分” 中，选择 “会话主机”，然后验证池是否由两个主机组成。
- [ ] 在 “avd-21-hp1 | 会话主机” 边栏选项卡上，选择 “+ 添加”。
- [ ] 在 “将虚拟机添加到主机池” 边栏选项卡的 “基本信息” 选项卡中，查看预配置的设置并选择 “下一步: 虚拟机”。
- [ ] 在 “将虚拟机添加到主机池” 边栏选项卡的 “虚拟机” 选项卡中，指定以下设置并选择 “查看 + 创建” （其他设置保留默认值）：

    |设置|值
    |:--|:--|
    |资源组|avd-21-RG
    |名称前缀|avd-21-p1
    |虚拟机位置|将前两个会话主机 VM 部署到其中的 Azure 区域的名称
    |可用性选项|不需要基础结构冗余
    |映像类型|库
    |映像|Windows 10 企业版多会话版本 2004 + Microsoft 365 应用
    |VM 数|1
    |虚拟网络|avd-adds-vnet11
    |子网|hp1-Subnet (10.0.1.0/24)
    |配置 SKU|基本
    |配置分配|动态
    |网络安全组|基本
    |公共入站端口|否
    |AD 域加入 UPN|avdAdmin@adatum.com
    |密码|Pa55w.rd1234
    |指定域或单元|是
    |要加入的域|adatum.com
    |组织单位路径|OU=WVDInfra,DC=adatum,DC=com
    |虚拟机管理员帐户用户名|avdAdmin
    |虚拟机管理员帐户密码|Pa55w.rd1234

    >备注： 正如你可能注意到的，在将会话主机添加到现有池时，可以更改 VM 的映像和前缀。通常，除非计划替换池中的所有 VM，否则不建议这样做。
- [ ] 在 “将虚拟机添加到主机池” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”
    >备注：等待部署完成后，再继续执行下一个任务。该部署可能需要大约 5 分钟。

#### 任务 4：配置 Azure 虚拟桌面应用程序组

- [ ] 在显示 Azure 门户的 Web 浏览器窗口中，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “应用程序组”。
- [ ] 在 “Azure 虚拟桌面 | 应用程序组” 边栏选项卡上，注意现有的自动生成的 avd-21-hp1-DAG 桌面应用程序组，并选择它。
- [ ] 在 “avd-21-hp1-DAG” 边栏选项卡中，选择 “分配”。
- [ ] 在 “avd-21-hp1-DAG | 分配” 边栏选项卡上，选择 “+ 添加”。
- [ ] 在 “选择 Azure AD 用户或用户组” 边栏选项卡上，选择 “avd-wvd-pooled”，然后单击 “选择”。
- [ ] 导航回到 “Azure 虚拟桌面 | 应用程序组” 边栏选项卡上，选择 “+ 添加”。
- [ ] 在 “创建应用程序组” 边栏选项卡的 “基本信息” 选项卡中，指定以下设置并选择 “下一步: 应用程序 >”：

    |设置|值
    |:--|:--|
    |订阅|在本实验室中使用的 Azure 订阅的名称
    |资源组|avd-21-RG
    |主机池|avd-21-hp1
    |应用程序组类型|RemoteApp
    |应用程序组名称|avd-21-hp1-Office365-RAG

- [ ] 在 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “+ 添加应用程序”。
- [ ] 在 “添加应用程序” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|值
    |:--|:--|
    |应用程序源|“开始” 菜单
    |应用程序|Word
    |说明|Microsoft Word
    |需要命令行|否

- [ ] 返回 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “+ 添加应用程序”。
- [ ] 在 “添加应用程序” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|值
    |:--|:--|
    |应用程序源|“开始” 菜单
    |应用程序|Excel
    |说明|Microsoft Excel
    |需要命令行|否
- [ ] 返回 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “+ 添加应用程序”。
- [ ] 在 “添加应用程序” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|值
    |:--|:--|
    |应用程序源|“开始” 菜单
    |应用程序|PowerPoint
    |说明|Microsoft PowerPoint
    |需要命令行|否

- [ ] 返回 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “下一步: 分配 >”。
- [ ] 在 “创建应用程序组” 边栏选项卡的 “分配” 选项卡中，选择 “+ 添加 Azure AD 用户或用户组”。
- [ ] 在 “选择 Azure AD 用户或用户组” 边栏选项卡上，选择 “avd-wvd-remote-app”，然后单击 “选择”。
- [ ] 返回 “创建应用程序组” 边栏选项卡的 “分配” 选项卡中，选择 “下一步:工作区 >”。
- [ ] 在 “创建工作区” 边栏选项卡的 “工作区” 选项卡中，指定以下设置并选择 “查看 + 创建”：

    |设置|值
    |:--|:--|
    |注册应用程序组|否

- [ ] 在 “创建应用程序组” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”。

    >备注： 等待应用程序组创建完成。所需时间应该不超过一分钟。
    >备注： 接下来，你将基于作为应用程序源的文件路径创建一个应用程序组。

- [ ] 搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “应用程序组”。

- [ ] 在 “Azure 虚拟桌面 | 应用程序组” 边栏选项卡上，选择 “+ 添加”。

- [ ] 在 “创建应用程序组” 边栏选项卡的 “基本信息” 选项卡中，指定以下设置并选择 “下一步: 应用程序 >”：

    |设置|值
    |:--|:--|
    |订阅|在本实验室中使用的 Azure 订阅的名称
    |资源组|avd-21-RG
    |主机池|avd-21-hp1
    |应用程序组类型|RemoteApp
    |应用程序组名称|avd-21-hp1-Utilities-RAG

- [ ] 在 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “+ 添加应用程序”。

- [ ] 在 “创建应用程序组” 边栏选项卡的 “应用程在 “添加应用程序” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|值
    |:--|:--|
    |应用程序源|文件路径
    |应用程序路径|C:\Windows\system32\cmd.exe
    |应用程序名称|命令提示符
    |显示名称|命令提示符
    |图标路径|C:\Windows\system32\cmd.exe
    |图标索引|0
    |说明|Windows 命令提示符
    |需要命令行|否

- [ ] 返回 “创建应用程序组” 边栏选项卡的 “应用程序” 选项卡中，选择 “下一步: 分配 >”。

- [ ] 在 “创建应用程序组” 边栏选项卡的 “分配” 选项卡中，选择 “+ 添加 Azure AD 用户或用户组”。

- [ ] 在 “选择 Azure AD 用户或用户组” 边栏选项卡上，选择 “wvd-remote-app” 和 “wvd-admins”，然后单击 “选择”。

- [ ] 返回 “创建应用程序组” 边栏选项卡的 “分配” 选项卡中，选择 “下一步: 工作区 >”。

- [ ] 在 “创建工作区” 边栏选项卡的 “工作区” 选项卡中，指定以下设置并选择 “查看 + 创建”：

    |设置|值
    |:--|:--|
    |注册应用程序组|否

- [ ] 在 “创建应用程序组” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”。

#### 任务 5：配置 Azure 虚拟桌面工作区

- [ ] 在显示 Azure 门户的 Microsoft Edge 窗口中，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “工作区”。

- [ ] 在 “Azure 虚拟桌面 | 工作区” 边栏选项卡上，选择 “+ 添加”。

- [ ] 在 “创建工作区” 边栏选项卡的 “基本信息” 选项卡中，指定以下设置并选择 “下一步: 应用程序组 >”：

    |设置|值
    |:--|:--|
    |订阅|在本实验室中使用的 Azure 订阅的名称
    |资源组|avd-21-RG
    |工作区名称|avd-21-ws1
    |易记名称|avd-21-ws1
    |位置|在本实验室的第一个练习中将资源部署到其中的 Azure 区域的名称，或一个靠近该区域的区域的名称

- [ ] 在 “创建工作区” 边栏选项卡的 “应用程序组” 选项卡上，指定以下设置：

    |设置|值
    |:--|:--|
    |注册应用程序组|是

- [ ] 在 “创建工作区” 边栏选项卡的 “工作区” 选项卡中，选择 “+ 注册应用程序组”。

- [ ] 在 “添加应用程序组” 边栏选项卡上，选择 “avd-21-hp1-DAG”、 “avd-21-hp1-Office365-RAG” 和 “avd-21-hp1-Utilities-RAG” 条目旁的加号，然后单击 “选择”。

- [ ] 返回到 “创建工作区” 边栏选项卡中的 “应用程序组” 选项卡，选择 “查看 + 创建”。

- [ ] 在 “创建工作区” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”。

### 练习 2：验证 Azure 虚拟桌面环境

本练习的主要任务如下：

- [ ] 在 Windows 10 计算机上安装 Microsoft 远程桌面客户端 (MSRDC)
- [ ] 订阅 Azure 虚拟桌面工作区
- [ ] 测试 Azure 虚拟桌面应用

#### 任务 1：在 Windows 10 计算机上安装 Microsoft 远程桌面客户端 (MSRDC)

- [ ] 在显示 Azure 门户的浏览器窗口中，搜索并选择 “虚拟机”，并在 “虚拟机” 边栏选项卡上，选择 “avd-cl-vm11” 条目。

- [ ] 在 “avd-cl-vm11” 边栏选项卡上，向下滚动到 “操作” 部分，然后选择 “运行命令”。

- [ ] 在 “avd-cl-vm11 | 运行命令” 边栏选项卡上，选择 “EnableRemotePS”，然后选择 “运行”。

    >备注： 请等待命令运行完成，再继续下一步。这大约需要 1 分钟。

- [ ] 在与 avd-dc-vm11 远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，将 ADATUM\wvd-users 的所有成员添加到本地 “远程桌面用户” 组，该组位于运行你在实验室 “准备部署 Azure 虚拟桌面 (AD DS)” 中部署的 Windows 10 的 Azure VM avd-cl-vm11 上。

    ```powershell
    $computerName = 'avd-cl-vm11'
    Invoke-Command -ComputerName $computerName -ScriptBlock {Add-LocalGroupMember -Group 'Remote Desktop Users' -Member 'ADATUM\wvd-users'}
    ```

- [ ] 在计算机的显示 Azure 门户的浏览器窗口中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡上，选择 “avd-cl-vm11” 条目。

- [ ] 在“avd-cl-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-cl-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    |设置|值
    |:--|:--|
    |用户名|avdAdmin@adatum.com
    |密码|Pa55w.rd1234

- [ ] 在与 avd-cl-vm11 的远程桌面会话中，启动 Microsoft Edge，并导航到 [Windows 桌面客户端下载页](https://docs.microsoft.com/en-us/windows-server/remote/remote-desktop-services/clients/windowsdesktop#install-the-client)，然后在出现提示时，选择 “运行” 以启动安装。在 “远程桌面安装” 向导的 “安装范围” 页上，选择 “为此计算机的所有用户安装” 选项，然后单击 “安装”。当用户帐户控制提示输入管理凭据时，请使用 ADATUM\Student 用户名并将 Pa55w.rd1234 作为密码进行身份验证。

- [ ] 安装完成后，请确保 “安装完成时启动远程桌面” 复选框处于选中状态，然后单击 “完成” 以启动“远程桌面客户端”。

#### 任务 2：订阅 Azure 虚拟桌面工作区

- [ ] 在“远程桌面”客户端窗口中，选择“订阅”，并在出现提示时使用 aduser1 凭据登录（通过提供你之前在本实验室中指定的 userPrincipalName，并使用你创建该帐户时设置的密码）。

    >备注： 或者，在“远程桌面”客户端窗口中，选择“通过 URL 订阅”，在“订阅工作区”窗格中的“电子邮件或工作区 URL”中，键入 “<https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery>” ，选择“下一步”，然后在出现提示时，使用 aduser1 凭据（使用其 userPrincipalName 属性作为用户名，并使用你创建帐户时设置的密码）进行登录。

- [ ] 在 “保持登录到所有应用” 窗口中，清除 “允许组织管理我的设备” 复选框，然后选择 “否，仅登录到此应用”。

- [ ] 确保 “远程桌面” 页显示发布到工作区的应用程序组中包含的且通过其组成员资格与用户帐户 aduser1 关联的应用程序列表。

#### 任务 3：测试 Azure 虚拟桌面应用

- [ ] 在与 avd-cl-vm11 的远程桌面会话中，在“远程桌面”客户端窗口的应用程序列表中，双击“命令提示符”，然后验证其是否启动了“命令提示符”窗口。提示进行身份验证时，输入你创建 aduser1 用户帐户时设置的密码，选中“记住我”复选框，然后选择“确定”。

    >备注：最开始应用程序可能需要几分钟才能启动，但之后启动速度应该会快得多。

- [ ] 在命令提示符下，键入主机名并按 Enter 键以显示运行命令提示符的计算机的名称。

- [ ] 备注： 验证显示的名称为 avd-21-p1-0、 avd-21-p1-1 或 avd-21-p1-2，而不是 avd-cl-vm11。

- [ ] 在命令提示符下，键入 “logoff” 并按 Enter 键以从当前远程应用会话中注销。

- [ ] 在与 avd-cl-vm11 的远程桌面会话中，在 “远程桌面” 客户端窗口的应用程序列表中，双击 “SessionDesktop”，然后验证其是否启动了远程桌面会话。

- [ ] 在 “默认桌面” 会话中，右键单击 “开始”，选择 “运行”，在 “运行” 对话框的 “打开” 文本框中，键入 “cmd” 并选择 “确定”。

- [ ] 在 “默认桌面” 会话的命令提示符下，键入 “主机名” 并按 Enter 键以显示运行远程桌面会话的计算机的名称。

- [ ] 验证显示的名称是否为 avd-21-p1-0、 avd-21-p1-1 或 avd-21-p1-2。

## 实现和管理 AVD 的存储 (AD DS)

实验室场景

- 你需要在 Azure Active Directory 域服务 (Azure AD DS) 环境中实现和管理 Azure 虚拟桌面部署的存储。

目标

完成本实验室后，你将能够：

- 配置 Azure 文件存储，以存储 Azure 虚拟桌面的配置文件容器

### 练习 1：配置 Azure 文件存储，以存储 Azure 虚拟桌面的配置文件容器

本练习的主要任务如下：

- [ ] 创建 Azure 存储帐户
- [ ] 创建 Azure 文件存储共享
- [ ] 为 Azure 存储帐户启用 AD DS 身份验证
- [ ] 配置 Azure 文件存储基于 RBAC 的权限
- [ ] 配置 Azure 文件存储文件系统权限

#### 任务 1：创建 Azure 存储帐户

- [ ] 启动 Web 浏览器，导航至 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在 Azure 门户中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡中，选择“avd-dc-vm11”。

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    |设置|值
    |:--|:--|
    |用户名|Student@adatum.com
    |密码|Pa55w.rd1234

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，启动 Microsoft Edge，并导航到 Azure 门户。如果出现提示，请使用在本实验室使用的订阅中具有所有者角色的用户帐户的 Azure AD 凭据登录。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在显示 Azure 门户的 Microsoft Edge 窗口中搜索并选择 “存储帐户”，并在 “存储帐户” 边栏选项卡上选择 “+ 添加”。

- [ ] 在 “创建存储帐户” 边栏选项卡的 “基本设置” 选项卡上，指定以下设置（其他设置则保留为默认值）：

    |设置|值
    |:--|:--|
    |订阅|在本实验室中使用的 Azure 订阅的名称
    |资源组|新资源组名称 avd-22-RG
    |存储帐户名称|由小写字母和数字组成、以字母开头、长度介于 3 到 15 之间的全局唯一名称
    |区域|托管 Azure 虚拟桌面实验室环境的 Azure 区域的名称
    |性能|标准
    |冗余|异地冗余存储 (GRS)
    |在区域不可用的情况下，提供对数据的读取访问|已启用

    >备注： 请确保存储帐户名称的长度不超过 15 个字符。该名称将用于在 Active Directory 域服务 (AD DS) 域中创建与包含存储帐户的 Azure 订阅关联的 Azure AD 租户集成的计算机帐户。这将允许在访问此存储帐户中托管的文件共享时进行基于 AD DS 的身份验证。

- [ ] 在 “创建存储帐户” 边栏选项卡的 “基本信息” 选项卡中，选择 “查看 + 创建”，等待验证过程完成，然后选择 “创建”。

    >备注： 请等待存储帐户创建完成。该操作需约 2 分钟。

#### 任务 2：创建 Azure 文件存储共享

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在显示 Azure 门户的 Microsoft Edge 窗口中导航回 “存储帐户” 边栏选项卡，并选择表示新创建的存储帐户的条目。

- [ ] 在“存储帐户”边栏选项卡的 “数据存储” 部分，选择 “文件共享”，然后选择 “+ 文件共享”。

- [ ] 在 “新建文件共享” 边栏选项卡上，指定以下设置并选择 “创建” （其他设置保留默认值）：

    |设置|值
    |:--|:--|
    |名称|avd-22-profiles
    |层级|事务优化

#### 任务 3：为 Azure 存储帐户启用 AD DS 身份验证

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在 Microsoft Edge 窗口中打开一个新标签页，导航到[Azure 文件存储示例 GitHub 存储库](https://github.com/Azure-Samples/azure-files-samples/releases)，下载最新版本的压缩“AzFilesHybrid.zip”PowerShell 模块，并将内容解压缩到 C:\Allfiles\Labs\02 文件夹（如有需要，请创建该文件夹）。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，以管理员的身份启动 Windows PowerShell ISE，然后从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令以删除 Zone.Identifier 备用数据流，该数据流具有一个值 3，指示其是从 Internet 下载的：

    ```powershell
    Get-ChildItem -Path C:\Allfiles\Labs\02 -File -Recurse | Unblock-File
    ```

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台运行以下命令，以登录 Azure 订阅：

    ```powershell
    Connect-AzAccount
    ```

- [ ] 如果出现提示，请使用在具有本实验室所用订阅的所有者角色的用户帐户的 Azure AD 凭据登录。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令，设置运行后续脚本所需的变量：

    ```powershell
    $subscriptionId = (Get-AzContext).Subscription.Id
    $resourceGroupName = 'avd-22-RG'
    $storageAccountName = (Get-AzStorageAccount -ResourceGroupName $resourceGroupName)[0].StorageAccountName
    ```

- [ ]  avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，创建 AD DS 计算机对象，其表示之前在本任务中创建的 Azure 存储帐户，并用于实现其 AD DS 身份验证：

    ```powershell
    Set-Location -Path 'C:\Allfiles\Labs\02'
    .\CopyToPSPath.ps1 
    Import-Module -Name AzFilesHybrid
    Join-AzStorageAccountForAuth `
      -ResourceGroupName $ResourceGroupName `
      -StorageAccountName $StorageAccountName `
      -DomainAccountType 'ComputerAccount' `
      -OrganizationalUnitDistinguishedName 'OU=WVDInfra,DC=adatum,DC=com'
    ```

- [ ]  在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，验证 Azure 存储帐户上是否启用了 AD DS 身份验证：

    ```powershell
    $storageaccount = Get-AzStorageAccount -ResourceGroupName $resourceGroupName -Name $storageAccountName
    $storageAccount.AzureFilesIdentityBasedAuth.ActiveDirectoryProperties
    $storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions
    ```

- [ ]  验证命令 $storageAccount.AzureFilesIdentityBasedAuth.ActiveDirectoryProperties 的输出是否返回 AD （表示存储帐户的目录服务），并验证 $storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions 命令的输出（表示目录域信息）是否类似以下格式（DomainGuid、DomainSid 和 AzureStorageSid 的值将不同）：

    ```powershell
    DomainName        : adatum.com
    NetBiosDomainName : adatum.com
    ForestName        : adatum.com
    DomainGuid        : 47c93969-9b12-4e01-ab81-1508cae3ddc8
    DomainSid         : S-1-5-21-1102940778-2483248400-1820931179
    AzureStorageSid   : S-1-5-21-1102940778-2483248400-1820931179-2109
    ```

- [ ]  在与 avd-dc-vm11 的远程桌面会话中，切换到显示 Azure 门户的 Microsoft Edge 窗口，在显示存储帐户的边栏选项卡上，选择“文件共享”，验证“Active Directory”设置是否为“已配置”。

    >备注： 可能必须刷新浏览器页面才能使更改在 Azure 门户中反映出来。

#### 任务 4：配置 Azure 文件存储基于 RBAC 的权限

- [ ]  依次在与 avd-dc-vm11 的远程桌面会话中，显示 Azure 门户的 Microsoft Edge 窗口中，显示你之前在本练习中创建的存储帐户属性的边栏选项卡上，左侧的垂直菜单中， “数据存储” 部分中，选择 “文件共享”。

- [ ]  在 “文件共享” 边栏选项卡的共享列表中，选择 “avd-22-profiles” 条目。

- [ ]  在 avd-22-profiles 边栏选项卡的左侧垂直菜单中，选择 “访问控制(IAM)”。

- [ ]  在存储帐户的 “访问控制(IAM)” 边栏选项卡上，单击 “+ 添加”，然后在下拉菜单中选择 “添加角色分配”，

- [ ]  在 “添加角色分配” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|
    |:--|:--|
    |角色|存储文件数据 SMB 共享参与者
    |将访问权限分配到|用户、组或服务主体
    |选择|avd-wvd-users

- [ ]  在存储帐户的 “访问控制(IAM)” 边栏选项卡上，单击 “+ 添加”，然后在下拉菜单中选择 “添加角色分配”，

- [ ]  在 “添加角色分配” 边栏选项卡上，指定以下设置并选择 “保存”：

    |设置|值
    |:--|:--|
    |角色|存储文件数据 SMB 共享提升参与者
    |将访问权限分配到|用户、组或服务主体
    |选择|avd-wvd-admins

#### 任务5：配置 Azure 文件存储文件系统权限

- [ ]  在与 avd-dc-vm11 的远程桌面会话中，切换到 “管理员: Windows PowerShell ISE” 窗口，然后从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，创建引用你之前在本练习中创建的存储帐户的名称和密钥的变量：

    ```powershell
    $resourceGroupName = 'avd-22-RG'
    $storageAccount = (Get-AzStorageAccount -ResourceGroupName $resourceGroupName)[0]
    $storageAccountName = $storageAccount.StorageAccountName
    $storageAccountKey = (Get-AzStorageAccountKey -ResourceGroupName $resourceGroupName -Name $storageAccountName).Value[0]
    ```

- [ ]   “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，创建映射到你之前在本练习中创建的文件共享的驱动器：

    ```powershell
    $fileShareName = 'avd-22-profiles'
    net use Z: "\\$storageAccountName.file.core.windows.net\$fileShareName" /u:AZURE\$storageAccountName $storageAccountKey
    ```

- [ ]   从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令，查看当前文件系统权限：

    ```powershell
    icacls Z:
    ```

    >备注： 默认情况下， NT Authority\Authenticated Users 和 BUILTIN\Users 都具有允许用户读取其他用户的配置文件容器的权限。你将删除它们并添加所需的最低权限。

- [ ] 从 “管理员: Windows PowerShell ISE” 脚本窗格中，运行以下命令，调整文件系统权限，使其符合最小权限原则：

    ```powershell
    $permissions = 'ADATUM\wvd-admins'+':(F)'
    cmd /c icacls Z: /grant $permissions
    $permissions = 'ADATUM\wvd-users'+':(M)'
    cmd /c icacls Z: /grant $permissions
    $permissions = 'Creator Owner'+':(OI)(CI)(IO)(M)'
    cmd /c icacls Z: /grant $permissions
    icacls Z: /remove 'Authenticated Users'
    icacls Z: /remove 'Builtin\Users'
    ```

    >备注： 或者，你可以通过使用文件资源管理器设置权限。

## 使用 PowerShell 部署和管理主机池和主机

实验室场景
你需要在 Active Directory 域服务 (AD DS) 环境中使用 PowerShell 自动部署 Azure 虚拟桌面主机池和主机。

目标
完成本实验室后，你将能够：

- 使用 PowerShell 部署 Azure 虚拟桌面主机池和主机
- 使用 PowerShell 将主机添加到 Azure 虚拟桌面主机池

### 练习 1：使用 PowerShell 实现 Azure 虚拟桌面主机池和会话主机

本练习的主要任务如下：

- [ ] 使用 PowerShell 准备部署 Azure 虚拟桌面主机池
- [ ] 使用 PowerShell 创建 Azure 虚拟桌面主机池
- [ ] 使用 PowerShell 对运行 Windows 10 企业版的 Azure VM 执行基于模板的部署
- [ ] 使用 PowerShell 将运行 Windows 10 企业版的 Azure VM 作为会话主机添加到 Azure 虚拟桌面主机
- [ ] 验证 Azure 虚拟桌面会话主机的部署

#### 任务 1：使用 PowerShell 准备部署 Azure 虚拟桌面主机池

- [ ] 启动 Web 浏览器，导航到 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在 Azure 门户中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡中，选择 “avd-dc-vm11”。

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    |设置|值
    |:--|:--|
    |用户名|avdAdmin
    |密码|Pa55w.rd1234

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令以标识名为 WVDInfra 的组织单位的可分辨名称，该组织单位将托管 Azure 虚拟桌面池主机的计算机会话对象：

    ```powershell
    (Get-ADOrganizationalUnit -Filter "Name -eq 'WVDInfra'").distinguishedName
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令确定 “ADATUM\avdAdmin” 帐户（该帐户用于将 Azure 虚拟桌面主机加入 AD DS 域 (avdAdmin@adatum.com） 的 UPN 后缀：

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令安装 DesktopVirtualization PowerShell 模块（系统出现提示时，选择 “全部确认”）：

    ```powershell
    Install-Module -Name Az.DesktopVirtualization -Force
    ```

    >备注： 忽略有关正在使用的现有 PowerShell 模块的任何警告。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，启动 Microsoft Edge，导航到 Azure 门户。如果出现提示，请使用在本实验室使用的订阅中具有所有者角色的用户帐户的 Azure AD 凭据登录。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在 Azure 门户中，使用 Azure 门户页顶部的 “搜索资源、服务和文档” 文本框，搜索并导航到 “虚拟网络”，然后在 “虚拟网络” 边栏选项卡中选择 “avd-adds-vnet11”。

- [ ] 在 “avd-adds-vnet11” 边栏选项卡上，选择 “子网”，在 “子网” 边栏选项卡中选择 “+ 子网”，在 “添加子网” 边栏选项卡上，指定以下设置（所有其他设置保留默认值），并单击 “保存”：

    设置|值
    |:--|:--|
    名称|hp3-Subnet
    子网地址范围|10.0.3.0/24

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在 Azure 门户中，使用 Azure 门户页顶部的 “搜索资源、服务和文档” 文本框，搜索并导航到 “网络安全组”，然后在 “网络安全组” 边栏选项卡中选择 “avd-11-RG” 资源组中的安全组。

- [ ] 在“网络安全组”边栏选项卡上的左侧垂直菜单中，在 “设置” 部分中，单击 “属性”。

- [ ] 在 “属性” 边栏选项卡中，单击 “资源 ID” 文本框右侧的 “复制到剪贴板” 图标。

#### 任务 2：使用 PowerShell 创建 Azure 虚拟桌面主机池

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令登录到 Azure 订阅：

    ```powershell
    Connect-AzAccount
    ```

- [ ] 如果出现提示，请提供具有本实验室所用订阅的所有者角色的用户帐户的凭据。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令确定托管 Azure 虚拟网络 avd-adds-vnet11 的 Azure 区域：

    ```powershell
    $location = (Get-AzVirtualNetwork -ResourceGroupName 'avd-11-RG' -Name 'avd-adds-vnet11').Location
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令创建将托管主机池及其资源的资源组：

    ```powershell
    $resourceGroupName = 'avd-24-RG'
    New-AzResourceGroup -Location $location -Name $resourceGroupName
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令创建空主机池：

    ```powershell
    $hostPoolName = 'avd-24-hp3'
    $workspaceName = 'avd-24-ws1'
    $dagAppGroupName = "$hostPoolName-DAG"
    New-AzWvdHostPool -ResourceGroupName $resourceGroupName -Name $hostPoolName -WorkspaceName $workspaceName -HostPoolType Pooled -LoadBalancerType BreadthFirst -Location $location -DesktopAppGroupName $dagAppGroupName -PreferredAppGroupType Desktop 
    ```

    >备注： 通过 New-AzWvdHostPool cmdlet 可创建主机池、工作区和桌面应用组，以及将桌面应用组注册到工作区。可选择创建新工作区或使用现有的工作区。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令检索名为 avd-wvd-pooled 的 Azure AD 组的 objectID 属性：

```powershell
$aadGroupObjectId = (Get-AzADGroup -DisplayName 'wvd-pooled').Id
```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令将名为 avd-wvd-pooled 的 Azure AD 组分配到新建的主机池的默认桌面应用组：

    ```powershell
    $roleDefinitionName = 'Desktop Virtualization User'
    New-AzRoleAssignment -ObjectId $aadGroupObjectId -RoleDefinitionName $roleDefinitionName -ResourceName $dagAppGroupName -ResourceGroupName $resourceGroupName -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
    ```

#### 任务 3：使用 PowerShell 对运行 Windows 10 企业版的 Azure VM 执行基于模板的部署

- [ ] 使用如下命令下载模板文件，并复制到 “C:\AllFiles\Labs\02” 文件夹（如果需要请创建）。

    ```powershell
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeployhp.json
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeployhp.parameters.json
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令部署运行 Windows 10 企业版（多会话）的 Azure VM，该 VM 将用作在前面的任务中创建的主机池中的 Azure 虚拟桌面会话主机：

    ```powershell
    $resourceGroupName = 'avd-24-RG'
    $location = (Get-AzResourceGroup -ResourceGroupName $resourceGroupName).Location
    New-AzResourceGroupDeployment `
     -ResourceGroupName $resourceGroupName `
     -Location $location `
     -Name avdlab24hpDeployment `
     -TemplateFile C:\AllFiles\Labs\02\azuredeployhp.json `
     -TemplateParameterFile C:\AllFiles\Labs\02\azuredeployhp.parameters.json
    ```

    >备注： 等待部署完成后，再继续执行下一个任务。该过程大约需要 5 分钟。
    >备注： 该部署使用 Azure 资源管理器模板来预配 Azure VM，并应用一个 VM 扩展，自动将操作系统加入 “adatum.com” AD DS 域。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令验证第三方会话主机是否成功加入 “adatum.com” AD DS 域：

    ```powershell
    Get-ADComputer -Filter "sAMAccountName -eq 'avd-24-p3-0$'"
    ```

#### 任务 4：使用 PowerShell 将运行 Windows 10 企业版的 Azure VM 作为主机添加到 Azure 虚拟桌面主机池

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在显示 Azure 门户的浏览器窗口，搜索并选择 “虚拟机”，并且在 “虚拟机” 边栏选项卡的虚拟机列表中选择 “avd-24-p3-0”。

- [ ] 在 “avd-24-p3-0” 边栏选项卡中，选择 “连接”，在下拉菜单中选择 “RDP”，在 “avd-24-p3-0 | 连接” 边栏选项卡的 “RDP” 选项卡的 “IP 地址” 下拉列表中，选择 “公共 IP 地址 (10.0.3.4)” 条目，然后选择 “下载 RDP 文件”。

- [ ] 系统出现提示时，请使用以下凭据登录：

    |设置|值
    |:--|:--|
    |用户名|ADATUM\avdAdmin
    |密码|Pa55w.rd1234

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令创建一个文件夹，用于托管将新部署的 Azure VM 作为会话主机添加到本实验室前面预配的主机池所需的文件：

    ```powershell
    $labFilesFolder = 'C:\AllFiles\Labs\02'
    New-Item -ItemType Directory -Path $labFilesFolder
    ```

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令下载 Azure 虚拟桌面代理和启动加载器安装程序（将会话主机添加到主机池必需的操作）：

    ```powershell
    $webClient = New-Object System.Net.WebClient
    $wvdAgentInstallerURL = 'https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrmXv'
    $wvdAgentInstallerName = 'WVD-Agent.msi'
    $webClient.DownloadFile($wvdAgentInstallerURL,"$labFilesFolder/$wvdAgentInstallerName")
    $wvdBootLoaderInstallerURL = 'https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrxrH'
    $wvdBootLoaderInstallerName = 'WVD-BootLoader.msi'
    $webClient.DownloadFile($wvdBootLoaderInstallerURL,"$labFilesFolder/$wvdBootLoaderInstallerName")
    ```

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令安装最新版本的 PowerShellGet 模块（如果提示确认，请选择 “是”）：

    ```powershell
    Install-Module -Name PowerShellGet -Force -SkipPublisherCheck
    ```

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令安装最新版本的 Az.DesktopVirtualization PowerShell 模块：

    ```powershell
    Install-Module -Name Az.DesktopVirtualization -AllowClobber -Force
    Install-Module -Name Az -AllowClobber -Force
    ```

- [ ] 从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令修改 PowerShell 执行策略并登录 Azure 订阅：

    ```powershell
    Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser -Force
    Connect-AzAccount
    ```

- [ ] 如果出现提示，请提供具有本实验室所用订阅的所有者角色的用户帐户的凭据。

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令以生成将新会话主机加入在本练习前面配置的池中所需的令牌：

    ```powershell
    $resourceGroupName = 'avd-24-RG'
    $hostPoolName = 'avd-24-hp3'
    $registrationInfo = New-AzWvdRegistrationInfo -ResourceGroupName $resourceGroupName -HostPoolName $hostPoolName -ExpirationTime $((get-date).ToUniversalTime().AddDays(1).ToString('yyyy-MM-ddTHH:mm:ss.fffffffZ'))
    ```

    >备注： 需要注册令牌才能授权会话主机加入主机池。令牌的到期日期值必须介于从当前日期和时间起一小时到一个月之间。

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令安装 Azure 虚拟桌面代理：

    ```powershell
    Set-Location -Path $labFilesFolder
    Start-Process -FilePath 'msiexec.exe' -ArgumentList "/i $WVDAgentInstallerName", "/quiet", "/qn", "/norestart", "/passive", "REGISTRATIONTOKEN=$($registrationInfo.Token)", "/l* $labFilesFolder\AgentInstall.log" | Wait-Process
    ```

- [ ] 在与 avd-24-p3-0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令安装 Azure 虚拟桌面启动加载器：

    ```powershell
    Start-Process -FilePath "msiexec.exe" -ArgumentList "/i $wvdBootLoaderInstallerName", "/quiet", "/qn", "/norestart", "/passive", "/l* $labFilesFolder\BootLoaderInstall.log" | Wait-process
    ```

#### 任务 5：验证 Azure 虚拟桌面主机的部署

- [ ] 切换到实验室计算机，在显示 Azure 门户的 Web 浏览器中，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “主机池”，然后在 “Azure 虚拟桌面 | 主机池” 边栏选项卡上，选择表示新修改的池的条目 avd-24-hp3。
- [ ] 在 “avd-24-hp3” 边栏选项卡左侧垂直菜单中的 “管理” 部分中，单击 “会话主机”。
- [ ] 在 “avd-24-hp3 | 会话主机” 边栏选项卡上，验证该部署是否包含单个主机。

#### 任务 6：使用 PowerShell 管理应用组

- [ ] 切换到与 avd-dc-vm11 的远程桌面会话，从 “管理员: Windows PowerShell ISE” 脚本窗格，运行以下命令创建远程应用组：

    ```powershell
    $subscriptionId = (Get-AzContext).Subscription.Id
    $appGroupName = 'avd-24-hp3-Office365-RAG'
    $resourceGroupName = 'avd-24-RG'
    $hostPoolName = 'avd-24-hp3'
    $location = (Get-AzVirtualNetwork -ResourceGroupName 'avd-11-RG' -Name 'avd-adds-vnet11').Location
    New-AzWvdApplicationGroup -Name $appGroupName -ResourceGroupName $resourceGroupName -ApplicationGroupType 'RemoteApp' -HostPoolArmPath "/subscriptions/$subscriptionId/resourcegroups/$resourceGroupName/providers/Microsoft.DesktopVirtualization/hostPools/$hostPoolName"-Location $location
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令在池主机上列出 “开始” 菜单应用并查看输出：

    ```powershell
    Get-AzWvdStartMenuItem -ApplicationGroupName $appGroupName -ResourceGroupName $resourceGroupName | Format-List | more
    ```

    >备注： 对于任何要发布的应用程序，都应记录包含在输出中的信息，其中包括 FilePath、 IconPath 和 IconIndex 等参数。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令发布 Microsoft Word：

    ```powershell
    $name = 'Microsoft Word'
    $filePath = 'C:\Program Files\Microsoft Office\root\Office16\WINWORD.EXE'
    $iconPath = 'C:\Program Files\Microsoft Office\Root\VFS\Windows\Installer\{90160000-000F-0000-1000-0000000FF1CE}\wordicon.exe'
    New-AzWvdApplication -GroupName $appGroupName -Name $name -ResourceGroupName $resourceGroupName -FriendlyName $name -Filepath $filePath -IconPath $iconPath -IconIndex 0 -CommandLineSetting 'DoNotAllow' -ShowInPortal:$true
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令发布 Microsoft Word：

    ```powershell
    $aadGroupObjectId = (Get-AzADGroup -DisplayName 'wvd-remote-app').Id
    New-AzRoleAssignment -ObjectId $aadGroupObjectId -RoleDefinitionName 'Desktop Virtualization User' -ResourceName $appGroupName -ResourceGroupName $resourceGroupName -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
    ```

- [ ] 切换到实验室计算机，在显示 Azure 门户的 Web 浏览器中，在 “avd-24-hp3 | 会话主机” 边栏选项卡左侧垂直菜单中的 “管理” 部分，选择 “应用程序组”。

- [ ] 在 “avd-24-hp3 | 应用程序组” 边栏选项卡的应用程序组列表中，选择 “avd-24-hp3-Office365-RAG” 条目。

- [ ] 在 “avd-24-hp3-Office365-RAG” 边栏选项卡上，验证应用程序组的配置，其中包括应用程序和分配。

## 创建和管理会话主机映像 (AD DS)

实验室场景
你需要在 Active Directory 域服务 (AD DS) 环境中创建和管理 Azure 虚拟桌面主机映像。

目标
完成本实验室后，你将能够：

- 创建和管理 WVD 会话主机映像

### 练习 1：创建和管理会话主机映像

本练习的主要任务如下：

- [ ] 准备配置 Azure 虚拟桌面主机映像
- [ ] 部署 Azure Bastion
- [ ] 配置 Azure 虚拟桌面主机映像
- [ ] 创建 Azure 虚拟桌面主机映像
- [ ] 使用自定义映像预配 Azure 虚拟桌面主机池

#### 任务 1：准备配置 Azure 虚拟桌面主机映像

- [ ] 启动 Web 浏览器，导航到 Azure 门户，然后通过提供你将在本实验室使用的订阅中具有所有者角色的用户帐户凭据进行登录。

- [ ] 在 Azure 门户中，通过选择搜索文本框右侧的“工具栏”图标，打开 “Cloud Shell” 窗格。

- [ ] 提示选择 “Bash” 或 “PowerShell” 时，选择 “PowerShell”。

- [ ] 在实验室计算机显示 Azure 门户的 Web 浏览器中，从“Cloud Shell”窗格内的 PowerShell 会话运行以下命令，以创建将包含 Azure 虚拟桌面主机映像的资源组：

    ```powershell
    $vnetResourceGroupName = 'avd-11-RG'
    $location = (Get-AzResourceGroup -ResourceGroupName $vnetResourceGroupName).Location
    $imageResourceGroupName = 'avd-25-RG'
    New-AzResourceGroup -Location $location -Name $imageResourceGroupName
    ```

- [ ] 在 Azure 门户的 Cloud Shell 窗格的工具栏中，运行如下命令下载文件。

    ```powershell
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeployvm.json
    wget -N https://raw.githubusercontent.com/shibaoxi/LearnNotes/master/azure/configfile/azuredeployvm.parameters.json
    ```

- [ ] 从“Cloud Shell”窗格中的 PowerShell 会话，运行以下命令将运行 Windows 10 的 Azure VM（该 VM 将用作 Azure 虚拟桌面客户端）部署到新建的子网：

    ```powershell
    New-AzResourceGroupDeployment `
     -ResourceGroupName $imageResourceGroupName `
     -Name avdlab0205vmDeployment `
     -TemplateFile $HOME/azuredeployvm.json `
     -TemplateParameterFile $HOME/azuredeployvm.parameters.json
    ```

    >备注：等待部署完成后，再继续下一个练习。部署可能需要大约 10 分钟。

#### 任务 2： 部署 Azure Bastion

>备注：通过 Azure Bastion，可以连接到你在本练习的上一个任务中部署的没有公共终结点的 Azure VM，同时提供针对操作系统级别凭据的暴力攻击防护。

- [ ] 在显示 Azure 门户的浏览器窗口中，打开另一个选项卡，然后在该浏览器选项卡中导航到 Azure 门户。

- [ ] 在 Azure 门户中，通过选择搜索文本框右侧的“工具栏”图标，打开 Cloud Shell 窗格。

- [ ] 在 Cloud Shell 窗格中的 PowerShell 会话中运行以下命令，将名为 AzureBastionSubnet 的子网添加到你在本练习前面部分创建的名为 avd-25-vnet 的虚拟网络中：

    ```powershell
    $resourceGroupName = 'avd-25-RG'
    $vnet = Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name 'avd-25-vnet'
    $subnetConfig = Add-AzVirtualNetworkSubnetConfig `
     -Name 'AzureBastionSubnet' `
     -AddressPrefix 10.25.254.0/24 `
     -VirtualNetwork $vnet
    $vnet | Set-AzVirtualNetwork
    ```

- [ ] 关闭 Cloud Shell 窗格。

- [ ] 在 Azure 门户中搜索并选择“Bastion”，然后在“Bastion”边栏选项卡中，选择“+ 创建”。

- [ ] 在“创建 Bastion”边栏选项卡的“基本信息”选项卡中，指定以下设置并选择“查看 + 创建”：

    设置|值
    |:--|:--|
    订阅|在本实验室中使用的 Azure 订阅的名称
    资源组|avd-25-RG
    名称|avd-25-bastion
    区域|在本练习前面的任务中向其中部署了资源的同一 Azure 区域
    层级|基本
    虚拟网络|avd-25-vnet
    子网|AzureBastionSubnet (10.25.254.0/24)
    公共 IP 地址|新建
    公共 IP 名称|avd-25-vnet-ip

- [ ] 在“创建 Bastion”边栏选项卡的“查看 + 创建”选项卡中，选择“创建”：

    >备注：等待部署完成后，再继续下一个练习。该部署可能需要大约 5 分钟。

#### 任务 3： 配置 Azure 虚拟桌面主机映像

- [ ] 在 Azure 门户中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡上，选择 “avd-25-vm0”。

- [ ] 在“avd-25-vm0”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-25-vm0 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    设置|值
    |:--|:--|
    用户名|Student
    密码|Pa55w.rd1234

    >备注：首先将安装 FSLogix 二进制文件。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令，创建将用作映像配置临时位置的文件夹：

    ```powershell
    New-Item -Type Directory -Path 'C:\Allfiles\Labs\02' -Force
    ```

- [ ] 在与 avd-25-vm0 的远程桌面会话中，启动 Microsoft Edge，浏览至 [FSLogix 下载页](https://aka.ms/fslogix_download)，将 FSLogix 压缩安装二进制文件下载到 “C:\Allfiles\Labs\02” 文件夹中，然后在文件资源管理器中，将 x64 子文件夹解压缩到同一文件夹。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，切换到 “管理员: Windows PowerShell ISE” 窗口，然后从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令以执行按计算机安装 OneDrive：

    ```powershell
    Start-Process -FilePath 'C:\Allfiles\Labs\02\x64\Release\FSLogixAppsSetup.exe' -ArgumentList '/quiet' -Wait
    ```

    >备注：请等待安装完成。这大约需要 1 分钟。若安装触发重启，请重新连接到 avd-25-vm0。
    >备注：接下来，你将逐步安装并配置 Microsoft Teams（出于学习目的，因为 Teams 已经存在于本实验室使用的映像中）。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，右键单击 “开始”，在右键单击菜单中，选择 “运行”，在 “运行” 对话框的 “打开” 文本框中，键入 “cmd” 并按 Enter 键，以启动 “命令提示符”。

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以准备按计算机安装 Microsoft Teams：

    ```powershell
    reg add "HKLM\Software\Microsoft\Teams" /v IsWVDEnvironment /t REG_DWORD /d 1 /f
    ```

- [ ] 在与 avd-25-vm0 的远程桌面会话中，在 Microsoft Edge 中导航至 [Microsoft Visual C++ 可再发行程序包下载页](https://aka.ms/vs/16/release/vc_redist.x64.exe)，将 VC_redist.x64 保存至 “C:\Allfiles\Labs\02” 文件夹。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，切换到 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以执行安装 Microsoft Visual C++ 可再发行程序包：

    ```powershell
    C:\Allfiles\Labs\02\vc_redist.x64.exe /install /passive /norestart /log C:\Allfiles\Labs\02\vc_redist.log
    ```

- [ ] 在与 avd-25-vm0 的远程桌面会话中，在 Microsoft Edge 中浏览至标题为 [将 Teams 桌面应用部署到 VM](https://docs.microsoft.com/zh-cn/microsoftteams/teams-for-vdi#deploy-the-teams-desktop-app-to-the-vm) 的文档页，单击 “64 位版本” 链接，然后系统出现提示时，将 Teams_windows_x64.msi 文件保存至 “C:\Allfiles\Labs\02” 文件夹。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，切换到 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以执行按计算机安装 Microsoft Teams：

    ```powershell
    msiexec /i C:\Allfiles\Labs\02\Teams_windows_x64.msi /l*v C:\Allfiles\Labs\02\Teams.log ALLUSER=1
    ```

    >备注： 此安装程序支持 ALLUSER=1 和 ALLUSERS=1 参数。ALLUSER=1 参数用于在 VDI 环境中的按计算机安装。ALLUSERS=1 参数可用于非 VDI 环境和 VDI 环境。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE，并从“管理员: Windows PowerShell ISE”控制台中运行以下命令，以安装 Microsoft Edge Chromium（出于学习目的，因为 Microsoft Edge 已存在于本实验室使用的映像中）：

    ```powershell
    Start-BitsTransfer -Source "https://aka.ms/edge-msi" -Destination 'C:\Allfiles\Labs\02\MicrosoftEdgeEnterpriseX64.msi'
    Start-Process -Wait -Filepath msiexec.exe -Argumentlist "/i C:\Allfiles\Labs\02\MicrosoftEdgeEnterpriseX64.msi /q"
    ```

    >备注： 请等待安装完成。该过程大约需要 2 分钟。
在多语言环境进行操作时，可能需要安装语言包。有关此过程的详细信息，请参阅 Microsoft Docs 文章[将语言包添加到 Windows 10 多会话映像](https://docs.microsoft.com/zh-cn/azure/virtual-desktop/language-packs)。接下来，你将禁用 Windows 自动更新和存储感知、配置时区重定向以及配置遥测收集。通常，应先应用所有当前的更新。在本实验室中，跳过此步骤是为了最大程度地缩短实验室时间。

- [ ] 在与 avd-25-vm0 的远程桌面会话中，切换到 “管理员: C:\windows\system32\cmd.exe” 窗口，从命令提示符运行以下命令以禁用自动更新：

    ```powershell
    reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v NoAutoUpdate /t REG_DWORD /d 1 /f
    ```

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以禁用存储感知：

    ```powershell
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\StorageSense\Parameters\StoragePolicy" /v 01 /t REG_DWORD /d 0 /f
    ```

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以配置时区重定向：

    ```powershell
    reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v fEnableTimeZoneRedirection /t REG_DWORD /d 1 /f
    ```

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以禁用反馈中心遥测数据收集：

    ```powershell
    reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
    ```

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行以下命令以删除前面在本任务中创建的临时文件夹：

    ```powershell
    rmdir C:\Allfiles /s /q
    ```

- [ ] 在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行磁盘清理实用工具，在完成后单击 “确定”：

    ```powershell
    cleanmgr /d C: /verylowdisk
    ```

#### 任务 4：创建 Azure 虚拟桌面主机映像

- [ ] 在与 avd-25-vm0 的远程桌面会话中，在 “管理员: C:\windows\system32\cmd.exe” 窗口中，从命令提示符运行 Sysprep 实用工具，以便让操作系统准备好生成映像并自动将操作系统关闭：

    ```powershell
    C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown
    ```

    >备注： 请等待 Sysprep 过程完成。该过程大约需要 2 分钟。这会自动关闭操作系统。

- [ ] 在实验室计算机显示 Azure 门户的 Web 浏览器中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡上，选择 avd-25-vm0” 条目。

- [ ] 在 avd-25-vm0” 边栏选项卡 “Essentials” 部分上方的工具栏中，单击 “刷新”，验证 Azure VM 的 “状态” 是否已更改为 “已停止”，单击 “停止”，并在出现确认提示时单击 “确定”，从而将 Azure VM 的状态转换为 “已停止（已解除分配）”。

- [ ] 在 avd-25-vm0” 边栏选项卡中，验证 Azure VM 的 “状态” 是否已更改为 “已停止（已解除分配）”，并在工具栏中单击 “捕获”。这将自动显示 “创建映像” 边栏选项卡。

- [ ] 在 “创建映像” 边栏选项卡的 “基本” 选项卡中，指定以下设置：

    设置|值
    |:--|:--|
    将映像共享到 Azure 计算库|是，将其作为映像版本共享到库
    创建映像后自动删除此虚拟机|复选框已清除
    目标 Azure 计算库|新库名称 avd25imagegallery
    操作系统状态|通用

- [ ] 在 “创建映像” 边栏选项卡的 “基本” 选项卡中，单击 “目标映像定义” 文本框下方的 “新建”。

- [ ] 在 “创建映像定义” 中，指定以下设置，然后单击 “确定”：

    |设置|值
    |:--|:--|
    |映像定义名称|avd-25-host-image
    |发布者|MicrosoftWindowsDesktop
    |产品/服务|office-365
    |SKU|20h1-evd-o365pp

- [ ] 返回 “创建映像” 边栏选项卡的 “基本” 选项卡上，指定以下设置并单击 “查看 + 创建”：

    设置|值
    |:--|:--|
    |版本号|1.0.0
    |从最新版中排除|复选框已清除
    |生命周期终结日期|距离当前日期还有一年
    |默认副本计数|1
    |目标区域副本计数|1
    |存储帐户类型|高级 SSD

- [ ] 在 “创建映像” 边栏选项卡的 “查看 + 创建” 选项卡上，单击 “创建”。

    >备注： 请等待部署完成。该操作需要约 20 分钟。

- [ ] 在显示 Azure 门户的 Web 浏览器中，搜索并选择“Azure 计算库”，在“Azure 计算库”边栏选项卡上选择“avd25imagegallery”条目，然后在“avd25imagegallery”边栏选项卡上，验证表示新建映像的 avd-25-host-image 条目是否存在。

#### 任务 5：使用自定义映像预配 Azure 虚拟桌面主机池

- [ ] 在Azure 门户中，使用 Azure 门户页顶部的 “搜索资源、服务和文档” 文本框，搜索并导航到 “虚拟网络”，然后在 “虚拟网络” 边栏选项卡中选择 “avd-adds-vnet11”。

- [ ] 在 “avd-adds-vnet11” 边栏选项卡中选择 “子网”，在 “子网” 边栏选项卡中选择 “+ 子网”，在 “添加子网” 边栏选项卡中指定以下设置（保留所有其他设置的默认值）后单击 “保存”：

    设置|值
    |:--|:--|
    |名称|hp4-Subnet
    |子网地址范围|10.0.4.0/24

- [ ] 在 Azure 门户的 Web 浏览器窗口中，搜索并选择 “Azure 虚拟桌面”，在 “Azure 虚拟桌面” 边栏选项卡上，选择 “主机池”，然后在 “Azure 虚拟桌面 | 主机池” 边栏选项卡中，选择 “+ 添加”。

- [ ] 在 “创建主机池” 边栏选项卡的 “基本信息” 选项卡中，指定以下设置并选择 “下一步: 虚拟机 >”：

    |设置|值
    |:--|:--|
    |订阅|在本实验室中使用的 Azure 订阅的名称
    |资源组|avd-25-RG
    |主机池名称|avd-25-hp4
    |位置|在本实验室的第一个练习中，将资源部署到其中的 Azure 区域的名称
    |验证环境|否
    |主机池类型|共用
    |最大会话限制|50
    |负载均衡算法|广度优先

- [ ] 在 “创建主机池” 边栏选项卡的 “虚拟机” 选项卡中，指定以下设置：

    |设置|值
    |:--|:--|
    |添加虚拟机|是
    |资源组|默认与主机池相同
    |名称前缀|avd-25-p4
    |虚拟机位置|在本实验室的第一个练习中，将资源部署到其中的 Azure 区域的名称
    |可用性选项|无需基础结构冗余
    |映像类型|库

- [ ] 在 “创建主机池” 边栏选项卡的 “虚拟机” 选项卡上，单击 “映像” 下拉列表正下方的 “查看所有映像” 链接。

- [ ] 在 “选择映像” 边栏选项卡上，依次单击 “我的项” 和 “共享映像”，然后在共享映像列表中选择 “avd-25-host-image”。

- [ ] 在 “创建主机池” 边栏选项卡的 “虚拟机” 选项卡中，指定以下设置并选择 “下一步: 工作区 >”

    |设置|值
    |:--|:--|
    |虚拟机大小|标准 D2s v3
    |VM 数|1
    |OS 磁盘类型|标准 SSD LRS
    |虚拟网络|avd-adds-vnet11
    |子网|hp4-Subnet (10.0.4.0/24)
    |网络安全组|基本
    |公共入站端口|是
    |允许的入站端口|RDP
    |AD 域加入 UPN|avdAdmin@adatum.com
    |密码|Pa55w.rd1234
    |指定域或单元|是
    |要加入的域|adatum.com
    |组织单位路径|OU=WVDInfra,DC=adatum,DC=com
    |用户名|avdAdmin
    |密码|Pa55w.rd1234
    |确认密码|Pa55w.rd1234

- [ ] 在 “创建主机池” 边栏选项卡的 “工作区” 选项卡中，指定以下设置并选择 “查看 + 创建”：

    |设置|值
    |:--|:--|
    |注册桌面应用组|否

- [ ] 在 “创建主机池” 边栏选项卡的 “查看 + 创建” 选项卡中，选择 “创建”。

    >备注： 请等待部署完成。这需要约 10 分钟时间。
    >在根据自定义映像部署主机后，应考虑运行虚拟桌面优化工具（由它的 [GitHub](https://github.com/The-Virtual-Desktop-Team/) 存储库提供）。

## 为 WVD (AD DS) 配置条件访问策略

实验室场景
在 Active Directory 域服务 (AD DS) 环境中，你需要使用 Azure Active Directory (Azure AD) 条件访问控制对 Azure 虚拟桌面的部署的访问。

目标
完成本实验室后，你将能够：

- 为 Azure 虚拟桌面准备基于 Azure Active Directory (Azure AD) 的条件访问
- 为 Azure 虚拟桌面实现基于 Azure AD 的条件访问

### 练习 1：为 Azure 虚拟桌面准备基于 Azure AD 的条件访问

本练习的主要任务如下：

- [ ] 配置 Azure AD Premium P2 许可
- [ ] 配置 Azure AD 多重身份验证 (MFA)
- [ ] 为 Azure AD MFA 注册用户
- [ ] 配置混合 Azure AD 联接
- [ ] 触发 Azure AD Connect 增量同步

#### 任务 1：配置 Azure AD Premium P2 许可

- [ ] 在实验室计算机上，启动 Web 浏览器，导航至 Azure 门户，然后通过提供用户帐户（该帐户具有你将在本实验室使用的订阅中的所有者角色以及与该订阅关联的 Azure AD 租户中的全局管理员角色）的凭据进行登录。

- [ ] 在 Azure 门户中，搜索并选择 Azure Active Directory，导航到与用于此实验室的 Azure 订阅相关联的 Azure AD 租户。

- [ ] 在 Azure Active Directory 边栏选项卡左侧的垂直菜单栏的 “管理” 部分中，单击 “用户”。

- [ ] 在 “用户 | 所有用户（预览）” 边栏选项卡上，选择 “aduser5”。

- [ ] 在 “aduser5 | 配置文件” 边栏选项卡上的工具栏中，单击 “编辑”，在 “设置” 部分的 “使用位置” 下拉列表中，选择实验室环境所在的国家/地区，然后在工具栏中，单击 “保存”。

- [ ] 在 “aduser5 | 配置文件” 边栏选项卡上的 “标识” 部分中，标识 “aduser5” 帐户的用户主体名称。

    >备注： 记录此值。稍后将在本实验室用到它。

- [ ] 在 “用户 | 所有用户（预览）” 边栏选项卡上，选择在此任务开始时用于登录的用户帐户，如果帐户没有分配 “使用位置”，则重复前面的步骤。

    >备注：必须设置 “使用位置” 属性才能为用户帐户分配 Azure AD Premium P2 许可证。

- [ ] 在 “用户 | 所有用户（预览）” 边栏选项卡上，选择 “aadsyncuser” 用户帐户并识别其用户主体名称。

    >备注： 记录此值。稍后将在本实验室用到它。

- [ ] 在 Azure 门户中，导航回 Azure AD 租户的 “概述” 边栏选项卡，并在左侧垂直菜单栏的 “管理” 部分中，单击 “许可证”。

- [ ] 在 “许可证 | 概述” 边栏选项卡左侧垂直菜单栏的 “管理” 部分中，单击 “所有产品”。

- [ ] 在 “许可证 | 所有产品” 边栏选项卡上的工具栏中，单击 “+ 试用/购买”。

- [ ] 在 “激活” 边栏选项卡上的 “企业移动性 + 安全性 E5” 部分中单击 “免费试用”，然后单击 “激活”。

- [ ] 在 “许可证 | 概述” 边栏选项卡上，刷新浏览器窗口以验证激活是否成功。

- [ ] 在 “许可证 - 所有产品” 边栏选项卡上，选择 “企业移动性 + 安全性 E5” 条目。

- [ ] 在 “企业移动性 + 安全性 E5” 边栏选项卡上的工具栏中，单击 “+ 分配”。

- [ ] 在“分配许可证”边栏选项卡上，单击“添加用户和组”，然后在“添加用户和组”边栏选项卡上，选择“aduser5”和你的用户帐户，并单击“选择”。

- [ ] 回到 “分配许可证” 边栏选项卡上，单击 “分配选项”，在 “分配选项” 边栏选项卡上，验证所有选项都已启用，然后依次单击 “查看 + 分配” 和 “分配”。

#### 任务 2：配置 Azure AD 多重身份验证 (MFA)

- [ ] 在实验室计算机上显示 Azure 门户的 Web 浏览器中，导航回 Azure AD 租户的 “概述” 边栏选项卡，并在左侧垂直菜单的 “管理” 部分中，单击 “安全性”。
- [ ] 在 “安全性 | 启动” 边栏选项卡上左侧垂直菜单栏的 “管理” 部分中，单击 “标识保护”。
- [ ] 在 “标识保护 | 概述” 边栏选项卡上左侧垂直菜单栏的 “保护” 部分中，单击 “MFA 注册策略” （如有必要，请刷新 Web 浏览器页面）。
- [ ] 在 “标识保护 | MFA 注册策略” 边栏选项卡上，在 “多重身份验证注册策略” 的 “分配” 部分中，单击 “所有用户”，在 “包括” 选项卡上，单击 “选择个人和组” 选项，在 “选择用户” 上，依次单击 “aduser5” 和 “选择”，在边栏选项卡底部，将 “执行策略” 切换为 “开”，然后单击 “保存”。

#### 任务 3：为 Azure AD MFA 注册用户

- [ ] 在实验室计算机上，打开“InPrivate” Web 浏览器会话，导航到 Azure 门户，并通过提供此前在此练习中指定的用户主体名称 aduser5 和创建该用户帐户时设置的密码来登录。
- [ ] 出现消息 “需要更多信息” 时，单击 “下一步”。这会自动将浏览器重定向到 “Microsoft Authenticator” 页面。
- [ ] 在 “附加安全验证” 页面上，在 “步骤 1: 我们应该如何与你联系？” 部分，选择你首选的身份验证方法并按照说明完成注册过程。
- [ ] 在 Azure 门户页的右上角，单击代表用户头像的图标，单击 “注销”，并关闭 “In private” 浏览器窗口。

#### 任务 4：配置混合 Azure AD 联接

>备注：当根据设备的 Azure AD 联接状态为其设置条件访问时，可以利用此功能实现额外的安全性。

- [ ] 切换到实验室计算机，在显示 Azure 门户的 Web 浏览器中，搜索并选择 “虚拟机”，然后在 “虚拟机” 边栏选项卡中选择 “avd-dc-vm11”。

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“avd-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

- [ ] 出现提示时，请提供以下凭据并选择“连接”：

    |设置|值
    |用户名|avdAdmin
    |密码|Pa55w.rd1234

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在“开始”菜单中展开 Azure AD Connect 文件夹，并选择“Azure AD Connect”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “欢迎使用 Azure AD Connect” 页上，选择 “配置”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “其他任务” 页上，选择 “配置设备选项”，并选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “概述” 页上，查看与 “混合 Azure AD 联接” 和 “设备写回” 相关的信息，并选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “连接到 Azure AD” 页上，使用在前面的练习中创建的用户帐户 “aadsyncuser” 的凭据进行身份验证，然后选择 “下一步”。

    >备注： 提供之前在本实验室记录的 aadsyncuser 帐户的 userPrincipalName 属性，并指定使用你创建此用户帐户时设置的密码。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “设备选项” 页上，确保已选择 “配置混合 Azure AD 联接” 选项，并选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “设备操作系统” 页上，选择 “Windows 10 或更高版本的已加入域的设备” 复选框，并选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “SCP 配置” 页上，选择 “adatum.com” 条目旁边的复选框，在 “身份验证服务” 下拉列表中，选择 “Azure Active Directory” 条目，并选择 “添加”。

- [ ] 出现提示时，在 “企业管理员凭据” 对话框中指定以下凭据，并选择 “确定”：

    |设置|值
    |:--|:--|
    |用户名|ADATUM\avdAdmin
    |密码|Pa55w.rd1234

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “准备配置” 页上，选择 “配置”，配置完成后选择 “退出”。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，以管理员身份启动 Windows PowerShell ISE。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令，将 “avd-cl-vm11” 计算机帐户移动到 “WVDClients” 组织单位 (OU)：

    ```powershell
    Move-ADObject -Identity "CN=az140-cl-vm11,CN=Computers,DC=adatum,DC=com" -TargetPath "OU=WVDClients,DC=adatum,DC=com"
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在 “开始” 菜单中展开 Azure AD Connect 文件夹，并选择 “Azure AD Connect”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “欢迎使用 Azure AD Connect” 页上，选择 “配置”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “附加任务” 页上，依次选择 “自定义同步选项” 和 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “连接到 Azure AD” 页上，使用在前面的练习中创建的用户帐户 “aadsyncuser” 的凭据进行身份验证，然后选择 “下一步”。

    >备注： 提供之前在本实验室记录的 aadsyncuser 帐户的 userPrincipalName 属性，并指定使用你创建此用户帐户时设置的密码。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “连接到目录” 页上，选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口中的 “域和 OU 筛选” 页上，确保已选择 “同步已选择的域和 OU” 选项，展开 “adatum.com” 节点，确保已选择 “ToSync” OU 旁的复选框，然后选择 “WVDClients” OU 旁的复选框，并选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “可选功能” 页上，接受默认设置，然后选择 “下一步”。

- [ ] 在 “Microsoft Azure Active Directory Connect” 窗口的 “准备配置” 页上，确保已选中 “配置完成后启动同步过程” 复选框，然后选择 “配置”。

- [ ] 查看 “配置完成” 页中的信息，然后选择 “退出” 关闭 “Microsoft Azure Active Directory Connect” 窗口。

#### 任务 5：触发 Azure AD Connect 增量同步

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，切换到 “管理员: Windows PowerShell ISE” 窗口。

- [ ] 在与 avd-dc-vm11 远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台窗格，运行以下命令，触发 Azure AD Connect 增量同步：

    ```powershell
    Import-Module -Name "C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync"
    Start-ADSyncSyncCycle -PolicyType Initial
    ```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，启动 Microsoft Edge，并导航到 Azure 门户。如果出现提示，请使用用户帐户的 Azure AD 凭据登录，该帐户应具有与本实验室所用 Azure 订阅关联的 Azure AD 租户的全局管理员角色。

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在显示 Azure 门户的 Microsoft Edge 窗口中搜索并选择 “Azure Active Directory”，以导航到与本实验室中所用 Azure 订阅关联的 Azure AD 租户。

- [ ] 在“Azure Active Directory”边栏选项卡左侧垂直菜单中的 “管理” 部分，单击 “设备”。

- [ ] 在 “设备 | 所有设备” 边栏选项卡上，查看设备列表，验证 “avd-cl-vm11” 设备是否在 “联接类型” 列中列出了 “已建立混合 Azure AD 联接” 条目。

    >备注： 在设备出现在 Azure 门户中之前，可能需要等待几分钟才能使同步生效。

### 练习 2：为 Azure 虚拟桌面实现基于 Azure AD 的条件访问

本练习的主要任务如下：

- [ ] 为所有 Azure 虚拟桌面连接创建基于 Azure AD 的条件访问策略
- [ ] 为所有 Azure 虚拟桌面连接测试基于 Azure AD 的条件访问策略
- [ ] 修改基于 Azure AD 的条件访问策略，将已加入混合 Azure AD 的计算机从 MFA 要求中排除
- [ ] 测试修改后的基于 Azure AD 的条件访问策略

#### 任务 1：为所有 Azure 虚拟桌面连接创建基于 Azure AD 的条件访问策略

>备注：在此任务中，你将配置一个基于 Azure AD 的条件访问策略，该策略要求 MFA 登录到 Azure 虚拟桌面会话。该策略还将在身份验证成功后的 4 个小时后强制重新进行身份验证。

- [ ] 在 Azure 门户的 web 浏览器中，导航回 Azure AD 租户的 “概述” 边栏选项卡，并在左侧垂直菜单的 “管理” 部分中，单击 “安全性”。

- [ ] 在 “安全性 | 启动” 边栏选项卡上左侧垂直菜单栏的 “保护” 部分中，单击 “条件访问”。

- [ ] 在 “条件访问 | 策略” 边栏选项卡上的工具栏中，单击 “新建策略”。

- [ ] 在 “新建” 边栏选项卡上，配置以下设置：

  - 在 “名称” 文本框中键入 “az140-31-wvdpolicy1”
  - 在 “分配” 部分中，单击 “用户和组”，单击 “选择用户和组” 选项，然后单击 “用户和组” 复选框，在 “选择” 边栏选项卡上，单击 “aduser5”，然后单击 “选择”。
  - 在 “分配” 部分中，单击 “云应用或操作”，确保已选择 “选择要应用的策略” 切换中的 “云应用” 选项，然后单击 “选择应用” 选项，在 “选择” 边栏选项卡上，选择 “Windows 虚拟桌面” 条目旁的复选框，然后单击 “选择”。
  - 在 “分配” 部分中，依次单击 “条件” 和 “客户端应用”，在 “客户端应用” 边栏选项卡上，将 “配置” 开关设置为 “是”，确保已选择 “浏览器” 和 “移动应用和桌面客户端” 复选框，然后单击 “完成”。
  - 在 “访问控制” 部分，单击 “授予”，在 “授予” 边栏选项卡上，确保已选择 “授予访问权限” 选项，选择 “要求多重身份验证” 复选框，然后单击 “选择”。
  - 在 “访问控制” 部分中，单击 “会话”，在 “会话” 边栏选项卡上，选择 “登录频率” 复选框，然后在第一个文本框中，键入 “4”，在 “选择单位” 下拉列表中，选择 “小时”，清除 “持续的浏览器会话” 复选框，然后单击 “选择”。
  - 将 “启用策略” 开关设置为 “开”。

- [ ] 在 “新建” 边栏选项卡上，单击 “创建”。

#### 任务 2：为所有 Azure 虚拟桌面连接测试基于 Azure AD 的条件访问策略