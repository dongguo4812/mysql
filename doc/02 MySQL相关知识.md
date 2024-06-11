# mysql密码强度评估

##  MySQL设置密码(可能出现)

```sql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
```

MySQL 从 5.7 版本开始引入了密码策略管理，通过插件 `validate_password` 来确保密码的强度和安全性。这个插件对密码的长度、复杂性（如大小写字母、数字、特殊字符）等有严格的要求。

在 MySQL 8.0 中，默认配置通常更为严格，这可能导致您设置的密码不满足要求，从而导致错误。

### 解决方案1：确保您的密码符合策略要求

```sql
alter user 'root' identified by 'HelloWorld_123';
```

### 解决方案2：禁用 `validate_password` 插件

```sql
UNINSTALL PLUGIN validate_password;
```

## 安装/启用validate_password插件

### 方式1：在参数文件my.cnf中添加参数

/etc/my.cnf

```shell
[mysqld]
plugin-load-add=validate_password.so
\#ON/OFF/FORCE/FORCE_PLUS_PERMANENT: 是否使用该插件(及强制/永久强制使用)
validate-password=FORCE_PLUS_PERMANENT
```

说明1： plugin library中的validate_password文件名的后缀名根据平台不同有所差异。 对于Unix和 Unix-like系统而言，它的文件后缀名是.so，对于Windows系统而言，它的文件后缀名是.dll。

说明2： 修改参数后必须重启MySQL服务才能生效。 

说明3： 参数FORCE_PLUS_PERMANENT是为了防止插件在MySQL运行时的时候被卸载。当你卸载插件时就会报错。如下所示。

```sql
mysql> SELECT PLUGIN_NAME, PLUGIN_LIBRARY, PLUGIN_STATUS, LOAD_OPTION
-> FROM INFORMATION_SCHEMA.PLUGINS
-> WHERE PLUGIN_NAME = 'validate_password';
+-------------------+----------------------+---------------+----------------------+
| PLUGIN_NAME | PLUGIN_LIBRARY | PLUGIN_STATUS | LOAD_OPTION |
+-------------------+----------------------+---------------+----------------------+
| validate_password | validate_password.so | ACTIVE | FORCE_PLUS_PERMANENT |
+-------------------+----------------------+---------------+----------------------+
1 row in set (0.00 sec)
mysql> UNINSTALL PLUGIN validate_password;
ERROR 1702 (HY000): Plugin 'validate_password' is force_plus_permanent and can not be
unloaded
mysql>
```

### 方式2：运行时命令安装（推荐）

```sql
mysql> INSTALL PLUGIN validate_password SONAME 'validate_password.so';
Query OK, 0 rows affected, 1 warning (0.11 sec)
```

此方法也会注册到元数据，也就是mysql.plugin表中，所以不用担心MySQL重启后插件会失效。

#  MySQL的安全策略

## validate_password说明 

未安装validate_password插件前，执行如下指令 ，执行效果：

```sql
mysql>  show variables like 'validate_password%';
Empty set (0.00 sec)
```

安装插件后，执行如下指令 ，执行效果：

mysql5.7：

```sql
mysql> show variables like 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_check_user_name    | OFF    |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
7 rows in set (0.00 sec)
```

mysql8：

```sql
mysql> show variables like 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_check_user_name    | ON     |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
7 rows in set (0.02 sec)
```

关于 validate_password 组件对应的系统变量说明：

![image-20240522164426204](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907090.png)

提示： 组件和插件的默认值可能有所不同。例如，MySQL 5.7. validate_password_check_user_name的默认 值为OFF。

此外，我们还可以修改密码中字符的长度

```sql
set global validate_password_length=1
```

## 密码强度测试

如果你创建密码是遇到“Your password does not satisfy the current policy requirements”，可以通过函数组 件去检测密码是否满足条件： 0-100。当评估在100时就是说明使用上了最基本的规则：大写+小写+特殊 字符+数字组成的8位以上密码

```sql
mysql> SELECT VALIDATE_PASSWORD_STRENGTH('medium');
+--------------------------------------+
| VALIDATE_PASSWORD_STRENGTH('medium') |
+--------------------------------------+
|                                   25 |
+--------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT VALIDATE_PASSWORD_STRENGTH('K354*45jKd5');
+-------------------------------------------+
| VALIDATE_PASSWORD_STRENGTH('K354*45jKd5') |
+-------------------------------------------+
|                                       100 |
+-------------------------------------------+
1 row in set (0.00 sec)
```

注意：如果没有安装validate_password组件或插件的话，那么这个函数永远都返回0。 关于密码复杂度对 应的密码复杂度策略。如下表格所示：

