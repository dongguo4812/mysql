# MySQL8

## 安装mysql

### 下载mysql

Linux系统下安装软件的常用三种方式：

方式1：rpm命令 使用rpm命令安装扩展名为".rpm"的软件包。

方式2：yum命令 需联网，从 互联网获取 的yum源，直接使用yum命令安装。

方式3：编译安装源码包 针对 tar.gz 这样的压缩格式，要用tar命令来解压；如果是其它压缩格式，就使用其它命令。

这里我们使用rpm方式

官网地址：https://dev.mysql.com/downloads/mysql/

![image-20240522113025731](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913752.png)

选择对应版本，这里进去默认是最新的版本，如果想要下载之前的版本的话，可以点击旁边的**Archives**里面查找，下载**RPM Bundle**版本。

![image-20240522113234267](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913194.png)

可以下载完成后使用Xftp等工具传输到自己安装的路径下，

或者直接使用wget命令进行下载。

```shell
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.35-1.el7.x86_64.rpm-bundle.tar
```

将下载的mysql放在/opt目录下

![image-20240522114527458](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110904532.png)

### 解压文件

```shell
tar -xvf mysql-8.0.35-1.el7.x86_64.rpm-bundle.tar
```

![image-20240522133251529](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110904357.png)

### 使用rpm 安装

必须按照顺序执行命令，否则会出现依赖错误的报错

```shell
rpm -ivh mysql-community-common-8.0.35-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.35-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.35-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.35-1.el7.x86_64.rpm
rpm -ivh mysql-community-icu-data-files-8.0.35-1.el8.x86_64.rpm
rpm -ivh mysql-community-server-8.0.35-1.el7.x86_64.rpm
```

1. **mysql-community-common**: 这个包包含 MySQL 的常用文件，其他包会依赖它。
2. **mysql-community-client-plugins**: 这个包包含 MySQL 客户端插件。
3. **mysql-community-libs**: 这个包包含 MySQL 的共享库。
4. **mysql-community-client**: 这个包包含 MySQL 客户端工具。
5. **mysql-community-icu-data-files**: 提供 ICU 数据文件，主要用于支持国际化。
6. **mysql-community-server**: 这个包包含 MySQL 服务器。

其他包如 `mysql-community-devel`, `mysql-community-embedded-compat`, `mysql-community-libs-compat`, `mysql-community-test`, 和 `mysql-community-server-debug` 可以根据需要安装。

![image-20240522135515488](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110904585.png)

#### 卸载mariadb

其中linux默认安装了mariadb，该软件与 MySQL 数据库有冲突，需要卸载

```shell
[root@dongguo opt]# rpm -qa | grep mariadb
mariadb-libs-5.5.68-1.el7.x86_64
[root@dongguo opt]# rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64
```

### 查看已安装的 MySQL 的版本

```
mysql --version
```

![image-20240522135849403](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907905.png)

执行如下命令，查看这些包是否安装成功。需要增加 -i 不用去区分大小写，否则搜索不到。

```shell
rpm -qa|grep -i mysql
```

![image-20240522140303971](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907100.png)

## 服务的初始化

为了保证数据库目录与文件的所有者为 mysql 登录用户，如果你是以 root 身份运行 mysql 服务，需要执 行下面的命令初始化：

```shell
mysqld --initialize --user=mysql
```

说明： --initialize 选项默认以“安全”模式来初始化，则会为 root 用户生成一个密码并将 该密码标记为过 期 ，登录后你需要设置一个新的密码。生成的 临时密码 会往日志中记录一份。

查看密码：

```shell
cat /var/log/mysqld.log
#临时密码
je)ea67,7s#G
```

![image-20240522141015661](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907072.png)

##  启动MySQL，查看状态

```shell
#启动：
systemctl start mysqld
#关闭：
systemctl stop mysqld
#重启：
systemctl restart mysqld
#查看状态：
systemctl status mysqld
```

![image-20240522141345430](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914784.png)

### MySQL服务开机自启动

查看MySQL服务是否自启动

```shell
systemctl list-unit-files|grep mysqld
```

![image-20240522142041270](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906973.png)

mysqld.service默认是enabled。

如不是enabled可以运行如下命令设置自启动

设置自启动

```shell
systemctl enable mysqld
```

取消自启动

```shell
systemctl disable mysqld
```

## MySQL登录

```shell
mysql -uroot -p
```

![image-20240522142651971](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906450.png)

密码为 je)ea67,7s#G

### 修改密码

