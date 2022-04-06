# 使用mysqldump 备份和还原Azure Database for MySQL

## 备份 Azure Database for MySQL

### 安装mysql 客户端

```shell
sudo apt install mysql-client
```

### 编写脚本备份数据库

```shell
#!bin/bash
cd /home/miguel
export DBNAMES="tesdb"
export MYPASSWORD="yourpassword" 
date=$(date +%s)
year=$(date +%Y)
month=$(date +%m)
day=$(date +%d)
hour=$(date +%H)
path=$year/$month/$day/$hour
echo $date
cd /home/miguel/mysqlbackups/
mkdir -p $path
cd $path
mysqldump --column-statistics=0 --databases $DBNAMES -h yourdatabase.mysql.database.chinacloudapi.cn -u dbuser@dbservername -p$MYPASSWORD  > back$date.bak
```

### 定期执行备份任务

```shell
# 星期日，凌晨2点开始执行，注意运行系统上的时区
0 2 * * 0 sh /home/bx/backup.sh >> /home/bx/backup.log 
```

> 可以把备份文件存储在Azure storage中，一种是使用azure file挂在到linux系统中，另外一种是先备份的本地磁盘中，然后使用az copy 复制到azure blob存储中。

## 还原数据库

### 在目标区域创建 azure database for mysql

使用mysql 客户端连接到创建好的数据库服务器，创建数据库。

```shell
# 连接到mysql数据库
mysql -h <dbserver>.mysql.database.chinacloudapi.cn -u <user>@<servername> -p
# 创建数据库
create database suez charset=utf8;
```

### 恢复数据库

使用上面生成的备份文件备份数据库

```shell
mysql -h suezsql.mysql.database.azure.com -u <user>@<servername> -p testdb < back1647329237.bak
```

### 数据对比

```sql
# 查询总表的数量
show tables;
# 查询每个表的数量
select count(*) from tbl_attachment;
```