![image-20240522164802107](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907634.png)

# 字符集的相关操作

## 修改MySQL5.7字符集

在MySQL 8.0版本之前，默认字符集为 latin1 ，utf8字符集指向的是 utf8mb3 。网站开发人员在数据库 设计的时候往往会将编码修改为utf8字符集。如果遗忘修改默认的编码，就会出现乱码的问题。从MySQL 8.0开始，数据库的默认编码将改为 utf8mb4 ，从而避免上述乱码的问题。

### 查看默认使用的字符集

MySQL 5.7 默认的客户端和服务器都用了 latin1 ，不支持中文，保存中文会报错。MySQL5.7如下：

```sql
mysql> show variables like 'character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

这是mysql8的字符集

```sql
mysql> show variables like 'character%';
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
+--------------------------+--------------------------------+
8 rows in set (0.00 sec)
```

在MySQL5.7中添加中文数据时，报错：

```sql
mysql> create database db1;
Query OK, 1 row affected (0.00 sec)

mysql> use db1;
Database changed
mysql> create table t_emp(id int,name varchar(15));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into t_emp(id,name)values(1001,"张三");
ERROR 1366 (HY000): Incorrect string value: '\xE5\xBC\xA0\xE4\xB8\x89' for column 'name' at row 1
```

因为默认情况下，创建表使用的是 latin1 。如下：

```sql
mysql> show create table t_emp;
+-------+------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                 |
+-------+------------------------------------------------------------------------------------------------------------------------------+
| t_emp | CREATE TABLE `t_emp` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(15) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+-------+------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

### 修改字符集

```shell
vim /etc/my.cnf
```

在MySQL5.7或之前的版本中，在文件最后加上中文字符集配置：

```shell
character_set_server=utf8
```

### 重新启动MySQL服务

```shell
systemctl restart mysqld
```

但是对已经创建的库、表的设定不会发生变化，参数修改只对新建的数据库生效。

### 已有库&表字符集的变更

MySQL5.7版本中，以前创建的库，创建的表字符集还是latin1。

```sql
mysql> show create table t_emp;
+-------+------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                 |
+-------+------------------------------------------------------------------------------------------------------------------------------+
| t_emp | CREATE TABLE `t_emp` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(15) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+-------+------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> show create database db1;
+----------+--------------------------------------------------------------+
| Database | Create Database                                              |
+----------+--------------------------------------------------------------+
| db1      | CREATE DATABASE `db1` /*!40100 DEFAULT CHARACTER SET latin1 */ |
+----------+--------------------------------------------------------------+
1 row in set (0.00 sec)
```

修改已创建数据库的字符集

```sql
alter database db1 character set 'utf8';
```

修改已创建数据表的字符集

```sql
alter table t_emp convert to character set 'utf8';
```

![image-20240522170549531](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907025.png)

再次添加中文数据

```sql
mysql> insert into t_emp(id,name)values(1001,"张三");
Query OK, 1 row affected (0.00 sec)

mysql> select * from t_emp;
+------+--------+
| id   | name   |
+------+--------+
| 1001 | 张三   |
+------+--------+
1 row in set (0.00 sec)
```

注意：但是已经存在的数据如果是用非'utf8'编码的话，数据本身编码不会发生改变。已有数据需要导出或删除，然后重新插入。

## 各级别的字符集

MySQL有4个级别的字符集和比较规则，分别是： 服务器级别、 数据库级别 、表级别 、列级别。

执行如下SQL语句：

```sql
mysql> show variables like 'character%';
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
+--------------------------+--------------------------------+
8 rows in set (0.00 sec)
```

- character_set_server：服务器级别的字符集 

- character_set_database：当前数据库的字符集 

- character_set_client：服务器解码请求时使用的字符集 

- character_set_connection：服务器处理请求时会把请求字符串从character_set_client转为 character_set_connection 

- character_set_results：服务器向客户端返回数据时使用的字符集

### 服务器级别

我们可以在启动服务器程序时通过启动选项或者在服务器程序运行过程中使用 SET 语句修改这两个变量 的值。

比如我们可以在配置文件中这样写：

```shell
[server]
character_set_server=gbk # 默认字符集
collation_server=gbk_chinese_ci #对应的默认的比较规则
```

当服务器启动的时候读取这个配置文件后这两个系统变量的值便修改了。

### 数据库级别

我们在创建和修改数据库的时候可以指定该数据库的字符集和比较规则，具体语法如下：

