事务有4种特性：原子性、一致性、隔离性和持久性。那么事务的四种特性到底是基于什么机制实现呢？

事务的隔离性由 锁机制 实现。

而事务的原子性、一致性和持久性由事务的 redo 日志和undo 日志来保证。

REDO LOG 称为 重做日志 ，提供再写入操作，恢复提交事务修改的页操作，用来保证事务的持久性。

UNDO LOG 称为 回滚日志 ，回滚行记录到某个特定版本，用来保证事务的原子性、一致性。



有的DBA或许会认为 UNDO 是 REDO 的逆过程，其实不然。REDO 和 UNDO 都可以视为是一种 恢复操作，但是：

redo log:是存储引擎层(innodb)生成的日志,记录的是"物理级别"上的页修改操作,比如页号xxx、偏移量yyy写入了'zzz'数据。主要为了保证数据的可靠性；

undo log:是存储引擎层(innodb)生成的日志,记录的是逻辑操作日志,比如对某一行数据进行了INSERT语句操作,那么undo log就记录一条与之相反的DELETE操作。主要用于事务的回滚(undo log记录的是每个修改操作的逆操作）和 一致性非锁定读(undo log 回滚行记录到某种特定的版本---MVCC，即多版本并发控制)。

# redo日志

InnoDB存储引擎是以页为单位来管理存储空间的。在真正访问页面之前,需要把在磁盘上的页缓存到内存中的Buffer Pool之后才可以访问。所有的变更都必须先更新缓冲池中的数据,然后缓冲池中的脏页会以一定的频率被刷入磁盘（checkPoint机制) ,通过缓冲池来优化CPU和磁盘之间的鸿沟,这样就可以保证整体的性能不会下降太快。

## 为什么需要REDO日志

一方面，缓冲池可以帮助我们消除CPU和磁盘之间的鸿沟，checkpoint机制可以保证数据的最终落盘，然而由于checkpoint 并不是每次变更的时候就触发 的，而是master线程隔一段时间去处理的。所以最坏的情况就是事务提交后，刚写完缓冲池，数据库宕机了，那么这段数据就是丢失的，无法恢复。

另一方面，事务包含 持久性 的特性，就是说对于一个已经提交的事务，在事务提交后即使系统发生了崩溃，这个事务对数据库中所做的更改也不能丢失。

那么如何保证这个持久性呢？ 一个简单的做法 ：在事务提交完成之前把该事务所修改的所有页面都刷新到磁盘，但是这个简单粗暴的做法有些问题

修改量与刷新磁盘工作量严重不成比例有时候我们仅仅修改了某个页面中的一个字节，但是我们知道在InnoDB中是以页为单位来进行磁盘10的，也就是说我们在该事务提交时不得不将一个完整的页面从内存中刷新到磁盘，我们又知道一个页面默认是16KB大小，只修改一个字节就要刷新16KB的数据到磁盘上显然是太小题大做了。

随机10刷新较慢一个事务可能包含很多语句，即使是一条语句也可能修改许多页面，假如该事务修改的这些页面可能并不相邻,这就意味着在将某个事务修改的Buffer Pool中的页面 刷新到磁盘时,需要进行很多的 随机10,随机10比顺序10要慢,尤其对于传统的机械硬盘来说。

另一个解决的思路 ：我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效，即使后来系统崩溃，在重启后也能把这种修改恢复出来。所以我们其实没有必要在每次事务提交时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要把 修改 了哪些东西 记录一下 就好。比如，某个事务将系统表空间中 第10号 页面中偏移量为 100 处的那个字节的值 1 改成 2 。我们只需要记录一下：将第0号表空间的10号页面的偏移量为100处的值更新为 2 。

InnoDB引擎的事务采用了WAL技术(Write-Ahead Logging) ,这种技术的思想就是先写日志,再写磁盘,只有日志写入成功,才算事务提交成功,这里的日志就是redo log。当发生宕机且数据未刷到磁盘的时候,可以通过redo log来恢复，保证ACID中的D，这就是redo log的作用。

![image-20240618134539659](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355575.png)

## REDO日志的好处、特点

好处
redo日志降低了刷盘频率
redo日志占用的空间非常小

存储表空间ID、页号、偏移量以及需要更新的值，所需的存储空间是很小的，刷盘快。

特点
redo日志是顺序写入磁盘的      在执行事务的过程中,每执行一条语句，就可能产生若干条repo日志，这些日志是按照产生的顺序写入磁盘的,也就是使用顺序10，效率比随机10快。
事务执行过程中，redo log不断记录   redo log 跟bin log的区别, redo log是存储引擎层产生的,而bin log是数据库层产生的。假设一个事务,对表做10万行的记录插入,在这个过程中,一直不断的往redo log顺序记录,而bin log不会记录,直到这个事务提交,才会一次写入到bin log文件中。

