# MySQL8的主要目录结构 

安装好MySQL 8之后，我们查看如下的目录结构： 

```shell
find / -name mysql
```

![image-20240523100455680](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110905803.png)



## 数据库文件的存放路径 

MySQL数据库文件的存放路径：/var/lib/mysql/ 

```sql
mysql> show variables like 'datadir';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+
1 row in set (0.01 sec)
```

## 相关命令目录

相关命令目录：/usr/bin（mysqladmin、mysqlbinlog、mysqldump等命令）和/usr/sbin。

![image-20240523103331954](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110905979.png)





![image-20240523103355789](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906728.png)

## 配置文件目录

配置文件目录：/usr/share/mysql-8.0（命令及配置文件），/etc（如my.cnf）

![image-20240523103224806](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906999.png)



![image-20240523103537697](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906798.png)

# 数据库和文件系统的关系

## 查看默认数据库

```sql
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```

可以看到有4个数据库是属于MySQL自带的系统数据库。

**mysql** 

MySQL 系统自带的核心数据库，它存储了MySQL的用户账户和权限信息，一些存储过程、事件的定 义信息，一些运行过程中产生的日志信息，一些帮助信息以及时区信息等。

**information_schema** 

MySQL 系统自带的数据库，这个数据库保存着MySQL服务器 维护的所有其他数据库的信息 ，比如有 哪些表、哪些视图、哪些触发器、哪些列、哪些索引。这些信息并不是真实的用户数据，而是一些 描述性信息，有时候也称之为 元数据 。在系统数据库 information_schema 中提供了一些以 innodb_sys (mysql8提供了一些新的视图来代替这些旧的 `innodb_sys` 表，以 `INNODB_` 开头)开头的表，用于表示内部系统表。

mysql5.7

```sql
mysql> USE information_schema;
Database changed
mysql> SHOW TABLES LIKE 'INNODB_SYS%';
+--------------------------------------------+
| Tables_in_information_schema (INNODB_SYS%) |
+--------------------------------------------+
| INNODB_SYS_DATAFILES                       |
| INNODB_SYS_VIRTUAL                         |
| INNODB_SYS_INDEXES                         |
| INNODB_SYS_TABLES                          |
| INNODB_SYS_FIELDS                          |
| INNODB_SYS_TABLESPACES                     |
| INNODB_SYS_FOREIGN_COLS                    |
| INNODB_SYS_COLUMNS                         |
| INNODB_SYS_FOREIGN                         |
| INNODB_SYS_TABLESTATS                      |
+--------------------------------------------+
10 rows in set (0.00 sec)
```

mysql8

```sql
mysql> USE information_schema;
Database changed
mysql> SHOW TABLES LIKE 'innodb%';
+----------------------------------------+
| Tables_in_information_schema (INNODB%) |
+----------------------------------------+
| INNODB_BUFFER_PAGE                     |
| INNODB_BUFFER_PAGE_LRU                 |
| INNODB_BUFFER_POOL_STATS               |
| INNODB_CACHED_INDEXES                  |
| INNODB_CMP                             |
| INNODB_CMPMEM                          |
| INNODB_CMPMEM_RESET                    |
| INNODB_CMP_PER_INDEX                   |
| INNODB_CMP_PER_INDEX_RESET             |
| INNODB_CMP_RESET                       |
| INNODB_COLUMNS                         |
| INNODB_DATAFILES                       |
| INNODB_FIELDS                          |
| INNODB_FOREIGN                         |
| INNODB_FOREIGN_COLS                    |
| INNODB_FT_BEING_DELETED                |
| INNODB_FT_CONFIG                       |
| INNODB_FT_DEFAULT_STOPWORD             |
| INNODB_FT_DELETED                      |
| INNODB_FT_INDEX_CACHE                  |
| INNODB_FT_INDEX_TABLE                  |
| INNODB_INDEXES                         |
| INNODB_METRICS                         |
| INNODB_SESSION_TEMP_TABLESPACES        |
| INNODB_TABLES                          |
| INNODB_TABLESPACES                     |
| INNODB_TABLESPACES_BRIEF               |
| INNODB_TABLESTATS                      |
| INNODB_TEMP_TABLE_INFO                 |
| INNODB_TRX                             |
| INNODB_VIRTUAL                         |
+----------------------------------------+
31 rows in set (0.00 sec)
```



**performance_schema** 

MySQL 系统自带的数据库，这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，可以 用来 监控 MySQL 服务的各类性能指标 。包括统计最近执行了哪些语句，在执行过程的每个阶段都 花费了多长时间，内存的使用情况等信息。

**sys** 

MySQL 系统自带的数据库，这个数据库主要是通过 视图 的形式把 information_schema 和 performance_schema 结合起来，帮助系统管理员和开发人员监控 MySQL 的技术性能。

## 数据库在文件系统中的表示

![image-20240523105413185](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906226.png)