```sql
CREATE DATABASE 数据库名
[[DEFAULT] CHARACTER SET 字符集名称]
[[DEFAULT] COLLATE 比较规则名称];
ALTER DATABASE 数据库名
[[DEFAULT] CHARACTER SET 字符集名称]
[[DEFAULT] COLLATE 比较规则名称];
```

### 表级别

我们也可以在创建和修改表的时候指定表的字符集和比较规则，语法如下：

```sql
CREATE TABLE 表名 (列的信息)
[[DEFAULT] CHARACTER SET 字符集名称]
[COLLATE 比较规则名称]]
ALTER TABLE 表名
[[DEFAULT] CHARACTER SET 字符集名称]
[COLLATE 比较规则名称]
```

如果创建和修改表的语句中没有指明字符集和比较规则，将使用该表所在数据库的字符集和比较规则作 为该表的字符集和比较规则。

### 列级别

对于存储字符串的列，同一个表中的不同的列也可以有不同的字符集和比较规则。我们在创建和修改列 定义的时候可以指定该列的字符集和比较规则，语法如下：

```sql
CREATE TABLE 表名(
列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称],
其他列...
);
ALTER TABLE 表名 MODIFY 列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称];
```

对于某个列来说，如果在创建和修改的语句中没有指明字符集和比较规则，将使用该列所在表的字符集 和比较规则作为该列的字符集和比较规则。

提示：在转换列的字符集时需要注意，如果转换前列中存储的数据不能用转换后的字符集进行表示会发生 错误。比方说原先列使用的字符集是utf8，列中存储了一些汉字，现在把列的字符集转换为ascii的 话就会出错，因为ascii字符集并不能表示汉字字符。





# 字符集与比较规则(了解)

## utf8 与 utf8mb4

utf8 字符集表示一个字符需要使用1～4个字节，但是我们常用的一些字符使用1～3个字节就可以表示 了。而字符集表示一个字符所用的最大字节长度，在某些方面会影响系统的存储和性能，所以设计 MySQL的设计者偷偷的定义了两个概念：

utf8mb3 ：阉割过的 utf8 字符集，只使用1～3个字节表示字符。

utf8mb4 ：正宗的 utf8 字符集，使用1～4个字节表示字符。

## 比较规则

MySQL版本一共支持41种字符集，其中的 Default collation 列表示这种字符集中一种默认 的比较规则，里面包含着该比较规则主要作用于哪种语言，比如 utf8_polish_ci 表示以波兰语的规则 比较， utf8_spanish_ci 是以西班牙语的规则比较， utf8_general_ci 是一种通用的比较规则。

后缀表示该比较规则是否区分语言中的重音、大小写。具体如下：

![image-20240522182525086](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110907649.png)

最后一列 Maxlen ，它代表该种字符集表示一个字符最多需要几个字节。

常用操作1：