## redo的组成

Redo log可以简单分为以下两个部分：

重做日志的缓冲 (redo log buffer) ，保存在内存中，是易失的。

在服务器启动时就向操作系统申请了一大片称之为redo log buffer的连续内存空间,翻译成中文就是redo日志缓冲区。这片内存空间被划分成若干个连续的 redo log block。一个redo log block占用512字节大小。

![image-20240618135252305](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355769.png)

​	**参数设置：****innodb_log_buffer_size****：**

redo log buffer 大小，默认 16M ，最大值是4096M，最小值为1M。

```sql
mysql> show variables like '%innodb_log_buffer_size%';
+------------------------+----------+
| Variable_name | Value |
+------------------------+----------+
| innodb_log_buffer_size | 16777216 |
+------------------------+----------+
```

重做日志文件 (redo log file) ，保存在硬盘中，是持久的

## redo的整体流程

以一个更新事务为例，redo log 流转过程，如下图所示：

![image-20240618134933698](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355773.png)

第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝

第2步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值

第3步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式

第4步：定期将内存中修改的数据刷新到磁盘中

## redo log的刷盘策略

redo log的写入并不是直接写入磁盘的，InnoDB引擎会在写redo log的时候先写redo log buffer，之后以 一定的频率 刷入到真正的redo log file 中。这里的一定频率怎么看待呢？这就是我们要说的刷盘策略。

![image-20240618135501166](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356760.png)

注意，redo log buffer刷盘到redo log file的过程并不是真正的刷到磁盘中去，只是刷入到 文件系统缓存

（page cache）中去（这是现代操作系统为了提高文件写入效率做的一个优化），真正的写入会交给系

统自己来决定（比如page cache足够大了）。那么对于InnoDB来说就存在一个问题，如果交给系统来同

步，同样如果系统宕机，那么数据也丢失了（虽然整个系统宕机的概率还是比较小的）。

针对这种情况，InnoDB给出 innodb_flush_log_at_trx_commit 参数，该参数控制 commit提交事务

时，如何将 redo log buffer 中的日志刷新到 redo log file 中。它支持三种策略：

设置为0 ：表示每次事务提交时不进行刷盘操作。（系统默认master thread每隔1s进行一次重做日志的同步）

设置为1 ：表示每次事务提交时都将进行同步，刷盘操作（ 默认值 ）

设置为2 ：表示每次事务提交时都只把 redo log buffer 内容写入 page cache，不进行同步。由os自己决定什么时候同步到磁盘文件。

```sql
mysql> show variables like 'innodb_flush_log_at_trx_commit';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
1 row in set (0.01 sec)
```

![image-20240618135710955](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356425.png)

也就是说，一个没有提交事务的 redo log 记录，也可能会刷盘。因为在事务执行过程 redo log 记录是会写入redo log buffer 中，这些 redo log 记录会被 后台线程 刷盘。

![image-20240618135746349](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356120.png)

除了后台线程每秒1次的轮询操作,还有一种情况,当redo log buffer占用的空间即将达到innodb_log_buffer_size (这个参数默认是16M)的一半的时候,后台线程会主动刷盘。

# **不同刷盘策略演示**

**流程图**

![image-20240618135826711](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356868.png)



为1时，只要事务提交成功，redo log 记录就一定在硬盘里，不会有任何数据丢失。如果事务执行期间 MySQL 挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。可以保证ACID的D,数据绝对不会丢失,但是效率最差的。建议使用默认值，虽然操作系统宕机的概率理论小于数据库宕机的概率，但是一般既然使用了事务，那么数据的安全相对来说更重要些。

![image-20240618135839403](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356654.png)



为2时,只要事务提交成功, redo log buffer中的内容只写入文件系统缓存(page cache)。如果仅仅只是 MySQL 挂了不会有任何数据丢失，但是操作系统宕机可能会有1秒数据的丢失，这种情况下无法满足ACID中的D。但是数值2 肯定是效率最高的。

![image-20240618135849422](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191356376.png)

为0时，master thread中每1秒进行一次重做日志的fsync操作，因此实例 crash 最多丢失1秒钟内的事务。(master thread是负责将缓冲池中的数据异步刷新到磁盘,保证数据的一致性)数值0的话,是一种折中的做法,它的10效率理论是高于1的,低于2的,这种策略也有丢失数据的风险,也无法保证D。

## 写入redo log buffer 过程

### 补充概念：Mini-Transaction