因为初始化密码默认是过期的，所以查看数据库会报错 。修改密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root';
```

![image-20240522145527227](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914454.png)

在 MySQL 8.0 版本中，默认的身份验证插件被更改为 `caching_sha2_password`，许多旧版本的客户端、库和应用程序可能尚未更新以支持 `caching_sha2_password` 插件。如果应用程序无法进行身份验证，可能会导致连接问题。

这个命令的作用是将 `root` 用户的身份验证插件更改为 `mysql_native_password`，并将密码设置为 `root`。

## 设置远程连接

Mysql默认配置不支持远程连接，只允许本机localhost连接。

```sql
use mysql;
select Host,User from user;
```

![image-20240522144824836](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110912661.png)

修改Host为通配符%， Host=% ，表示所有IP都有连接权限。

注意：在生产环境下不能为了省事将host设置为%，这样做会存在安全问题，具体的设置可以根据生产 环境的IP进行设置。

```sql
update user set host = '%' where user ='root';
```

Host修改完成后记得执行flush privileges使配置立即生效：

```sql
flush privileges;
```

![image-20240522145613519](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110912197.png)

### 远程连接

![image-20240522145751444](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110912097.png)

# MySQL5.7

## 安装mysql

官网地址：https://dev.mysql.com/downloads/mysql/

![image-20240522151324490](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913019.png)

或者使用wget命令进行下载。

```shell
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar
```

将下载的mysql放在/opt目录下

![image-20240522152527468](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913529.png)

### 解压文件

```shell
tar -xvf mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar
```

![image-20240522152653692](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913339.png)

### 使用rpm 安装

#### 卸载mariadb

其中linux默认安装了mariadb，该软件与 MySQL 数据库有冲突，需要卸载

```shell
[root@dongguo opt]# rpm -qa | grep mariadb
mariadb-libs-5.5.68-1.el7.x86_64
[root@dongguo opt]# rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64
```



必须按照顺序执行命令，否则会出现依赖错误的报错

```shell
rpm -ivh mysql-community-common-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.28-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.28-1.el7.x86_64.rpm
```

![image-20240522153124949](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913607.png)

### 查看已安装的 MySQL 的版本

```
mysql --version
```

![image-20240522153139788](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913250.png)

执行如下命令，查看这些包是否安装成功。需要增加 -i 不用去区分大小写，否则搜索不到。

```shell
rpm -qa|grep -i mysql
```

![image-20240522153219459](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110913877.png)

## 服务的初始化

为了保证数据库目录与文件的所有者为 mysql 登录用户，如果你是以 root 身份运行 mysql 服务，需要执 行下面的命令初始化：

```shell
mysqld --initialize --user=mysql
```

说明： --initialize 选项默认以“安全”模式来初始化，则会为 root 用户生成一个密码并将 该密码标记为过 期 ，登录后你需要设置一个新的密码。生成的 临时密码 会往日志中记录一份。

查看密码：

```shell
cat /var/log/mysqld.log
#临时密码
jm6y;qGw.k_z
```

![image-20240522153607909](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914297.png)

## 启动MySQL，查看状态

```shell
#启动：
systemctl start mysqld
#关闭：
systemctl stop mysqld
#重启：
systemctl restart mysqld
#查看状态：
systemctl status mysqld
```

![image-20240522153654083](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914238.png)

### MySQL服务开机自启动

查看MySQL服务是否自启动

```shell
systemctl list-unit-files|grep mysqld
```

![image-20240522153719685](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914375.png)

mysqld.service默认是enabled。

如不是enabled可以运行如下命令设置自启动

设置自启动

```shell
systemctl enable mysqld
```

取消自启动

```shell
systemctl disable mysqld
```

## MySQL登录

```shell
mysql -uroot -p
```

密码为 jm6y;qGw.k_z

![image-20240522153843267](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110904534.png)

### 修改密码

因为初始化密码默认是过期的，所以查看数据库会报错 。修改密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
```

![image-20240522154048009](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110904200.png)

## 设置远程连接

Mysql默认配置不支持远程连接，只允许本机localhost连接。

```sql
use mysql;
select Host,User from user;
```

![image-20240522154155625](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914692.png)

修改Host为通配符%， Host=% ，表示所有IP都有连接权限。

注意：在生产环境下不能为了省事将host设置为%，这样做会存在安全问题，具体的设置可以根据生产 环境的IP进行设置。

```sql
update user set host = '%' where user ='root';
```

Host修改完成后记得执行flush privileges使配置立即生效：

```sql
flush privileges;
```

![image-20240522154229050](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110914408.png)

### 远程连接

![image-20240522154306173](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110915831.png)