```sql
#查看GBK字符集的比较规则
SHOW COLLATION LIKE 'gbk%';

#查看UTF-8字符集的比较规则
SHOW COLLATION LIKE 'utf8%';



mysql> SHOW COLLATION LIKE 'gbk%';
+----------------+---------+----+---------+----------+---------+---------------+
| Collation      | Charset | Id | Default | Compiled | Sortlen | Pad_attribute |
+----------------+---------+----+---------+----------+---------+---------------+
| gbk_bin        | gbk     | 87 |         | Yes      |       1 | PAD SPACE     |
| gbk_chinese_ci | gbk     | 28 | Yes     | Yes      |       1 | PAD SPACE     |
+----------------+---------+----+---------+----------+---------+---------------+
2 rows in set (0.00 sec)

mysql> SHOW COLLATION LIKE 'utf8%';
+-----------------------------+---------+-----+---------+----------+---------+---------------+
| Collation                   | Charset | Id  | Default | Compiled | Sortlen | Pad_attribute |
+-----------------------------+---------+-----+---------+----------+---------+---------------+
| utf8mb3_bin                 | utf8mb3 |  83 |         | Yes      |       1 | PAD SPACE     |
| utf8mb3_croatian_ci         | utf8mb3 | 213 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_czech_ci            | utf8mb3 | 202 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_danish_ci           | utf8mb3 | 203 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_esperanto_ci        | utf8mb3 | 209 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_estonian_ci         | utf8mb3 | 198 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_general_ci          | utf8mb3 |  33 | Yes     | Yes      |       1 | PAD SPACE     |
| utf8mb3_general_mysql500_ci | utf8mb3 | 223 |         | Yes      |       1 | PAD SPACE     |
| utf8mb3_german2_ci          | utf8mb3 | 212 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_hungarian_ci        | utf8mb3 | 210 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_icelandic_ci        | utf8mb3 | 193 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_latvian_ci          | utf8mb3 | 194 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_lithuanian_ci       | utf8mb3 | 204 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_persian_ci          | utf8mb3 | 208 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_polish_ci           | utf8mb3 | 197 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_romanian_ci         | utf8mb3 | 195 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_roman_ci            | utf8mb3 | 207 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_sinhala_ci          | utf8mb3 | 211 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_slovak_ci           | utf8mb3 | 205 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_slovenian_ci        | utf8mb3 | 196 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_spanish2_ci         | utf8mb3 | 206 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_spanish_ci          | utf8mb3 | 199 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_swedish_ci          | utf8mb3 | 200 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_tolower_ci          | utf8mb3 |  76 |         | Yes      |       1 | PAD SPACE     |
| utf8mb3_turkish_ci          | utf8mb3 | 201 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_unicode_520_ci      | utf8mb3 | 214 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_unicode_ci          | utf8mb3 | 192 |         | Yes      |       8 | PAD SPACE     |
| utf8mb3_vietnamese_ci       | utf8mb3 | 215 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_0900_ai_ci          | utf8mb4 | 255 | Yes     | Yes      |       0 | NO PAD        |
| utf8mb4_0900_as_ci          | utf8mb4 | 305 |         | Yes      |       0 | NO PAD        |
| utf8mb4_0900_as_cs          | utf8mb4 | 278 |         | Yes      |       0 | NO PAD        |
| utf8mb4_0900_bin            | utf8mb4 | 309 |         | Yes      |       1 | NO PAD        |
| utf8mb4_bg_0900_ai_ci       | utf8mb4 | 318 |         | Yes      |       0 | NO PAD        |
| utf8mb4_bg_0900_as_cs       | utf8mb4 | 319 |         | Yes      |       0 | NO PAD        |
| utf8mb4_bin                 | utf8mb4 |  46 |         | Yes      |       1 | PAD SPACE     |
| utf8mb4_bs_0900_ai_ci       | utf8mb4 | 316 |         | Yes      |       0 | NO PAD        |
| utf8mb4_bs_0900_as_cs       | utf8mb4 | 317 |         | Yes      |       0 | NO PAD        |
| utf8mb4_croatian_ci         | utf8mb4 | 245 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_cs_0900_ai_ci       | utf8mb4 | 266 |         | Yes      |       0 | NO PAD        |
| utf8mb4_cs_0900_as_cs       | utf8mb4 | 289 |         | Yes      |       0 | NO PAD        |
| utf8mb4_czech_ci            | utf8mb4 | 234 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_danish_ci           | utf8mb4 | 235 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_da_0900_ai_ci       | utf8mb4 | 267 |         | Yes      |       0 | NO PAD        |
| utf8mb4_da_0900_as_cs       | utf8mb4 | 290 |         | Yes      |       0 | NO PAD        |
| utf8mb4_de_pb_0900_ai_ci    | utf8mb4 | 256 |         | Yes      |       0 | NO PAD        |
| utf8mb4_de_pb_0900_as_cs    | utf8mb4 | 279 |         | Yes      |       0 | NO PAD        |
| utf8mb4_eo_0900_ai_ci       | utf8mb4 | 273 |         | Yes      |       0 | NO PAD        |
| utf8mb4_eo_0900_as_cs       | utf8mb4 | 296 |         | Yes      |       0 | NO PAD        |
| utf8mb4_esperanto_ci        | utf8mb4 | 241 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_estonian_ci         | utf8mb4 | 230 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_es_0900_ai_ci       | utf8mb4 | 263 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_0900_as_cs       | utf8mb4 | 286 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_trad_0900_ai_ci  | utf8mb4 | 270 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_trad_0900_as_cs  | utf8mb4 | 293 |         | Yes      |       0 | NO PAD        |
| utf8mb4_et_0900_ai_ci       | utf8mb4 | 262 |         | Yes      |       0 | NO PAD        |
| utf8mb4_et_0900_as_cs       | utf8mb4 | 285 |         | Yes      |       0 | NO PAD        |
| utf8mb4_general_ci          | utf8mb4 |  45 |         | Yes      |       1 | PAD SPACE     |
| utf8mb4_german2_ci          | utf8mb4 | 244 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_gl_0900_ai_ci       | utf8mb4 | 320 |         | Yes      |       0 | NO PAD        |
| utf8mb4_gl_0900_as_cs       | utf8mb4 | 321 |         | Yes      |       0 | NO PAD        |
| utf8mb4_hr_0900_ai_ci       | utf8mb4 | 275 |         | Yes      |       0 | NO PAD        |
| utf8mb4_hr_0900_as_cs       | utf8mb4 | 298 |         | Yes      |       0 | NO PAD        |
| utf8mb4_hungarian_ci        | utf8mb4 | 242 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_hu_0900_ai_ci       | utf8mb4 | 274 |         | Yes      |       0 | NO PAD        |
| utf8mb4_hu_0900_as_cs       | utf8mb4 | 297 |         | Yes      |       0 | NO PAD        |
| utf8mb4_icelandic_ci        | utf8mb4 | 225 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_is_0900_ai_ci       | utf8mb4 | 257 |         | Yes      |       0 | NO PAD        |
| utf8mb4_is_0900_as_cs       | utf8mb4 | 280 |         | Yes      |       0 | NO PAD        |
| utf8mb4_ja_0900_as_cs       | utf8mb4 | 303 |         | Yes      |       0 | NO PAD        |
| utf8mb4_ja_0900_as_cs_ks    | utf8mb4 | 304 |         | Yes      |      24 | NO PAD        |
| utf8mb4_latvian_ci          | utf8mb4 | 226 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_la_0900_ai_ci       | utf8mb4 | 271 |         | Yes      |       0 | NO PAD        |
| utf8mb4_la_0900_as_cs       | utf8mb4 | 294 |         | Yes      |       0 | NO PAD        |
| utf8mb4_lithuanian_ci       | utf8mb4 | 236 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_lt_0900_ai_ci       | utf8mb4 | 268 |         | Yes      |       0 | NO PAD        |
| utf8mb4_lt_0900_as_cs       | utf8mb4 | 291 |         | Yes      |       0 | NO PAD        |
| utf8mb4_lv_0900_ai_ci       | utf8mb4 | 258 |         | Yes      |       0 | NO PAD        |
| utf8mb4_lv_0900_as_cs       | utf8mb4 | 281 |         | Yes      |       0 | NO PAD        |
| utf8mb4_mn_cyrl_0900_ai_ci  | utf8mb4 | 322 |         | Yes      |       0 | NO PAD        |
| utf8mb4_mn_cyrl_0900_as_cs  | utf8mb4 | 323 |         | Yes      |       0 | NO PAD        |
| utf8mb4_nb_0900_ai_ci       | utf8mb4 | 310 |         | Yes      |       0 | NO PAD        |
| utf8mb4_nb_0900_as_cs       | utf8mb4 | 311 |         | Yes      |       0 | NO PAD        |
| utf8mb4_nn_0900_ai_ci       | utf8mb4 | 312 |         | Yes      |       0 | NO PAD        |
| utf8mb4_nn_0900_as_cs       | utf8mb4 | 313 |         | Yes      |       0 | NO PAD        |
| utf8mb4_persian_ci          | utf8mb4 | 240 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_pl_0900_ai_ci       | utf8mb4 | 261 |         | Yes      |       0 | NO PAD        |
| utf8mb4_pl_0900_as_cs       | utf8mb4 | 284 |         | Yes      |       0 | NO PAD        |
| utf8mb4_polish_ci           | utf8mb4 | 229 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_romanian_ci         | utf8mb4 | 227 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_roman_ci            | utf8mb4 | 239 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_ro_0900_ai_ci       | utf8mb4 | 259 |         | Yes      |       0 | NO PAD        |
| utf8mb4_ro_0900_as_cs       | utf8mb4 | 282 |         | Yes      |       0 | NO PAD        |
| utf8mb4_ru_0900_ai_ci       | utf8mb4 | 306 |         | Yes      |       0 | NO PAD        |
| utf8mb4_ru_0900_as_cs       | utf8mb4 | 307 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sinhala_ci          | utf8mb4 | 243 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_sk_0900_ai_ci       | utf8mb4 | 269 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sk_0900_as_cs       | utf8mb4 | 292 |         | Yes      |       0 | NO PAD        |
| utf8mb4_slovak_ci           | utf8mb4 | 237 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_slovenian_ci        | utf8mb4 | 228 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_sl_0900_ai_ci       | utf8mb4 | 260 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sl_0900_as_cs       | utf8mb4 | 283 |         | Yes      |       0 | NO PAD        |
| utf8mb4_spanish2_ci         | utf8mb4 | 238 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_spanish_ci          | utf8mb4 | 231 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_sr_latn_0900_ai_ci  | utf8mb4 | 314 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sr_latn_0900_as_cs  | utf8mb4 | 315 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sv_0900_ai_ci       | utf8mb4 | 264 |         | Yes      |       0 | NO PAD        |
| utf8mb4_sv_0900_as_cs       | utf8mb4 | 287 |         | Yes      |       0 | NO PAD        |
| utf8mb4_swedish_ci          | utf8mb4 | 232 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_tr_0900_ai_ci       | utf8mb4 | 265 |         | Yes      |       0 | NO PAD        |
| utf8mb4_tr_0900_as_cs       | utf8mb4 | 288 |         | Yes      |       0 | NO PAD        |
| utf8mb4_turkish_ci          | utf8mb4 | 233 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_unicode_520_ci      | utf8mb4 | 246 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_unicode_ci          | utf8mb4 | 224 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_vietnamese_ci       | utf8mb4 | 247 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_vi_0900_ai_ci       | utf8mb4 | 277 |         | Yes      |       0 | NO PAD        |
| utf8mb4_vi_0900_as_cs       | utf8mb4 | 300 |         | Yes      |       0 | NO PAD        |
| utf8mb4_zh_0900_as_cs       | utf8mb4 | 308 |         | Yes      |       0 | NO PAD        |
+-----------------------------+---------+-----+---------+----------+---------+---------------+
117 rows in set (0.01 sec)
```