MySQL把对底层页面中的一次原子访问的过程称之为一个Mini-Transaction，简称 mtr，比如，向某个索引对应的B+树中插入一条记录的过程就是一个Mini-Transaction。一个所谓的mtr 可以包含一组redo日志，在进行崩溃恢复时这一组 redo 日志作为一个不可分割的整体。

一个事务可以包含若干条语句，每一条语句其实是由若干个 mtr 组成，每一个 mtr 又可以包含若干条redo日志，画个图表示它们的关系就是这样：

![image-20240618140407129](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357453.png)

## redo 日志写入log buffer

向log buffer中写入redo日志的过程是顺序的,也就是先往前边的block中写,当该block的空闲空间用完之后再往下一个block中写。当我们想往log buffer中写入redo日志时,第一个遇到的问题就是应该写在哪个block的哪个偏移量处,所以InnoDB的设计者特意提供了一个称之为buf_free的全局变量,该变量指明后续写入的redo日志应该写入到log buffer中的哪个位置,如图所示:

![image-20240618140458333](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357074.png)

一个mtr执行过程中可能产生若干条redo日志，这些redo日志是一个不可分割的组，所以其实并不是每生成一条redo日志,就将其插入到log buffer中,而是每个mtr运行过程中产生的日志先暂时存到一个地方,当该mtr结束的时候,将过程中产生的一组redo日志再全部复制到log buffer中。我们现在假设有两个名为T1、T2的事务,每个事务都包含2个mtr，我们给这几个mtr命名一下：

事务 T1 的两个mtr 分别称为mtrT1_1 和mtr_T1_2。事务 T2 的两个mtr 分别称为mtr_T2_1 和mtr_T2_2。

每个mtr都会产生一组redo日志，用示意图来描述一下这些mtr产生的日志情况：

![image-20240618140512452](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357962.png)

不同的事务可能是并发执行的,所以T1、T2之间的mtr可能是交替执行的。每当一个mtr执行完成时,伴随该mtr生成的一组redo日志就需要被复制到log buffer中，也就是说不同事务的mtr可能是交替写入log buffer的，我们画个示意图（为了美观，我们把一个mtr中产生的所有的redo日志当作一个整体来画）：

![image-20240618140525203](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357372.png)

### redo log block的结构图

一个redo log block是由日志头、日志体、日志尾组成。日志头占用12字节,日志尾占用8字节,所以一个block真正能存储的数据就是512-12-8=492字节。

![image-20240618140804203](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357220.png)

真正的redo日志都是存储到占用 496 字节大小的 log block body 中，图中的log block header 和logblock trailer存储的是一些管理信息。我们来看看这些所谓的管理信息都有什么。

![image-20240618140905139](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357481.png)

![image-20240618140922027](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191357871.png)

## **redo log file**

innodb_log_group_home_dir ：指定 redo log 文件组所在的路径，默认值为 ./ ，表示在数据库的数据目录下。MySQL的默认数据目录（ var/lib/mysql ）下默认有两个名为 ib_logfile0 和ib_logfile1 的文件，log buffer中的日志默认情况下就是刷新到这两个磁盘文件中。此redo日志文件位置还可以修改。

**innodb_log_files_in_group**：指明redo log file的个数，命名方式如：ib_logfile0，iblogfile1...iblogfilen。默认2个，最大100个

```sql
mysql> show variables like 'innodb_log_files_in_group';
+---------------------------+-------+
| Variable_name | Value |
+---------------------------+-------+
| innodb_log_files_in_group | 2 |
+---------------------------+-------+
#ib_logfile0
#ib_logfile1
```

**innodb_flush_log_at_trx_commit**：控制 redo log 刷新到磁盘的策略，默认为1。

**innodb_log_file_size**：单个 redo log 文件设置大小，默认值为 48M 。最大值为512G，注意最大值指的是整个 redo log 系列文件之和，即（innodb_log_files_in_group * innodb_log_file_size ）不能大于最大值512G。

```sql
mysql> show variables like 'innodb_log_file_size';
+----------------------+----------+
| Variable_name | Value |
+----------------------+----------+
| innodb_log_file_size | 50331648 |
+----------------------+----------+
```

根据业务修改其大小，以便容纳较大的事务。编辑my.cnf文件并重启数据库生效，如下所示

```shell
[root@localhost ~]# vim /etc/my.cnf
innodb_log_file_size=200M
```

 

### **日志文件组**

![image-20240618141143273](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191358179.png)

总共的redo日志文件大小其实就是： innodb_log_file_size × innodb_log_files_in_group 。

采用循环使用的方式向redo日志文件组里写数据的话，会导致后写入的redo日志覆盖掉前边写的redo日志？当然！所以InnoDB的设计者提出了checkpoint的概念。

