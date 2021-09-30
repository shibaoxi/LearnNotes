# Azure 防火墙介绍与部署

## Azure 防火墙介绍

### 简介

Azure 防火墙是托管的基于云的网络安全服务，可保护 Azure 虚拟网络资源。 它是一个服务形式的完全有状态防火墙，具有内置的高可用性和不受限制的云可伸缩性。

<img src="https://i.loli.net/2021/09/18/zjhTaNsRtCk2rB4.png" width=600 />

>可以跨订阅和虚拟网络集中创建、实施和记录应用程序与网络连接策略。 Azure 防火墙对虚拟网络资源使用静态公共 IP 地址，使外部防火墙能够识别来自你的虚拟网络的流量。 该服务与用于日志记录和分析的 Azure Monitor 完全集成。

**Azure 防火墙高级版介绍**：

Azure 防火墙高级版使用防火墙策略，这是一项可用来通过 Azure 防火墙管理器集中管理防火墙的全局资源。 从此版本开始，所有新功能都只能通过防火墙策略进行配置。 防火墙规则（经典）继续受支持，并可用于配置现有标准防火墙功能。 防火墙策略可以独立管理，也可以通过 Azure 防火墙管理器进行管理。 与单个防火墙关联的防火墙策略不会产生任何费用。

<img src="https://i.loli.net/2021/09/18/InwO4AbXBUFjpPM.png" width=600 />

Azure 防火墙高级版包括以下功能：

* TLS 检查 - 解密出站流量、处理数据，然后加密数据并将数据发送到目的地。
* IDPS - 网络入侵检测和防护系统 (IDPS) 使你可以监视网络活动是否出现恶意活动，记录有关此活动的信息，予以报告，选择性地尝试阻止。
* URL 筛选 - 扩展 Azure 防火墙的 FQDN 筛选功能，以涵盖整个 URL。 例如，www.contoso.com/a/c，而非 www.contoso.com。
* Web 类别 - 管理员可以允许或拒绝用户访问某些类别的网站，例如赌博网站、社交媒体网站等。

## 部署和配置

### 部署架构

<img src="https://i.loli.net/2021/09/18/wDl93861r4CKvSk.png" width=600 />

### 一.创建和配置网络

#### 1.创建Hub虚拟网络

```powershell
# 创建虚拟网络
$Hubvnet = @{
    Name = 'DemoHubVNet'
    ResourceGroupName = 'Demo-RG'
    Location = 'chinaeast2'
    AddressPrefix = '10.1.0.0/16'
    Tag = @{
        enviroment = "demo"
        create_user = "baoxi"
    }
}
$HubvirtualNetwork = New-AzVirtualNetwork @Hubvnet
# 添加子网
$bastionSubnet = @{
    Name = 'BastionSubnet'
    VirtualNetwork = $HubvirtualNetwork
    AddressPrefix = '10.1.0.0/24'
}
$GatewaySubnet = @{
    Name = 'GatewaySubnet'
    VirtualNetwork = $HubvirtualNetwork
    AddressPrefix = '10.1.1.0/24'
}
$AzureFirewallSubnet = @{
    Name = 'AzureFirewallSubnet'
    VirtualNetwork = $HubvirtualNetwork
    AddressPrefix = '10.1.2.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @bastionSubnet
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @GatewaySubnet
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @AzureFirewallSubnet
# 将子网关联到虚拟网络
$HubvirtualNetwork | Set-AzVirtualNetwork
```

#### 2.创建spoke虚拟网络

```powershell
#创建spocke虚拟网络
$spokevnet = @{
    Name = 'DemoSpokeVNet'
    ResourceGroupName = 'Demo-RG'
    Location = 'chinaeast2'
    AddressPrefix = '10.2.0.0/16'
    Tag = @{
        enviroment = "demo"
        create_user = "baoxi"
    }
}
$SpokevirtualNetwork = New-AzVirtualNetwork @spokevnet
#创建spoke虚拟网络子网
$WebSubnet = @{
    Name = 'WebSubnet'
    VirtualNetwork = $SpokevirtualNetwork
    AddressPrefix = '10.2.1.0/24'
}
$AppSubnet = @{
    Name = 'AppSubnet'
    VirtualNetwork = $SpokevirtualNetwork
    AddressPrefix = '10.2.2.0/24'
}
$DBSubnet = @{
    Name = 'DBSubnet'
    VirtualNetwork = $SpokevirtualNetwork
    AddressPrefix = '10.2.3.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @WebSubnet
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @AppSubnet
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @DBSubnet

#将子网关联到虚拟网络
$SpokevirtualNetwork | Set-AzVirtualNetwork
```