常用操作2：

```sql
#查看服务器的字符集和比较规则
SHOW VARIABLES LIKE '%_server';

#查看数据库的字符集和比较规则
SHOW VARIABLES LIKE '%_database';

#查看具体数据库的字符集
SHOW CREATE DATABASE db1;

#修改具体数据库的字符集
ALTER DATABASE db1 DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
```

![image-20240522182855189](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908789.png)

 `utf8mb3` 指的是 MySQL 过去所称的 `utf8` 编码，其实它支持最多3个字节的UTF-8字符。

常用操作3：

```sql
#查看表的字符集
show create table t_emp;

#查看表的比较规则
show table status from db1 like 't_emp';

#修改表的字符集和比较规则
ALTER TABLE t_emp DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';
```

![image-20240522184003881](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908964.png)

## 请求到响应过程中字符集的变化

![image-20240522184742448](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908744.png)

这几个系统变量在我的计算机上的默认值如下（不同操作系统的默认值可能不同）：

```sql
mysql> show variables like 'character%';
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
+--------------------------+--------------------------------+
8 rows in set (0.00 sec)
```

为了体现出字符集在请求处理过程中的变化，我们这里特意修改一个系统变量的值：

```sql
mysql> set character_set_connection = gbk;
Query OK, 0 rows affected (0.00 sec)
```