/var/lib/mysql这个数据目录下的文件和子目录比较多，除了 information_schema 这个系统数据库外，其他的数据库 在 数据目录 下都有对应的子目录。

以db1数据库为例，在MySQL5.7 中打开：

![image-20240523105614613](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906316.png)

1. **`db.opt`**:
   - **描述**: 这个文件包含数据库的选项和设置。通常，这包括默认字符集和排序规则等设置。
   - **作用**: 它确保当你访问数据库时，使用正确的字符集和排序规则。
2. **`t_emp.frm`**:
   - **描述**: 这是表结构文件，用于存储表的定义（表结构）。
   - **作用**: 包含表的元数据，例如列的名称、类型、索引等。在 MySQL 8 之前的版本中，这个文件用于所有存储引擎。但是，从 MySQL 8 开始，InnoDB 内部也存储了表定义，因此 `.frm` 文件在新版本中已不再是必须的。
3. **`t_emp.ibd`**:
   - **描述**: 这是 InnoDB 表空间文件，存储实际的表数据和索引数据。
   - **作用**: 包含表的所有行数据和索引信息。InnoDB 会将每个表的数据存储在单独的 `.ibd` 文件中（假设启用了 innodb_file_per_table 设置）。这个文件通常会包含所有与该表相关的数据和索引页。

在MySQL8.0中打开：

![image-20240523105641040](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110906312.png)

`.ibd` 文件是 InnoDB 存储引擎使用的独立表空间文件，它们包含了表的数据和索引。

## 表在文件系统中的表示

### InnoDB存储引擎模式

#### 表结构

为了保存表结构， InnoDB 在 数据目录 下对应的数据库子目录下创建了一个专门用于 描述表结构的文 件 ，文件名是这样：

`表名.frm`

.frm文件 的格式在不同的平台上都是相同的。这个后缀名为.frm是以 二进制格式 存储的，我们直接打开是乱码 的。

#### 表中数据和索引

① 系统表空间（system tablespace）

默认情况下，InnoDB会在数据目录下创建一个名为 ibdata1 、大小为 12M 的文件，这个文件就是对应 的 系统表空间 在文件系统上的表示。怎么才12M？注意这个文件是 自扩展文件 ，当不够用的时候它会自 己增加文件大小。

当然，如果你想让系统表空间对应文件系统上多个实际文件，或者仅仅觉得原来的 ibdata1 这个文件名 难听，那可以在MySQL启动时配置对应的文件路径以及它们的大小，比如我们这样修改一下my.cnf 配置 文件：

```shell
[server]
innodb_data_file_path=data1:512M;data2:512M:autoextend
```

② 独立表空间(file-per-table tablespace)

在MySQL5.6.6以及之后的版本中，InnoDB并不会默认的把各个表的数据存储到系统表空间中，而是为 每 一个表建立一个独立表空间 ，也就是说我们创建了多少个表，就有多少个独立表空间。使用 独立表空间 来 存储表数据的话，会在该表所属数据库对应的子目录下创建一个表示该独立表空间的文件，文件名和表 名相同，只不过添加了一个 .ibd 的扩展名而已，所以完整的文件名称长这样：`表名.ibd`

③ 系统表空间与独立表空间的设置

我们可以自己指定使用 系统表空间 还是 独立表空间 来存储数据，这个功能由启动参数 innodb_file_per_table 控制，比如说我们想刻意将表数据都存储到 系统表空间 时，可以在启动 MySQL服务器的时候这样配置my.cnf ：

```shell
[server]
innodb_file_per_table=0 # 0：代表使用系统表空间； 1：代表使用独立表空间
```

默认情况： 使用独立表空间

```sql
mysql> show variables like 'innodb_file_per_table';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.01 sec)
```

### MyISAM存储引擎模式

####  表结构

在存储表结构方面， MyISAM 和 InnoDB 一样，也是在 数据目录 下对应的数据库子目录下创建了一个专 门用于描述表结构的文件：`表名.frm`

####  表中数据和索引

在MyISAM中的索引全部都是 二级索引 ，该存储引擎的 数据和索引是分开存放 的。所以在文件系统中也是 使用不同的文件来存储数据文件和索引文件，同时表数据都存放在对应的数据库子目录下。

假如 test 表使用MyISAM存储引擎的话，那么在它所在数据库对应的 atguigu 目录下会为 test 表创建这三个文 件：

```shell
test.frm 存储表结构
test.MYD 存储数据 (MYData)
test.MYI 存储索引 (MYIndex)
```

## 视图在文件系统中的表示

我们知道MySQL中的视图其实是虚拟的表,也就是某个查询语句的一个别名而已,所以在存储视图的时候是不需要存储真实的数据的,只需要把它的结构存储起来就行了。和表一样,描述视图结构的文件也会被存储到所属数据库对应的子目录下边,只会存储一个视图名.frm的文件。