#### 3. 创建对等互联

```powershell
#将DemoHubVnet对等互联到DemoSpokeVnet
Add-AzVirtualNetworkPeering -Name DemoHubVnet-DemoSpokeVnet -VirtualNetwork $HubvirtualNetwork -RemoteVirtualNetworkId $SpokevirtualNetwork.Id
#将DemoSpokeVnet对等互联到DemoHubVnet
Add-AzVirtualNetworkPeering -Name DemoSpokeVnet-DemoHubVnet -VirtualNetwork $SpokevirtualNetwork -RemoteVirtualNetworkId $HubvirtualNetwork.Id
#检查连接状态
Get-AzVirtualNetworkPeering -ResourceGroupName Demo-RG -VirtualNetworkName DemoHubVnet | Select PeeringState
```

### 二、创建Azure Firewall

1. 选择**Firewall**然后点击**创建**

    <img src="https://i.loli.net/2021/09/22/VQvYWT6UK9EGmBn.png" width=600 />

2. 在**创建防火墙**页上，使用下图配置防火墙：

    <img src="https://i.loli.net/2021/09/22/RfizrqnLvGxXHDI.png" width=600 />
    <img src="https://i.loli.net/2021/09/22/cbpovOtENjyHJRu.png" width=600 />
    <img src="https://i.loli.net/2021/09/22/sXCYwIovr3pMKOQ.png" width=600 />

3. 在**标记**页上，输入相应键和值，然后点击**下一步：查看+创建**
    <img src="https://i.loli.net/2021/09/22/BKvce7aTGq9DRLQ.png" width=600 />

4. 点击**创建**
    <img src="https://i.loli.net/2021/09/22/mN95Bead6vRsgCO.png" width=600 />

### 三、创建默认路由

1. 在市场上搜索**路由表**，然后点击**创建**
    <img src="https://i.loli.net/2021/09/22/6gaweIjkxWfS41H.png" width=600 />
2. 在创建路由表页面，填写如下信息，然后点击**创建**
    <img src="https://i.loli.net/2021/09/22/4ImOQyZxlj19qS5.png" width=600 />
3. 在**防火墙路由**页上，选择**子网**，然后选择**关联**
    <img src="https://i.loli.net/2021/09/22/yWAxCmi8zGh54Jo.png" width=600 />
4. 添加路由，然后填写如图信息，下一个跃点类型，选择**虚拟设备**，下一个跃点地址，填写防火墙专用IP地址。
    <img src="https://i.loli.net/2021/09/22/kZSWnAq9wtuNF2b.png" width=600 />

### 四、配置应用程序规则

本例，配置允许访问www.baidu.com应用程序规则

1. 打开fw-test防火墙策略，然后点击**Application Rules**，点击**添加规则集合**

    <img src="https://i.loli.net/2021/09/22/DmL8AW3CUq49HcO.png" width=600 />
2. 在添加规则集合页面，然后配置如图所示，点击**添加**
    <img src="https://i.loli.net/2021/09/22/XvVUnHOC6Djyx3I.png" width=600 />

### 五、配置DNAT规则

此规则允许通过防火墙将远程桌面连接到虚拟机

1. 选择**DNAT 规则**,选择**添加规则集合**

    <img src="https://i.loli.net/2021/09/22/w3YxvFO9r5kgSf1.png" width=600 />

2. 填写规则集合信息，然后点击**添加**

    <img src="https://i.loli.net/2021/09/22/ZG4L9AjFXfcKhpD.png" width=600 />

### 六、测试防火墙

1. 将远程桌面连接到防火墙公共 IP 地址，并登录到虚拟机。
    <img src="https://i.loli.net/2021/09/22/YvV9keyO8aBGmiR.png" width=600 />
2. 浏览到 <https://www.microsoft.com>，防火墙会阻止

    <img src="https://i.loli.net/2021/09/22/t3eiYa5ch9ubdqz.png" width=600 />
3. 浏览到<https://www.baidu.com>,正常访问。

    <img src="https://i.loli.net/2021/09/22/n1klEZ8jQwx45bq.png" width=600 />