现在假设我们客户端发送的请求是下边这个字符串：

```sql
SELECT * FROM t WHERE s = '我';
```

为了方便大家理解这个过程，我们只分析字符 '我' 在这个过程中字符集的转换。

 现在看一下在请求从发送到结果返回过程中字符集的变化：

1. 客户端发送请求所使用的字符集

一般情况下客户端所使用的字符集和当前操作系统一致，不同操作系统使用的字符集可能不一 样，如下：

​	类 Unix 系统使用的是 

​	utf8 Windows 使用的是 gbk

当客户端使用的是 utf8 字符集，字符 '我' 在发送给服务器的请求中的字节形式就是： ==0xE68891==

提示: 如果你使用的是可视化工具，比如navicat之类的，这些工具可能会使用自定义的字符集来编 码发送到服务器的字符串，而不采用操作系统默认的字符集（尽量用命令行窗口）。

2. 服务器接收到客户端发送来的请求其实是一串二进制的字节，它会认为这串字节采用的字符集是 character_set_client ，然后把这串字节转换为 character_set_connection 字符集编码的 字符。

由于我的计算机上 character_set_client 的值是 utf8 ，首先会按照 utf8 字符集对字节串 0xE68891 进行解码，得到的字符串就是 '我' ，然后按照 character_set_connection 代表的 字符集，也就是 gbk 进行编码，得到的结果就是字节串 ==0xCED2== 。

3. 因为表 t 的列 col 采用的是 gbk 字符集，与 character_set_connection 一致，所以直接到列 中找字节值为 0xCED2 的记录，最后找到了一条记录。

提示: 如果某个列使用的字符集和character_set_connection代表的字符集不一致的话，还需要进行 一次字符集转换。

4. 上一步骤找到的记录中的 col 列其实是一个字节串 0xCED2 ， col 列是采用 gbk 进行编码的，所 以首先会将这个字节串使用 gbk 进行解码，得到字符串 '我' ，然后再把这个字符串使用 character_set_results 代表的字符集，也就是 utf8 进行编码，得到了新的字节串： 0xE68891 ，然后发送给客户端。

