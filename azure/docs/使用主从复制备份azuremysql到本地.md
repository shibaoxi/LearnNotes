# 使用主从复制实时备份Azure database for MySQL到本地MySQL服务器

## 先决条件

- 本地数据库版本和Azure Database for MySQL版本一致
- 目标服务器应使用 MySQL InnoDB 引擎
- 请确本地服务器的 IP 地址已添加到 Azure Database for MySQL 副本服务器的防火墙规则中
- 请确保托管源服务器的计算机在端口 3306 上允许入站和出站流量。

## 配置主从复制

> 用户帐户不会从源服务器复制到副本服务器。 如果打算为用户提供访问副本服务器的权限，则需在这个副本服务器上手动创建所有帐户和相应的特权。

### 查看是否启用二进制日志记录

```sql
SHOW VARIABLES LIKE 'log_bin';
```

如果返回了值为“ON”的变量 log_bin，则表示已在服务器上启用了二进制日志记录。

### 查看lower_case_table_names是否为1

```sql
SHOW VARIABLES LIKE 'lower_case_%';
```

#### 设置lower_case_table_names

由于lower_case_table_names只能在初始化安装时才能设置，如果要更改，则必须重新进行安装。

```bash
# 备份mysqld.cnf 文件
cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.backup
# 停止服务
service mysql stop
# 删除mysql
apt-get --purge autoremove mysql-server
# 删除数据文件
rm -rf /var/lib/mysql*
# 还原配置文件
cp /etc/mysql/mysql.conf.d/mysqld.cnf.backup /etc/mysql/mysql.conf.d/mysqld.cnf
# 编辑配置文件
vim /etc/mysql/mysql.conf.d/mysqld.cnf
# 添加下面条目
lower_case_table_names=1
# 安装mysql
apt-get install mysql-server
# 启动服务
service mysql start
# 初始化配置
mysql_secure_installation
```

### 创建用于专门主从复制用的用户

主库创建

```sql
CREATE USER 'syncuser'@'%' IDENTIFIED BY 'password01!';
GRANT REPLICATION SLAVE ON *.* TO ' syncuser'@'%';
```

### 将源服务器设置为只读模式

```sql
FLUSH TABLES WITH READ LOCK;
```

### 获取二进制日志文件名和偏移量

```sql
show master status;
```

结果应如下所示。 请务必记下二进制文件名和偏移量，以供在后续步骤中使用。

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220331193840.png" width=600 />

### 转储并还原源服务器

使用mysqldump备份数据库文件

```bash
mysqldump --column-statistics=0 --databases $DBNAMES -h yourdatabase.mysql.database.chinacloudapi.cn -u dbuser@dbservername -p$MYPASSWORD  > back$date.bak
```

将转储文件还原到新服务器。

```bash
mysql -h mysqlhost -u <user>@<servername> -p testdb < back1647329237.bak
```

### 配置复制

自8.0.23版本后的语法：

```sql
CHANGE REPLICATION SOURCE TO SOURCE_HOST='mysqltest.mysql.database.chinacloudapi.cn',SOURCE_PORT=3306,SOURCE_USER='syncuser',SOURCE_PASSWORD='password01!',SOURCE_LOG_FILE='mysql-bin.000001',SOURCE_LOG_POS=3834,SOURCE_SSL=1;
```

最后开启主从复制工作

- 低于8.0.22版本的语法：

    ```bash
    START SLAVE;
    ```

- 自8.0.22版本后的语法：

    ```bash
    START REPLICA;
    ````

可执行命令查看详细信息以及状态

- 低于8.0.22版本的语法：

    ```bash
    SHOW SLAVE STATUS;
    ```

- 自8.0.22版本后的语法：

    ```bash
    SHOW REPLICA STATUS\G;
    ```

假如显示 Slave_IO_Running/Replica_IO_Running和 Slave_SQL_Running/Replica_SQL_Running 为 Yes ，以及Slave_IO_State/Replica_IO_State 为 Waiting for master to send event/Waiting for source to send event，则证明主从复制成功！

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220331200927.png" width=600 />

假如需要停止主从复制工作，则执行以下命令

- 低于8.0.22版本的语法：

    ```sql
    STOP SLAVE;
    ```

- 自8.0.22版本后的语法：

    ```sql
    STOP REPLICA;
    ```

假如需要重启主从复制工作，则执行以下命令

- 低于8.0.22版本的语法：

    ```sql
    RESTART SLAVE;
    ```

- 自8.0.22版本后的语法：

```sql
RESTART REPLICA;
```

清空复制工作

```sql
RESET REPLICA;
```

### 将源服务器设置改为读写模式

```sql
UNLOCK TABLES;
```
