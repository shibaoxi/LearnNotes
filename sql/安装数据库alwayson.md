# 安装SQL Server AlwaysON

## 前提条件

* sql 节点都加入域中
* 使用域账户登录到sql服务器的操作系统中（该账户拥有本地管理员权限）
* 创建一个服务账户(普通域账户，建议密码策略设置永不过期)
* 需要提供一个群集名称和一个群集的IP地址

---

## 部署实施

### 安装群集功能

<div align = "center" ><img src = https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114120343242.png width=600 height=300 /></div>

​![image-20210114120343242](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114120343242.png)

> 每个数据库节点都要安装

### 配置群集

1. 打开群集管理器，点击新建群集

    ![image-20210114121241077](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114121241077.png)

2. 在选择服务器界面，把要加入sql 群集节点的服务器都加进来。

   ![image-20210114121535518](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114121535518.png)

3. 为了保障群集的健康稳定性，先验证群集是否符合条件。

   ![image-20210114121938346](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114121938346.png)

4. 选择运行所有的测试

   ![image-20210114122102557](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114122102557.png)

5. 浏览要测试的项目，然后点击下一步。

   ![image-20210114122216405](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114122216405.png)

6. 查看测试报告，解决一些有问题的地方，如果测试通过可进行下一步。

   ![image-20210114122654484](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114122654484.png)

7. 输入群集名称和群集IP地址

   ![image-20210114122848155](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114122848155.png)

8. 取消勾选“将所有符合条件的存储添加到群集”，然后点击下一步。

   ![image-20210114123118152](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114123118152.png)

   > 需要注意的是，如果你的SQL 计算机节点不是在默认的OU下面，需要对相应的OU添加“创建所有子对象”的权限，如图所示：
   > 【需要把你创建群集的域账号临时添加该OU的权限】
   > ![image-20210114131538219](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114131538219.png)

9. 创建群集

   ![image-20210114131845926](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114131845926.png)

---

### 安装数据库

> 每个节点安装数据库

1. 选择相应的功能和安装路径

   ![image-20210114134342963](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114134342963.png)

2. 选择默认实例或者自定义实例，这里我们选择默认实例

   ![image-20210114134450001](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114134450001.png)

3. 在SQL Server 数据库引擎服务上输入服务账户名称和密码，排序规则如需要改动则在这里修改。

   > 注意：该服务账户要加入本地管理员组中

   ![image-20210114134730893](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114134730893.png)

4. 服务器配置填上相应的账户

   ![image-20210114141552370](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114141552370.png)

5. 点击安装

   ![image-20210114142135823](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114142135823.png)

---

### 配置SQL Always On

1. 打开Sql server configuration manager，启用SQL Always On

   > 每个节点都需要此操作

   ![image-20210114152845346](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114152845346.png)

2. 重启该服务

   ![image-20210114153103945](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114153103945.png)

3. 新建可用性组，打开新建可用性组向导

   ![image-20210114153539412](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114153539412.png)

4. 输入可用性组名称，以及相关选项

   > * 数据库级别运行状况检测：启用该选项则可检测到数据库不再处于联机状态以及其他问题，然后将触发可用性组的自动故障转移(该选项针对与该可用性组下面的所有数据库，不能单独指定数据库)，需要注意的是该选项不会监视SQL Server 磁盘运行时间，并且SQL Server 不会直接监视数据库文件可用性，如果磁盘驱动器故障或者不可用，单独这一问题不一定会触发可用性组自动故障转移。
   > * SQL Server 2016 (13.x) 服务包 2 及更高版本完全支持可用性组中的分布式事务。 在服务包 2 之前的 SQL Server 2016 (13.x)!INCLUDE[SQL2016] 版本中，不支持涉及可用性组中的数据库的跨数据库分布式事务（即，使用同一 SQL Server 实例上的数据库的事务）。 SQL Server 2017 (14.x) 没有此限制。

   ![image-20210114154504971](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114154504971.png)

5. 添加数据库到可用性组中

   ![image-20210114155912699](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114155912699.png)

6. 添加副本节点，然后选择自动故障转移

   ![image-20210114160230633](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114160230633.png)

7. 选择自动种子设定

   ![image-20210114160433452](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114160433452.png)

8. 验证成功后，点击下一步

   ![image-20210114160524043](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114160524043.png)

9. 添加侦听器

   ![image-20210114160642424](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114160642424.png)

10. 输入侦听器DNS名称和IP地址

   > 注意：该dns名称会向AD中写入一个计算机对象，请确保有相应的写入权限。

      ![image-20210114160936548](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114160936548.png)

11. telnet 到192.168.11.6的1433端口

      ``` telnet 192.168.11.6 1433 ```

      ![image-20210114161708025](https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/image-20210114161708025.png)