5. 由于客户端是用的字符集是 utf8 ，所以可以顺利的将 0xE68891 解释成字符 我 ，从而显示到我 们的显示器上，所以我们人类也读懂了返回的结果。

总结图示如下：

![image-20240522200716940](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908115.png)



开发中通常把character_set_client、character_set_connection、character_set_results这三个系统变量设置成和客户端使用的字符集一致的情况，这样减少了很多无谓的字符集转换。为了方便我们设置，MySQL提供了一条非常简便的语句：

```sql
SET NAMES 字符集名;
```

比方说我的客户端使用的是utf8字符集，所以需要把这几个系统变量的值都设置为utf8：

```sql
SET NAMES utf8;
```

另外,如果你想在启动客户端的时候就把character_set_client、character_set_connection、character_set_results 这三个系统变量的值设置成一样的，那我们可以在启动客户端的时候指定一个叫default-character-set的启动选项，比如在配置文件里可以这么写：

```shell
[client]
default-character-set=utf8
```

它起到的效果和执行一遍SET NAMES utf8是一样的.

# SQL大小写规范

## Windows和Linux平台区别

在 SQL 中，关键字和函数名是不用区分字母大小写的，比如 SELECT、WHERE、ORDER、GROUP BY 等关 键字，以及 ABS、MOD、ROUND、MAX 等函数名。

不过在 SQL 中，你还是要确定大小写的规范，因为在 Linux 和 Windows 环境下，你可能会遇到不同的大 小写问题。 windows系统默认大小写不敏感 ，但是 linux系统是大小写敏感的 。

通过如下命令查看：

Windows :

![image-20240522202312523](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908180.png)

Linux :

```sql
mysql> SHOW VARIABLES LIKE '%lower_case_table_names%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 0     |
+------------------------+-------+
1 row in set (0.00 sec)
```

lower_case_table_names参数值的设置：

​	默认为0，大小写敏感 。

​	设置1，大小写不敏感。创建的表，数据库都是以小写形式存放在磁盘上，对于sql语句都是转 换为小写对表和数据库进行查找。

​	设置2，创建的表和数据库依据语句上格式存放，凡是查找都是转换为小写进行。



MySQL在Windows的环境下全部不区分大小写

MySQL在Linux下数据库名、表名、列名、别名大小写规则是这样的：

​	数据库名、表名、表的别名、变量名是严格区分大小写的；

​	关键字、函数名称在 SQL 中不区分大小写；

​	列名（或字段名）与列的别名（或字段别名）在所有的情况下均是忽略大小写的；



## Linux下大小写规则设置

当想设置为大小写不敏感时，要在 my.cnf 这个配置文件 [mysqld] 中加入

```shell
lower_case_table_names=1
```

然后重启服务器。

​	但是要在重启数据库实例之前就需要将原来的数据库和表转换为小写，否则将找不到数据库名。

此参数适用于MySQL5.7。





在MySQL 8下禁止在重新启动 MySQL 服务时将 lower_case_table_names 设置成不同于初始化 MySQL 服务时设置的 lower_case_table_names 值。如果非要将MySQL8设置为大小写不敏感，具体步骤为：

1、停止MySQL服务

2、删除数据目录，即删除 /var/lib/mysql 目录 (数据库 软链)

3、在MySQL配置文件（ /etc/my.cnf ）中添加 lower_case_table_names=1 

4、启动MySQL服务

注意：在进行数据库参数设置之前，需要掌握这个参数带来的影响，切不可盲目设置。

## SQL编写建议

如果你的变量名命名规范没有统一，就可能产生错误。这里有一个有关命名规范的建议：

1. 关键字和函数名称全部大写； 
2.  数据库名、表名、表别名、字段名、字段别名等全部小写； 
3.  SQL 语句必须以分号结尾。

数据库名、表名和字段名在 Linux MySQL 环境下是区分大小写的，因此建议你统一这些字段的命名规 则，比如全部采用小写的方式。

虽然关键字和函数名称在 SQL 中不区分大小写，也就是如果小写的话同样可以执行。但是同时将关键词 和函数名称全部大写，以便于区分数据库名、表名、字段名。

# sql_mode的合理设置

sql_mode 会影响MySQL支持的sQL语法以及它执行的数据验证检查。通过设置sql_mode,可以完成不同严格程度的数据校验，有效地保障数据准确性。

MySQL5.6和MySQL5.7默认的sql_mode模式参数是不一样的：

