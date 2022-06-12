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

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“az140-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

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
- [ ] 在 “选择 Azure AD 用户或用户组” 边栏选项卡上，选择 “az140-wvd-remote-app”，然后单击 “选择”。
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

- [ ] 在 “选择 Azure AD 用户或用户组” 边栏选项卡上，选择 “az140-wvd-remote-app” 和 “az140-wvd-admins”，然后单击 “选择”。

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
    |资源组|新资源组名称 az140-22-RG
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
    $resourceGroupName = 'az140-22-RG'
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
    |选择|az140-wvd-users

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
    $permissions = 'ADATUM\az140-wvd-admins'+':(F)'
    cmd /c icacls Z: /grant $permissions
    $permissions = 'ADATUM\az140-wvd-users'+':(M)'
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

- [ ] 在“avd-dc-vm11”边栏选项卡上，选择“连接”，在下拉菜单中选择“Bastion”，在“az140-dc-vm11 | 连接”边栏选项卡的“Bastion”选项卡上，选择“使用 Bastion”。

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

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，在 Azure 门户中，使用 Azure 门户页顶部的 “搜索资源、服务和文档” 文本框，搜索并导航到 “虚拟网络”，然后在 “虚拟网络” 边栏选项卡中选择 “az140-adds-vnet11”。

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
$aadGroupObjectId = (Get-AzADGroup -DisplayName 'az140-wvd-pooled').Id
```

- [ ] 在与 avd-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令将名为 avd-wvd-pooled 的 Azure AD 组分配到新建的主机池的默认桌面应用组：

    ```powershell
    $roleDefinitionName = 'Desktop Virtualization User'
    New-AzRoleAssignment -ObjectId $aadGroupObjectId -RoleDefinitionName $roleDefinitionName -ResourceName $dagAppGroupName -ResourceGroupName $resourceGroupName -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
    ```

#### 任务 3：使用 PowerShell 对运行 Windows 10 企业版的 Azure VM 执行基于模板的部署

使用如下命令下载模板文件，并复制到 “C:\AllFiles\Labs\02” 文件夹（如果需要请创建）。



在与 az140-dc-vm11 的远程桌面会话中，从 “管理员: Windows PowerShell ISE” 控制台，运行以下命令部署运行 Windows 10 企业版（多会话）的 Azure VM，该 VM 将用作在前面的任务中创建的主机池中的 Azure 虚拟桌面会话主机：