### **checkpoint**

![image-20240618141215322](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191358393.png)

如果 write pos 追上 checkpoint ，表示**日志文件组**满了，这时候不能再写入新的 redo log记录，MySQL 得停下来，清空一些记录，把 checkpoint 推进一下。

![image-20240618141233023](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191358066.png)

# **Undo****日志**

redo log是事务持久性的保证，undo log是事务原子性的保证。在事务中 更新数据 的 前置操作 其实是要先写入一个 undo log 。

## 如何理解Undo日志

事务需要保证 原子性 ，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半

会出现一些情况，比如：

情况一：事务执行过程中可能遇到各种错误，比如 服务器本身的错误 ， 操作系统错误 ，甚至是突然 断电 导致的错误。

情况二：程序员可以在事务执行过程中手动输入 ROLLBACK 语句结束当前事务的执行。

以上情况出现，我们需要把数据改回原先的样子，这个过程称之为 回滚 ，这样就可以造成一个假象：这个事务看起来什么都没做，所以符合 原子性 要求。

MySQL把这些为了回滚而记录的这些内容称之为撤销日志或者回滚日志(即undo log) 。注意,由于查询操作(SELECT)并不会修改任何用户记录,所以在查询操作执行时,并不需要记录相应的undo日志。

##  Undo日志的作用

作用1：回滚数据

用户对undo日志可能有误解: undo用于将数据库物理地恢复到执行语句或事务之前的样子。但事实并非如此。undo是逻辑日志,因此只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了,但是数据结构和页本身在回滚之后可能大不相同。这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如,一个事务在修改当前一个页中某几条记录,同时还有别的事务在对同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

作用2：MVCC

undo的另一个作用是MVCC,即在InnoDB存储引擎中MVCc的实现是通过undo来完成。当用户读取一行记录时,若该记录已经被其他事务占用，当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取。

## undo的存储结构

### 回滚段与undo页

InnoDB对undo log的管理采用段的方式，也就是 回滚段（rollback segment） 。每个回滚段记录了1024 个 undo log segment ，而在每个undo log segment段中进行 undo页 的申请

在 InnoDB1.1版本之前 （不包括1.1版本），只有一个rollback segment，因此支持同时在线的事务限制为 1024 。虽然对绝大多数的应用来说都已经够用。

从1.1版本开始InnoDB支持最大 128个rollback segment ，故其支持同时在线的事务限制提高到了 128*1024 。

```sql
mysql> show variables like 'innodb_undo_logs';
+------------------+-------+
| Variable_name | Value |
+------------------+-------+
| innodb_undo_logs | 128 |
+------------------+-------+
```

 

### **回滚段与事务**

\1. 每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务。

\2. 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段。

\3. 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘区或者在回滚段允许的情况下扩展新的盘区来使用。

\4. 回滚段存在于undo表空间中，在数据库中可以存在多个undo表空间，但同一时刻只能使用一个undo表空间。

\5. 当事务提交时，InnoDB存储引擎会做以下两件事情：

将undo log放入列表中，以供之后的purge操作

判断undo log所在的页是否可以重用，若可以分配给下个事务使用

### **回滚段中的数据分类**

\1. 未提交的回滚数据(uncommitted undo information)

\2. 已经提交但未过期的回滚数据(committed undo information)

\3. 事务已经提交并过期的数据(expired undo information)

###  undo的类型

在InnoDB存储引擎中，undo log分为：

insert undo log

update undo log

##  undo log的生命周期

### **简要生成过程**

只有Buffer Pool的流程：

![image-20240618142103453](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191358882.png)

有了Redo Log和Undo Log之后：

![image-20240618142118373](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191358635.png)

### **详细生成过程**

![image-20240618142152856](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355670.png)

当我们执行INSERT时：

```
begin;
INSERT INTO user (name) VALUES ("tom");
```

![image-20240618142212526](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355620.png)

当我们执行UPDATE时：

![image-20240618142225961](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355216.png)

```
UPDATE user SET id=2 WHERE id=1;
```

![image-20240618142241639](https://gitee.com/dongguo4812_admin/image/raw/master/image/202406191355619.png)

### undo log是如何回滚的

​	

以上面的例子来说，假设执行rollback，那么对应的流程应该是这样：

\1. 通过undo no=3的日志把id=2的数据删除

\2. 通过undo no=2的日志把id=1的数据的deletemark还原成0

\3. 通过undo no=1的日志把id=1的数据的name还原成Tom

\4. 通过undo no=0的日志把id=1的数据删除

### **undo log**的删除

针对于insert undo log

因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

针对于update undo log

该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。