5.6的mode默认值为空（即：NO_ENGINE_SUBSTITUTION），其实表示的是一个空值，相当于没有什么模式设置，可以理解为宽松模式。在这种设置下是可以允许一些非法操作的，比如允许一些非法数据的插入。

5.7的mode是STRICT_TRANS_TABLES,也就是严格模式。用于进行数据的严格校验,错误数据不能插入,报error（错误），并且事务回滚。

## 宽松模式 vs 严格模式

#### 宽松模式：

如果设置的是宽松模式，那么我们在插入数据的时候，即便是给了一个错误的数据，也可能会被接受， 并且不报错。

举例 ：我在创建一个表时，该表中有一个字段为name，给name设置的字段类型时 char(10) ，如果我 在插入数据的时候，其中name这个字段对应的有一条数据的 长度超过了10 ，例如'1234567890abc'，超 过了设定的字段长度10，那么不会报错，并且取前10个字符存上，也就是说你这个数据被存为 了'1234567890'，而'abc'就没有了。但是，我们给的这条数据是错误的，因为超过了字段长度，但是并没 有报错，并且mysql自行处理并接受了，这就是宽松模式的效果。

应用场景 ：通过设置sql mode为宽松模式，来保证大多数sql符合标准的sql语法，这样应用在不同数据 库之间进行 迁移 时，则不需要对业务sql 进行较大的修改。

#### 严格模式：

出现上面宽松模式的错误，应该报错才对，所以MySQL5.7版本就将sql_mode默认值改为了严格模式。所 以在 生产等环境 中，我们必须采用的是严格模式，进而 开发、测试环境 的数据库也必须要设置，这样在 开发测试阶段就可以发现问题。并且我们即便是用的MySQL5.6，也应该自行将其改为严格模式。

开发经验 ：MySQL等数据库总想把关于数据的所有操作都自己包揽下来，包括数据的校验，其实开发 中，我们应该在自己 开发的项目程序级别将这些校验给做了 ，虽然写项目的时候麻烦了一些步骤，但是这 样做之后，我们在进行数据库迁移或者在项目的迁移时，就会方便很多。

改为严格模式后可能会存在的问题：

若设置模式中包含了 NO_ZERO_DATE ，那么MySQL数据库不允许插入零日期，插入零日期会抛出错误而 不是警告。例如，表中含字段TIMESTAMP列（如果未声明为NULL或显示DEFAULT子句）将自动分配 DEFAULT '0000-00-00 00:00:00'（零时间戳），这显然是不满足sql_mode中的NO_ZERO_DATE而报错。

## 模式查看和设置

### 查看当前的sql_mode

```sql
select @@session.sql_mode;
select @@global.sql_mode;
#或者
show variables like 'sql_mode';
```

![image-20240522211938148](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908304.png)



![image-20240522211955201](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110908552.png)

临时设置方式：设置当前窗口中设置sql_mode

```sql
SET GLOBAL sql_mode = 'modes...'; #全局
SET SESSION sql_mode = 'modes...'; #当前会话
```

举例：

```sql
#改为严格模式。此方法只在当前会话中生效，关闭当前会话就不生效了。
set SESSION sql_mode='STRICT_TRANS_TABLES';
```



```sql
#改为严格模式。此方法在当前服务中生效，重启MySQL服务后失效。
set GLOBAL sql_mode='STRICT_TRANS_TABLES';
```

永久设置方式：在/etc/my.cnf中配置sql_mode

在my.cnf文件(windows系统是my.ini文件)，新增：

```shell
[mysqld]
sql_mode=ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR
_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

然后 重启MySQL 。

当然生产环境上是禁止重启MySQL服务的，所以采用 临时设置方式 + 永久设置方式 来解决线上的问题， 那么即便是有一天真的重启了MySQL服务，也会永久生效了。

![image-20240523091608975](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110909368.png)

## 宽松模式举例

```sql
CREATE TABLE mytbl (id INT, NAME VARCHAR(16) ,age INT, dept INT);

INSERT INTO mytbl VALUES(1,'zhang3',33,101);
INSERT INTO mytbl VALUES(2,'li4',34,101);
INSERT INTO mytbl VALUES(3,'wang5',34,102);
INSERT INTO mytbl VALUES(4,'zhao6',34,102);
INSERT INTO mytbl VALUES(5,'tian7',36,102);
```

![image-20240523090615006](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110909888.png)

```sql
#查询每个部门年龄最大的人
SELECT NAME,dept,MAX(age) FROM mytbl GROUP BY dept;
set SESSION sql_mode = '';
SELECT NAME,dept,MAX(age) FROM mytbl GROUP BY dept;
```

![image-20240523090843252](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406110909307.png)