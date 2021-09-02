##最近在学习极客时间的MySQL实战45讲，记录一下学习内容：

### mysql的组件主要分为：连接器——查询缓存——分析器——优化器——执行器——存储引擎

连接器、分析器、优化器、执行器的功能跨多个存储引擎。


### 连接器：管理连接、权限认证
  1、通过show processlist命令可以看到已经建立的连接。
  2、wait_timeout参数控制连接空闲断开时间，默认为8小时。建议使用长连接，但执行过程中的临时内存是管理在连接对象里的，因此可能造成内存上涨。解决办法：
    a)定期断开长连接。通过程序判断执行过一个占用大内存的查询后，断开连接，后续查询时再重连。
    b)Mysql5.7+版本支持通过mysql_rest_connection参数来重新初始化连接资源。
### 查询缓存：不建议使用，因为更新操作会清空缓存，Mysql8.0已经将查询缓存功能删除。
### 分析器：词法分析、语法分析
  判断语法、表、列是否存在等。
### 优化器：执行计划生成，索引选择
### 执行器：操作引擎，返回结果
  慢查询日志中的rows_examined表示语句执行过程中**执行器**向**存储引擎**请求了多少次，而不是存储引擎扫描了多少行。



##两个日志对比：

|          | redo log                                                 | bin log                                                  |
| -------- | -------------------------------------------------------- | -------------------------------------------------------- |
| 位置   | InnoDB引擎特有，位于mysql的存储引擎层       | Server层实现，所有存储引擎都可以使用      |
| 记录内容 | 记录物理日志，即在某个数据页做了什么修改，真实的物理操作 | 记录逻辑日志，语句的原始逻辑，如给ID=2的这行数据的C列加1 |
| 写形式 | 固定的空间大小，循环写，满了就写入到磁盘中 | 追加写，不会覆盖，记录全量的备份数据   |


![image](https://user-images.githubusercontent.com/32328586/128743051-ed96ac23-02eb-4f4d-94d4-2bcff59e45ab.png)

- MySql 在5.6.2版本后，引入了binlog-checksum参数，用于验证bin log的完整性。
- redo log和bin log有一个共同的id，叫XID。
  - 如果碰到既有prepare、又有commit的redo log，就直接提交。
  - 如果碰到只有prepare，没有commit的redo log，则拿着XID去bin log找对应的事务。假如bin log对应的事务完整，则提交，否则回滚事务。


redo log 常见设置为 4 个文件、每个文件 1GB 。


#### binlog的写入机制
事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。**一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 binlog cache 的保存问题。**

系统给 binlog cache 分配了一片内存，每个线程一个，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。


双“1”配置：
- 对于binlog，sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
- 对于redo log，innodb_flush_log_at_trx_commit设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；

也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。

> 两阶段提交:时序上 redo log 先 prepare， 再写 binlog，最后再把 redo log commit。 前两个需要刷入到磁盘，最后的redo log commit 不需要，因为崩溃恢复只依赖于前两个。

组提交（group commit）机制：

![image](https://user-images.githubusercontent.com/32328586/130348799-95808409-4b8f-4bde-a61c-83ae9ce0ab83.png)

![image](https://user-images.githubusercontent.com/32328586/130348806-d238b6d2-f4a2-42e6-aa6f-36caff85779e.png)

![image](https://user-images.githubusercontent.com/32328586/130348817-9f27a405-012c-4aea-8563-06bfeae3493e.png)


##### 如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？

WAL机制主要的好处原因：
- 1、redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
- 2、组提交机制，可以大幅度降低磁盘的 IOPS 消耗。


1、设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2、将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。
3、将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。


不建议你把 innodb_flush_log_at_trx_commit 设置成 0。因为把这个参数设置成 0，表示 redo log 只保存在内存中，这样的话 MySQL 本身异常重启也会丢数据，风险太大。而 redo log 写到文件系统的 page cache 的速度也是很快的，所以将这个参数设置成 2 跟设置成 0 其实性能差不多，但这样做 MySQL 异常重启时就不会丢数据了，相比之下风险会更小。



#### binlog 主备同步过程：
主库内部有一个线程，专门用于服务备库B的长连接，日志完整同步过程如下：
1、在备库B上通过change master命令，设置主库A的IP、端口、用户名、密码，以及开始请求binlog的位置（包含文件名和日志偏移量）
2、在备库B上执行start slave命令，这个时候备库会启动两个线程，io_thread和sql_thread，期中io_thread负责与主库建立连接
3、主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发送给备库B。
4、备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。
5、sql_thread读取中转日志，解析出日志来了吗的命令，并执行。sql_thread当前已经是多线程来读。


![image](https://user-images.githubusercontent.com/32328586/130349754-8d04fcda-22bc-4600-8b95-a0903d545ff2.png)

- 当binlog_format=statement时，binlog记录的就是SQL的原文。 当涉及limit操作时，可能会引发主备不一致，因为不保证主备使用的索引是同一个。
- 当 binlog_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题。
- row格式占用空间大，例如我用一个sql删除10万条数据，statement只记录一条sql，row格式会记录10万条记录id。
因此Mysql有折中的方案，例如mixed格式的binlog。MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。（mixed格式的bin log现在使用得不多）

现在越来越多场景会使用row格式，因为即使是Delete场景，也会把记录的整行信息保存起来，也可以用来**恢复数据**
MariaDB 的Flashback工具就是基于row的整行信息恢复数据的。



> 不能直接用statement语句来直接拷贝出来恢复数据。举例：insert into table values(1,1,now())。当恢复数据的时候，获取的是恢复数据的时候的时间。需要用到对应的工具，因为bin log在记录event的时候，会记录当时的时间戳。用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。

``` sql
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```




***

## 事务

### ACID：Atomicity原子性、Consistency一致性、Isolation隔离性、Durability持久性

### 脏读dirty read（读取了另一个未提交的事务中的数据）、不可重复读non-repeatable read（读取了前一事务提交的数据，多次读取结果不同）、幻读phantom read [举例](https://www.zhihu.com/question/47007926)

### 隔离级别：
#### read uncommitted读未提交：一个事务还没提交时，他做的变更就能被别的事务看到。
#### read committed读提交：一个事务提交后，它的变更才会被其他事务看到。
#### repeatable read可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的
#### serializable串行化：写会加锁，读会加锁。

例子：

![image](https://user-images.githubusercontent.com/32328586/117567002-7e242300-b0ec-11eb-9340-3b34995e633a.png)

- 若隔离级别是“读未提交”， 则 V1 的值就是 2。这时候事务 B 虽然还没有提交，但是结果已经被 A 看到了。因此，V2、V3 也都是 2。
- 若隔离级别是“读提交”，则 V1 是 1，V2 的值是 2。事务 B 的更新在提交后才能被 A 看到。所以， V3 的值也是 2。
- 若隔离级别是“可重复读”，则 V1、V2 是 1，V3 是 2。之所以 V2 还是 1，遵循的就是这个要求：事务在执行期间看到的数据前后必须是一致的。
- 若隔离级别是“串行化”，则在事务 B 执行“将 1 改成 2”的时候，会被锁住。直到事务 A 提交后，事务 B 才可以继续执行。所以从 A 的角度看， V1、V2 值是 1，V3 的值是 2。

可重复读：数据库会通过MVCC机制创建一个视图，访问的时候以视图的逻辑结果为准，该级别的视图是在事务开始时创建的，整个事务都会使用这个视图。

Mysql的默认级别是RR，Oracle的级别是rc。

可以使用show variables like 'transaction_isolation';来查看当前的值。


### MVCC的实现：
在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。

![image](https://user-images.githubusercontent.com/32328586/117567765-3a331d00-b0f0-11eb-97a9-55dd7bb79aa1.png)

前值是 4，但是在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view。如图中看到的，在视图 A、B、C 里面，这一个记录的值分别是 1、2、4，同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（MVCC）。对于 read-view A，要得到 1，就必须将当前值依次执行图中所有的回滚操作得到。

回滚日志在不需要的时候才删除。系统会判断，当没有事务再需要用到这些回滚日志时，回滚日志会被删除，即当系统里没有比这个回滚日志更早的 read-view 的时候。
因此不建议使用长事务，因为在事务结束前，会保留大量的回滚日志。

可以在information_schema库的innodb_trx表中查询长事务：
``` markdown
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```



## 索引

二叉树的搜索效率是最高的，但实际大多数据库存储实现却不使用二叉树，因为索引不仅存在内存中，还要写到磁盘上。
二叉树的高度较高，访问树的一个节点=访问一次磁盘，一张100万数据的表，从磁盘读取一个数据块的时间大概为10ms，树高20，即一次查询需要10*20=200ms，很慢。（InnoDB通常使用1200叉树作为索引的数据结构）

### 主键索引和非主键索引

#### 主键索引
又称聚簇索引，叶子节点存的是整行数据

#### 非主键索引
又称二级索引，叶子节点存的是主键的值。


例子：一个主键列为 ID 的表，表中有字段 k，并且在 k 上有索引
``` markdown
mysql> create table T(
id int primary key, 
k int not null, 
name varchar(16),
index (k))engine=InnoDB;
```

R1~R5 的 (ID,k) 值分别为 (100,1)、(200,2)、(300,3)、(500,5) 和 (600,6)，两棵树的示例示意图如下。

![image](https://user-images.githubusercontent.com/32328586/117578885-29e86580-b123-11eb-9dde-be266328170f.png)



### 索引维护
如果插入新的行 ID 值为 700，则只需要在 R5 的记录后面插入一个新记录。如果新插入的 ID 值为 400，就相对麻烦了，需要逻辑上挪动后面的数据，空出位置。而更糟的情况是，如果 R5 所在的数据页已经满了，根据 B+ 树的算法，这时候需要申请一个新的数据页，然后挪动部分数据过去。这个过程称为页分裂。在这种情况下，性能自然会受影响。
页分裂也影响数据页的利用率，一个页数据分成两个页，对于原本的一页下降50%。当删除数据时，相邻两页利用率变很低后，也会自动合并页。

**自增主键的优势：每次插入数据，都是追加操作，不涉及挪动其他记录，也不触发叶子节点的页分裂**，而当uuid作为主键，不保证有序追加插入，这样写数据成本相对较高，因为有可能触发页分裂和挪其他记录。

由于每个**非主键索引**的叶子节点上都是主键的值，假如用uuid作为主键，每个二级索引的叶子节点占用约32个字节。而用整形做主键，只要4个字节，长整型是8个字节。

**显然，主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小**

- ASCII 码中，一个英文字母（不分大小写）为一个字节，一个中文汉字为两个字节。
- UTF-8 编码中，一个英文字为一个字节，一个中文为三个字节。

适合用业务字段做主键的场景：
1、只有一个索引，即二级索引很少的时候或没有，这时就可以直接使用主键索引查询，不需要二级索引导致查询2次。
2、该索引是唯一索引


### 回表
回到主键索引树搜索的过程，称为回表。

避免回表的场景：**覆盖索引**
假如只需要查询主键，例如只查询id：select **ID** from T where k between 3 and 5;要查询的值已经在k索引树上了，不需要回表。

**由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。**

注意点：在引擎内部使用覆盖索引在索引 k 上其实读了三个记录，R3~R5（对应的索引 k 上的记录项），但是对于 MySQL 的 Server 层来说，它就是找引擎拿到了两条记录，因此 MySQL 认为扫描行数是 2，即rows_examined为2。


### 多列索引
需要考虑最左前缀匹配原则，索引的顺序取决于：
- 索引的使用频率，高频的放前面
- 索引占用的空间，空间大的索引尽量复用。
- 覆盖索引场景（当多列索引的列覆盖需要查询的列时，会触发覆盖索引，减少回表查询）


![image](https://user-images.githubusercontent.com/32328586/117581140-e5ae9280-b12d-11eb-8808-46edadcdd296.png)

（a,b）的联合索引，可以看到a的值是有顺序的，1，1，2，2，3，3，而b的值是没有顺序的1，2，1，4，1，2。但是我们又可发现a在等值的情况下，b值又是按顺序排列的，但是这种顺序是相对的。这是因为MySQL创建联合索引的规则是首先会对联合索引的最左边第一个字段排序，在第一个字段的排序基础上，然后在对第二个字段进行排序。所以b=2这种查询条件没有办法利用索引。


经典例子：
表结构如下：
``` markdown

CREATE TABLE `geek` (
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  PRIMARY KEY (`a`,`b`),
  KEY `c` (`c`),
  KEY `ca` (`c`,`a`),
  KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;
```

由于历史原因，这个表需要 a、b 做联合主键。既然主键包含了 a、b 这两个字段，那意味着单独在字段 c 上创建一个索引，就已经包含了三个字段了呀，为什么要创建“ca”“cb”这两个索引？
因为存在如下业务场景：
``` markdown
select * from geek where c=N order by a limit 1;
select * from geek where c=N order by b limit 1;
```

索引不合理，索引ca可以去掉，因为：

**InnoDB会把主键字段放到索引定义字段后面**，因为会回表进行查询，距离，当使用c索引，查询到了相关列的主键后，需要使用主键索引进行回表查询相关字段，因为主键索引本来就是根据a，b进行排序的，所以实际上就是c，a，b索引。
不显示是因为去重的原因。

所以，当主键是(a,b)的时候，
定义为c的索引，实际上是（c,a,b)，**因此c索引已经包含了ca索引了**;
定义为(c,a)的索引，实际上是(c,a,b)

ps 定义为(c,b）的索引，实际上是（c,b,a)



### 锁

- 全局锁
  - 主要用于全库逻辑备份，加全局读锁方法Flush tables with read lock（FTWRL），一致性读视图（single-transaction）可以代替FTWRL来进行备份，但前提是存储引擎需要支持事务，像MyISAM引擎，还是需要使用FTWRL。
    - 当主库上备份时，在备份期间不能执行更新，业务处于中断时间
    - 当从库上备份，那备份期间从库不能执行主库同步过来的binlog，会导致主从延迟。
  - 不建议使用set global readonly=true的方式，避免数据库异常断开导致数据库一直处于只读状态。第二点是，这个值可能会被用于判断其他逻辑，例如主库还是备库。
- 表级锁
  - 表锁 lock tables read/write
  - 元数据锁 MDL，不需要显示使用，在访问一个表的时候会自动加上；
    - 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。（由于隔离级别，读是读一个旧视图）
    - 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。（安全地给热表加字段：使用wait n/nowait语法，如alter table tbl_name nowait add column/alter tbl_name wait n add column）
    - 当你对一个表做DML（CURD）时，获取的是表读锁，当你对一个表做DDL（create、alter、drop）时，获取的是表写锁。

例子

![image](https://user-images.githubusercontent.com/32328586/117699625-a51b4b80-b1f7-11eb-8ad1-fea180495e1e.png)



### 行锁
MyIsam不支持行锁

DDL(create、alter、drop)，获取的是表写锁，也包括行写锁
DML（CURD）获取的是表读锁，CUD获取的是行写锁，R获取的是行读锁

> 在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。

例子：
![image](https://user-images.githubusercontent.com/32328586/127766227-e56fe216-7349-401e-a5d7-0d0a06ec8cd4.png)
实际上事务 B 的 update 语句会被阻塞，直到事务 A 执行 commit 之后，事务 B 才能继续执行。

- 实际应用规则：如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

例子2：
```
假设你负责实现一个电影票在线交易业务，顾客 A 要在影院 B 购买电影票。我们简化一点，这个业务需要涉及到以下操作：
1、从顾客 A 账户余额中扣除电影票价；
2、给影院 B 的账户余额增加这张电影票价；
3、记录一条交易日志。也就是说，要完成这个交易，我们需要 update 两条记录，并 insert 一条记录。

当然，为了保证交易的原子性，我们要把这三个操作放在一个事务中。那么，你会怎样安排这三个语句在事务中的顺序呢？

试想如果同时有另外一个顾客 C 要在影院 B 买票，那么这两个事务冲突的部分就是语句 2 了。因为它们要更新同一个影院账户的余额，需要修改同一行数据。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。所以，如果你把语句 2 安排在最后，比如按照 3、1、2 这样的顺序，那么影院账户余额这一行的锁时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。
```

> 反思例子：例如cmdb修改实例的状态，需要在region视图和云服务视图中进行update操作，但考虑到云服务视图的使用比region视图要多，因此可以先对region视图进行操作（操作中会加行锁），然后再修改云服务视图中的实例记录。


### 幻读
> select * from t where d=5 for update; (使用的是当前读，并且加上写锁。mysql认为，for update已经给当前的行加了写锁，因此没必要再进行快照读，但是这样会造成幻读的问题。)
> 当d列没有对应的索引情况下：
1 : RC级别下,只会在满足条件的行加行锁(直到事务commit/rollback才会释放),不满足的直接释放; 
2 : RR级别下会加行锁 + Gap lock,会将(0,5],(5,10],(10,15]这三个区间间隙锁起来（next-key lock是左开右闭, Gap-Lock是左开右开）;

#### 解决方法
引入gap锁，为了保持bin log的一致性。

间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间。也就是说，我们的表 t 初始化以后，如果用 select * from t for update 要把整个表所有记录锁起来，就形成了 7 个 next-key lock，分别是 (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]。

间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的。因为间隙锁之间不互斥，可能会导致死锁。
间隙锁是在可重复读隔离级别下才会生效的。所以，你如果把隔离级别设置为读提交的话，就没有间隙锁了。但同时，你要解决可能出现的数据和日志不一致问题，需要把 binlog 格式设置为 row。row格式记录的是实际受影响的数据是真实删除行的主键id。row模式下保存的是每一行的前后记录，虽然占空间，但是不会因为保存命令而造成幻读。


大家都用读提交，可是逻辑备份的时候，mysqldump 为什么要把备份线程设置成可重复读呢？
mysqldump 使用参数single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持，可重复读不会影响数据库的正常读写

在备份期间，备份线程用的是可重复读，而业务线程用的是读提交。同时存在两种事务隔离级别，会不会有问题？
没有问题，不管是提交读还是可重复读，都是MVCC支持，唯一的不同，就是生成快照的时间点不同，也就是能够看到的数据版本不同，即使是提交读情况下，多个不同的事物时间不也是这种情况吗，所以相互并不影响。


### 事务
> begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用 start transaction with consistent snapshot 这个命令。

每个事务都会有一个数组，用来记录事务启动瞬间，处于活跃状态（启动了，但还没提交的事务）的所有事务的ID。
然后每一行记录，都会有一个row_trx_id，用来跟当前事务的数据进行比较。来获取当前事务，当前行应该看到的是哪个值。
![image](https://user-images.githubusercontent.com/32328586/127780320-e2b62ad2-500d-4213-a636-a5668f60acdc.png)
事务的数组，活跃的事务的id，包括两个，1、未提交的事务。2、正在执行的事务。

InnoDB 利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。

![image](https://user-images.githubusercontent.com/32328586/127780049-90ce1bb7-565f-4bc2-96ae-067d3cea9200.png)

> 更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”（current read）。
> 其实，除了 update 语句外，select 语句如果加锁，也是当前读。
> 当前读必须加锁

如果事务 B 在更新之前查询一次数据，这个查询返回的 k 的值确实是 1。但是，当它要去更新数据的时候，就不能再在历史版本上更新了，否则事务 C 的更新就丢失了。因此，事务 B 此时的 set k=k+1 是在（1,2）的基础上进行的操作。
因此，事务B读到的是(1,3)


### 索引
#### 唯一索引和普通索引
当服务是写多读少时，使用普通索引，因为有change buffer的存在，会减少大量的io访问。
当服务是读多写少，写完马上要读的场景多时，假如使用普通索引，change buffer不会减少io访问次数，却增加了维护代价。

> 建议尽量选择使用**普通索引**。如果所有的更新后面，都马上伴随着对这个记录的查询，那么你应该关闭 change buffer。而在其他情况下，change buffer 都能提升更新性能。

![image](https://user-images.githubusercontent.com/32328586/127904952-77479649-8c55-499b-8f03-7be28f92b236.png)


索引的选择，是靠优化器来完成的。一个索引上，不同的值越多（不同的值个数称为基数，即基数越大），这个索引的区分度越好。

通过show index命令，可以看到cardinality字段对应的索引基数，但是并不准确，因为mysql采用的是“采样统计方法”。

> analyze table t 命令，可以用来重新统计索引信息


> 加了limit 1 不一定能减少索引扫描的行数，因为优化器也不确定，【得执行才知道】，所以显示的时候还是按照“最多可能扫多少行”来显示。

mysql优化器索引选择错误解决
- 使用force index，但更新不及时，跟索引名强相关，难维护
- 修改语句，引导优化器选择期望的索引。

例子
``` sql
select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 1;
```
可以优化为
``` sql
select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b,a limit 1;
```

之前优化器选择使用索引 b，是因为它认为使用索引 b 可以避免排序（b 本身是索引，已经是有序的了，如果选择索引 b 的话，不需要再做排序，只需要遍历），所以即使扫描行数多，也判定为代价更小。

现在 order by b,a 这种写法，要求按照 b,a 排序，就意味着使用这两个索引都需要排序。因此，扫描行数成了影响决策的主要条件，于是此时优化器选了只需要扫描 1000 行的索引 a。

当然，这种修改并不是通用的优化手段，只是刚好在这个语句里面有 limit 1，因此如果有满足条件的记录， order by b limit 1 和 order by b,a limit 1 都会返回 b 是最小的那一行，逻辑上一致，才可以这么做。

- 删掉误用的索引

#### 前缀索引
例子：
``` sql
mysql> alter table SUser add index index1(email);
或
mysql> alter table SUser add index index2(email(6));
```

##### 优点
1、减少索引的大小，存储空间小，相同的数据页能放下的索引值就越多，搜索的效率也就会越高。

##### 缺点
1、可能会增加扫描的行数，因为前缀索引一对多。
2、无法使用覆盖索引
3、


##### 区分度越高，索引扫描行数越少
可以通过下面sql判断区分度,left函数为从左开始截取字符串。
``` sql
select 
count(distinct left(email,4)）as L4, 
count(distinct left(email,5)）as L5, 
count(distinct left(email,6)）as L6, 
count(distinct left(email,7)）as L7,
from SUser;
```

##### 当某些字符串前缀不好区分时：
1、倒序存储————不推荐使用，比较麻烦
2、新增一个hash字段，然后为hash字段加前缀索引
例子：
>alter table t add id_card_crc int unsigned, add index(id_card_crc);
每次插入新记录的时候，都同时用 crc32() 这个函数得到校验码填到这个新字段。由于校验码可能存在冲突，也就是说两个不同的身份证号通过 crc32() 函数得到的结果可能是相同的，所以你的查询语句 where 部分要判断 id_card 的值是否精确相同。
>select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'

缺点：两种方式都不支持范围搜索，因为1、倒序肯定不会支持范围搜索。2、hash方法，原字段没有索引。


#### redo log 刷新到内存的四种场景
- InnoDB的redo log 写满了，需要把checkpoint往前推进一下，把对应的log刷新到磁盘中。
- 节点内存不足，无法容纳新的数据页，
- 节点比较空闲的时候，负载低的时候
- 正常关闭Mysql的时候

第一种应该避免，因为此时所有的更新都会阻塞。
第二种是常态，InnoDB使用缓冲池（buffer pool）管理内存。而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页呢，就必须将脏页先刷到磁盘，变成干净页后才能复用。

因此，InnoDB有刷脏页的控制策略，用于避免上述两种情况。

- innodb_io_capacity 参数，它会告诉 InnoDB 你的磁盘能力。这个值建议设置成磁盘的 IOPS。
磁盘的 IOPS 可以通过 fio 这个工具来测试，下面的语句是我用来测试磁盘随机读写的命令：
> fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest

- **参数 innodb_max_dirty_pages_pct 是脏页比例上限，默认值是 75%**

![image](https://user-images.githubusercontent.com/32328586/128391100-b9f1433d-bbc4-4f6b-a85b-8d931233813e.png)

影响因素两点
  - 脏页的比例
  - redo log 的写入速度

- innodb_flush_neighbors参数：刷脏页时，会把隔壁也是脏页的数据页一起刷入到磁盘中。意义是可以减少IO操作，机械硬盘建议设置此参数，但假如磁盘是ssd，建议把这个参数设置为0，提高sql语句的相应时间。Mysql 8.0中该参数默认值为0。

#### count
因为MVCC机制的存在，每行数据都有多个版本，每个事务都需要去判断该行是否在本视图内可见。
每一行记录都要判断自己是否对这个会话可见，因此对于 count(\*) 请求来说，InnoDB 只好把数据一行一行地读出依次判断，可见的行才能够用于计算“基于这个查询”的表的总行数。

通过扫描二级索引树，因为b+树的叶子节点才是节点数，所以扫描索引树，可以获取总数量。
> B+树只有叶子结点上有数据，全部遍历其实就是对叶子结点的链表进行遍历。此时如果遍历主键索引树，由于其叶子结点上存放的是完整的行信息，对于一个数据页而言其行密度会比较小，最终导致要扫描的数据页较多，进而IO开销也比较大。如果遍历第二索引树，其叶子结点只存放主键信息，其数据页的行密度比较大，最终扫描的数据页较少，节省了IO开销。

##### 自己保存每个表的总数量
- 适用于频繁增删的表
- 在缓存中存储（容易出现数据不一致，还需要考虑缓存崩溃丢失的场景）
- 在数据库中存储计数（使用事务）

例子：
![image](https://user-images.githubusercontent.com/32328586/128600500-f8fb4c3e-7664-4d03-ad0d-48edc4b0800b.png)

虽然会话 B 的读操作仍然是在 T3 执行的，但是因为这时候更新事务还没有提交，所以计数值加 1 这个操作对会话 B 还不可见。因此，会话 B 看到的结果里， 查计数值和“最近 100 条记录”看到的结果，逻辑上就是一致的。
**但是，要求对数据表的操作，要在事务里，才会有事务的隔离性。**

##### 不同count的用法
count(\)、count(主键 id) 和 count(1) 都表示返回满足条件的结果集的总行数；而 count(字段），则表示返回满足条件的数据行里面，参数“字段”不为 NULL 的总个数。


按照效率排序的话，count(字段)<count(主键 id)<count(1)≈count(\*)，所以我建议你，尽量使用 count(\*)。


### 数据库表的空间回收

> 在 MySQL 8.0 版本以前，表结构是存在以.frm 为后缀的文件里。而 MySQL 8.0 版本，则已经允许把表结构定义放在系统数据表中了。

##### 参数 innodb_file_per_table。从 MySQL 5.6.6 版本开始，它的默认值就是 ON 了。
- 这个参数设置为 OFF 表示的是，表的数据放在系统共享表空间，也就是跟数据字典放在一起；该场景，drop table时，空间是不会回收的。
- 这个参数设置为 ON 表示的是，每个 InnoDB 表数据存储在一个以 .ibd 为后缀的文件中。该场景，drop table时，空间是回回收的，因为直接删除了文件。

##### 删除部分数据的场景
delete 命令其实只是把记录的位置，或者数据页标记为了“可复用”，但磁盘文件的大小是不会变的。

insert场景也会造成数据页存在没利用的场景（即空间可复用，但实际上没被使用），因为页分裂的场景，分裂完后，页的空间不会被完全被使用。

##### 重建表
作用：把空洞的空间去掉，收缩表空间。（类似于jvm的垃圾回收空间整理）

- 法一：新建一张相同的表B，按主键ID递增的顺序，把数据一行一行地从A插入到B中。再把B替换A。
- 法二：使用alter table A engine=InnoDB的命令来重建表。相当于法一。

Mysql 5.6版本开始，引入了Online的DDL，重建表的过程中，使用了row log机制（用来存储insert过程中，表A的增删改操作。），当临时表B生成后，将A对应的row log写入到临时表里。最终表A和表B的数据会一致。

> DDL 之前是要拿 MDL 写锁的，这样还能叫 Online DDL 吗？alter 语句在启动的时候需要获取 MDL 写锁，但是这个写锁在真正拷贝数据之前就退化成读锁了。为什么要退化呢？为了实现 Online，MDL 读锁不会阻塞增删改操作。


如果想要比较安全的操作的话，我推荐你使用 GitHub 开源的 gh-ost 来做。

有同学问到使用 optimize table、analyze table 和 alter table 这三种方式重建表的区别。

- 从 MySQL 5.6 版本开始，alter table t engine = InnoDB（也就是 recreate）默认的就是上面online的流程了；
- analyze table t 其实不是重建表，只是对表的索引信息做重新统计，没有修改数据，这个过程中加了 MDL 读锁；
- optimize table t 等于 recreate+analyze。

>在重建表的时候，InnoDB 不会把整张表占满，每个页留了 1/16 给后续的更新用。也就是说，其实重建表之后不是“最”紧凑的。

1、将表 t 重建一次；
2、插入一部分数据，但是插入的这些数据，用掉了一部分的预留空间；
3、这种情况下，再重建一次表 t，就可能会出现表空间没减少，反而增加的情况。

### 恢复误删的数据
- 使用 delete 语句误删数据行；
- 使用 drop table 或者 truncate table 语句误删数据表；
- 使用 drop database 语句误删数据库；
- 使用 rm 命令误删整个 MySQL 实例。

#### 使用 delete 语句误删数据行
可以使用Flashback工具通过闪回把数据恢复，原理是通过“修改”binlog，拿回原库执行。需要确保binlog_format=row和binlog_row_image=FULL。
具体“修改”：
1、对于insert语句，对应的binlog event类型是Write_rows_event，把他改为Delete_rows_event即可。
2、对于delete语句，将Delete_rows_event改为Write_rows_event。
3、对于updata语句，bin log记录了数据行修改前和修改后的值，把Update_rows对应的两行对调位置即可。


不建议你直接在主库上执行这些操作。
恢复数据比较安全的做法，是恢复出一个备份，或者找一个从库作为临时库，在这个临时库上执行这些操作，然后再将确认过的临时库的数据，恢复回主库。

#### 误删库/表
使用全量备份，加增量日志的方式

1、取最近一次全量备份，假设这个库是一天一备，上次备份是当天 0 点；
2、用备份恢复出一个临时库；
3、从日志备份里面，取出凌晨 0 点之后的日志；
4、把这些日志，除了误删除数据的语句外，全部应用到临时库。


![image](https://user-images.githubusercontent.com/32328586/131380889-9fa0132c-e095-4e0b-8578-577f4ad65b45.png)

一种加速的方法是，在用备份恢复出临时实例之后，将这个临时实例设置成线上备库的从库。
好处是：1、在 start slave 之前，先通过执行﻿﻿change replication filter replicate_do_table = (tbl_name) 命令，就可以让临时库只同步误操作的表；
2、可以用上并行复制技术，来加速整个数据恢复过程




### 排序
##### 全字段排序
sort_buffer_size，即Mysql为每个线程开辟的内存大小。
- 如果要排序的数据量小于sort_buffer_size，排序则在内存中完成。
- 如果要排序的数据量大于sort_buffer_size，则使用磁盘临时文件进行排序，在sort buffer中排好序然后把结果存入临时文件，最后采用**归并排序**合并成一个大的临时文件。

判断一个排序语句是否使用了临时文件： 从MySQL5.6版本开始，optimizer_trace 可支持把MySQL查询执行计划树打印出来。
``` sql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```



Mysql 8.0前，OPTIMIZER_TRACE 的结果中，number_of_tmp_files 表示的是，排序过程中使用的临时文件数。


##### rowid排序
当mysql认为单行长度太大，会采用另一种排序方法，因为内存能同时放下的行数少，需要分成更多的临时文件，效率低。

> set max_length_for_sort_data = 16;   # 控制排序行数据的长度参数，16代表，假如一行的长度超过16，则mysql认为单行太大。

算法不同点：在于只会取主键和需要排序的字段，在内存中进行排序和limit后，再通过主键回原表取出对应的数据。

设计思想：**如果内存够，就要多利用内存，尽量减少磁盘访问。**


##### 排序优化举例
当前有联合索引：city_name(city, name)

> sql为：select * from t where city in ('杭州',"苏州") order by name limit 100;

虽然有 (city,name) 联合索引，对于单个 city 内部，name 是递增的。但是由于这条 SQL 语句不是要单独地查一个 city 的值，而是同时查了"杭州"和" 苏州 "两个城市，因此所有满足条件的 name 就不是递增的了。也就是说，这条 SQL 语句需要排序。

使用java代码避免排序：
1、执行 select * from t where city=“杭州” order by name limit 100; 这个语句是不需要排序的，客户端用一个长度为 100 的内存数组 A 保存结果。
2、执行 select * from t where city=“苏州” order by name limit 100; 用相同的方法，假设结果被存进了内存数组 B。
3、现在 A 和 B 是两个有序数组，然后你可以用归并排序的思想，得到 name 最小的前 100 值，就是我们需要的结果了。

内存临时表
> 如果你创建的表没有主键，或者把一个表的主键删掉了，那么 InnoDB 会自己生成一个长度为 6 字节的 rowid 来作为主键

磁盘临时表
tmp_table_size 这个配置限制了内存临时表的大小，默认值是 16M。如果临时表大小超过了 tmp_table_size，那么内存临时表就会转成磁盘临时表。



#### 性能
- 条件字段函数操作
where条件里面的字段，不要使用函数。如果使用了，会导致索引失效。
> 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。

- 隐式类型转换
假如字段原本是varchar类型，而你传入的是整型，需要做类型转换cast(tradeid as signed int);。编译器会放弃走索引，进行全表扫描。


转换规则：字符串和数字做比较的话，是将字符串转换成数字。


这里有一个简单的方法，看 select “10” > 9 的结果：
- 如果规则是“将字符串转成数字”，那么就是做数字比较，结果应该是 1；
- 如果规则是“将数字转成字符串”，那么就是做字符串比较，结果应该是 0。
- 
![image](https://user-images.githubusercontent.com/32328586/129396376-48801b69-6e32-4c46-b058-14eda9f06450.png)



反例子：
id 的类型是 int，如果执行语句：select * from tradelog where id="83126";  不会全表扫描，依然会走id的索引。
因为字符串和数字做比较的话，是将字符串转换成数字。因此相当于select * from tradelog where id=83126。

- 隐式字符编码转换
表之间字符集不同导致关联表的时候，没办法用上索引。
例如一个表是utf8，一个表是utf8mb4，两个表关联时，没办法使用索引。

![image](https://user-images.githubusercontent.com/32328586/129397406-004ac55d-e844-42d9-84e3-287b32921dc9.png)

> 这个设定很好理解，utf8mb4 是 utf8 的超集。类似地，在程序设计语言里面，做自动类型转换的时候，为了避免数据在转换过程中由于截断导致数据错误，也都是“按数据长度增加的方向”进行转换的。

写法等同于
``` sql
select * from trade_detail where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value;
```
这就再次触发了我们上面说到的原则：对索引字段做函数操作，优化器会放弃走树搜索功能。


mysql>select l.operator from tradelog l , trade_detail d where d.tradeid=l.tradeid and d.id=4;


优化方案：
1、修改字符集，统一一下
2、mysql> select d.* from tradelog l , trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2;





简单sql，长时间不返回：
1、是否有索引
2、等待MDL（MetaData Lock）
  - 可以使用 show processlist命令查看：
![image](https://user-images.githubusercontent.com/32328586/129484128-8a32f02e-c9ab-4fff-afb9-cec82640cf29.png)
  - 通过查询 sys.schema_table_lock_waits 这张表，我们就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。
![image](https://user-images.githubusercontent.com/32328586/129484130-c5e2e805-30fc-48eb-8bd3-bb0c89c12661.png)

3、等flush
4、行锁
> mysql> select * from t where id=1 lock in share mode;
由于访问 id=1 这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的 select 语句就会被堵住。
![image](https://user-images.githubusercontent.com/32328586/129484254-428f5ab3-2e85-4db1-bc5e-f59137351eae.png)


##### 连接数
参数：max_connections
调高 max_connections 的值。但这样做是有风险的。因为设计 max_connections 这个参数的目的是想保护 MySQL，如果我们把它改得太大，让更多的连接都可以进来，那么系统的负载可能会进一步加大，大量的资源耗费在权限验证等逻辑上，结果可能是适得其反，已经连接的线程拿不到 CPU 资源去执行业务的 SQL 请求。

wait_timeout 参数:一个线程空闲 wait_timeout 这么多秒之后，就会被 MySQL 直接断开连接。show variables like 'wait_timeout';


如果现在数据库确认是被连接行为打挂了，那么一种可能的做法，是让数据库跳过权限验证阶段。跳过权限验证的方法是：重启数据库，并使用–skip-grant-tables 参数启动。这样，整个 MySQL 会跳过所有的权限验证阶段，包括连接过程和语句执行过程在内。

##### 慢查询
###### 紧急加索引：
1、在备库 B 上执行 set sql_log_bin=off，也就是不写 binlog，然后执行 alter table 语句加上索引。关闭 binlog 因为如果表数据很大，alter 之后会产生大量的日志，并且会讲变更同步到主库
2、执行主备切换；
3、这时候主库是 B，备库是 A。在 A 上执行 set sql_log_bin=off，然后执行 alter table 语句加上索引。

###### 紧急改写sql
MySQL 5.7 提供了 query_rewrite 功能，可以把输入的一种语句改写成另外一种模式。
比如，语句被错误地写成了 select * from t where id + 1 = 10000，你可以通过下面的方式，增加一个语句改写规则。
``` sql
mysql> insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");

call query_rewrite.flush_rewrite_rules();
```

##### QPS突增
可以把压力最大的sql直接重写为select 1返回，或者limit1.




##### 可重复读隔离级别：
加锁规则：两个原则，两个优化，一个bug
###### 原则
- 1、加锁的基本单位是 next-key lock。
- 2、查找过程中访问到的对象才会加锁。
###### 优化
- 1、索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
- 2、索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
###### bug
- 1、唯一索引上的范围查询会访问到不满足条件的第一个值为止。（说是bug，因为唯一索引没必要这样做）

``` sql
# 初始建表语句
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

1、先根据查询条件判断用的是**唯一索引（包括主键索引）**还是**普通索引**
2、根据select的字段，判断是否覆盖索引，假如是覆盖索引，则不会锁主表。
3、判断next-key lock范围
4、假如找到了，唯一索引场景下，只会锁一行，退化为行锁。


###### 案例一：等值查询间隙锁
![image](https://user-images.githubusercontent.com/32328586/129607802-68d6dcbc-c98e-4c52-807c-d7865e54cc22.png)

1，根据原则1，加锁单位是 next-key lock，由于id=7的记录不存在，需要找到不满足条件的第一个值，即10，于是session A 加锁范围就是 (5,10]；
2、根据优化 2，这是一个等值查询 (id=7)，而 id=10 不满足查询条件，next-key lock 退化成间隙锁，因此最终加锁的范围是 (5,10)。
3、所以，session B 要往这个间隙里面插入 id=8 的记录会被锁住，但是 session C 修改 id=10 这行是可以的。

###### 案例二：非唯一索引等值锁
![image](https://user-images.githubusercontent.com/32328586/129607815-b540531e-5ea7-4b5e-bd25-fe94504c0253.png)

1、根据原则 1，加锁单位是 next-key lock，因此会给 (0,5]加上 next-key lock。
2、c 是普通索引，因此仅访问 c=5 这一条记录是不能马上停下来的，需要向右遍历，查到 c=10 才放弃。根据原则 2，访问到的都要加锁，因此要给 (5,10]加 next-key lock。
3、但是同时这个符合优化 2：等值判断，向右遍历，最后一个值不满足 c=5 这个等值条件，因此退化成间隙锁 (5,10)。
4、根据原则 2 ，只有访问到的对象才会加锁，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么 session B 的 update 语句可以执行完成。
5、但 session C 要插入一个 (7,7,7) 的记录，就会被 session A 的间隙锁 (5,10) 锁住。

> 需要注意，在这个例子中，lock in share mode 只锁覆盖索引，但是如果是 for update 就不一样了。 执行 for update 时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

###### 案例三：主键索引范围锁
![image](https://user-images.githubusercontent.com/32328586/129947074-d9404d7b-df41-4975-b4b0-28f269a20413.png)
1、开始执行的时候，要找到第一个 id=10 的行，因此本该是 next-key lock(5,10]。 根据优化 1， 主键 id 上的等值条件，退化成行锁，只加了 id=10 这一行的行锁。
2、范围查找就往后继续找，找到 id=15 这一行停下来，因此需要加 next-key lock(10,15]。这里是next-key lock的（10,15]，不是间隙锁。

###### 案例四：非唯一索引范围锁
![image](https://user-images.githubusercontent.com/32328586/129947218-e1d3ef8c-2f66-4bd9-9ee4-9050136fb12c.png)

这次 session A 用字段 c 来判断，加锁规则跟案例三唯一的不同是：在第一次用 c=10 定位记录的时候，索引 c 上加了 (5,10]这个 next-key lock 后，由于索引 c 是非唯一索引，没有优化规则，也就是说不会蜕变为行锁，因此最终 sesion A 加的锁是，索引 c 上的 (5,10] 和 (10,15] 这两个 next-key lock。


###### 案例五：唯一索引范围锁bug
![image](https://user-images.githubusercontent.com/32328586/129948241-4eb6867a-d37d-4951-816d-35686c50a14c.png)

1、session A 是一个范围查询，按照原则 1 的话，应该是索引 id 上只加 (10,15]这个 next-key lock，并且因为 id 是唯一键，所以循环判断到 id=15 这一行就应该停止了。
2、但是实现上，InnoDB 会往前扫描到第一个不满足条件的行为止，也就是 id=20。而且由于这是个范围扫描，因此索引 id 上的 (15,20]这个 next-key lock 也会被锁上。

> 这里没有等值查询的过程，是id>10 and id<=15的范围查询过程，没有优化规则。 。。。。。。。 如果查询条件改为：select * from t where id>=10 and id<=15 for update; 那么就有了等值查询(id=10)和范围查询(id>10 and id<=15)，其中等值查询可以应用优化一规则，那么锁的范围会是10,(10,15],(15,20]。

###### 案例六：非唯一索引上存在"等值"的例子
> insert into t values(30,10,30);新增一行
索引C的样子：
![image](https://user-images.githubusercontent.com/32328586/129948888-0a659f60-9f7d-4038-8568-b9c8d7255d52.png)

![image](https://user-images.githubusercontent.com/32328586/129948931-9f0731b0-86c1-47ea-8ee6-9674b9184986.png)

1、session A 在遍历的时候，先访问第一个 c=10 的记录。同样地，根据原则 1，这里加的是 (c=5,id=5) 到 (c=10,id=10) 这个 next-key lock。
2、然后，session A 向右查找，直到碰到 (c=15,id=15) 这一行，循环才结束。根据优化 2，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成 (c=10,id=10) 到 (c=15,id=15) 的间隙锁。

###### 案例七：limit语句加锁

![image](https://user-images.githubusercontent.com/32328586/129949596-b3aebe79-ed67-4487-9759-48deb4f00711.png)
1、session A 的 delete 语句加了 limit 2。你知道表 t 里 c=10 的记录其实只有两条，因此加不加 limit 2，删除的效果都是一样的，但是加锁的效果却不同。可以看到，session B 的 insert 语句执行通过了，跟案例六的结果不同。
2、因为，案例七里的 delete 语句明确加了 limit 2 的限制，因此在遍历到 (c=10, id=30) 这一行之后，满足条件的语句已经有两条，循环就结束了。
3、因此，索引 c 上的加锁范围就变成了从（c=5,id=5) 到（c=10,id=30) 这个前开后闭区间，如下图所示：

> 这个例子对我们实践的指导意义就是，在删除数据的时候尽量加 limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。

###### 案例八：一个死锁的例子
![image](https://user-images.githubusercontent.com/32328586/129949975-fa3cb556-7534-485b-bae4-0dd8674be4f1.png)

1、session A 启动事务后执行查询语句加 lock in share mode，在索引 c 上加了 next-key lock(5,10] 和间隙锁 (10,15)；
2、session B 的 update 语句也要在索引 c 上加 next-key lock(5,10] ，进入锁等待；（session B 的“加 next-key lock(5,10] ”操作，实际上分成了两步，先是加 (5,10) 的间隙锁，加锁成功；然后加 c=10 的行锁，这时候才被锁住的。）
3、然后 session A 要再插入 (8,8,8) 这一行，被 session B 的间隙锁锁住。由于出现了死锁，InnoDB 让 session B 回滚。



***


### explain
![image](https://user-images.githubusercontent.com/32328586/129395681-38e68a96-4d4e-4285-8421-45160c508ce0.png)

key="t_modified"表示的是，使用了 t_modified 这个索引；

Extra 字段的 Using index，表示的是使用了覆盖索引。


## 数据可靠性
在备库执行show slave status命令，返回结果会显示seconds_behind_master，用于表示当前备库延迟了多少秒。

**主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产 binlog 的速度要慢。**

- 备库性能比主库差。
- 备库压力大。通常读写分离后，主库提供写能力，备库提供读能力，备库上的查询耗费了大量的CPU资源，影响的同步速度，造成主备延迟。
  - 解决方法1：一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
  - 通过 binlog 输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力。
- 大事务。主库上必须等事务执行完成才会写入 binlog，再传给备库。所以，如果一个主库上的语句执行 10 分钟，那这个事务很可能就会导致从库延迟 10 分钟。

典型大事务场景：
- 一次性使用delete语句删除大量数据
- 大表DDL（计划内的 DDL，建议使用 gh-ost 方案）
- 备库的并行复制能力

### 并行复制
![image](https://user-images.githubusercontent.com/32328586/130478174-c3915f79-f056-462f-b877-6b4f3241b39f.png)


coordinator在分发时的基本要求
- 不能造成更新覆盖。即更新同一行的两个事务，必须分发到同一个worker中
- 同一个事务不能被拆开，必须放到同一个worker中。

#### 按表分发策略
1、如果跟所有 worker 都不冲突，coordinator 线程就会把这个事务分配给最空闲的 woker;
2、如果跟多于一个 worker 冲突，coordinator 线程就进入等待状态，直到和这个事务存在冲突关系的 worker 只剩下 1 个；
3、如果只跟一个 worker 冲突，coordinator 线程就会把这个事务分配给这个存在冲突关系的 worker。

按表分发的方案，在多个表负载均匀的场景里应用效果很好。但是，如果碰到热点表，比如所有的更新事务都会涉及到某一个表的时候，所有事务都会被分配到同一个 worker 中，就变成单线程复制了。

#### 按行分发策略
> 要求binlog格式必须是row格式，因为原则是，如果两个事务没有更新相同的行，他们可以在备库上并行地执行。但判断事务更新哪些行，需要用到row格式存的主键id。

- 要能够从 binlog 里面解析出表名、主键值和唯一索引的值。也就是说，主库的 binlog 格式必须是 row；
- 表必须有主键；
- 不能有外键。表上如果有外键，级联更新的行不会记录在 binlog 中，这样冲突检测就不准确。


#### MySQL 5.6 版本的并行复制策略，只支持按库并行。

#### MariaDB的并行复制策略，参考redo log的组提交（group commit）
- 能够在同一组里提交的事务，一定不会修改同一行；
- 主库上可以并行执行的事务，备库上也一定是可以并行执行的。
- 核心：所有处于commit状态的事务可以并行（一起commit的事务可以并行）

1、在一组里面一起提交的事务，有一个相同的 commit_id，下一组就是 commit_id+1；
2、commit_id 直接写到 binlog 里面；
3、传到备库应用的时候，相同 commit_id 的事务分发到多个 worker 执行；
4、这一组全部执行完成后，coordinator 再去取下一批。


缺点：这个策略有一个问题，它并没有实现“真正的模拟主库并发度”这个目标。在主库上，一组事务在 commit 的时候，下一组事务是同时处于“执行中”状态的。而MariaDB在并行复制的时候，要等第一组事务完全执行完成后，第二组事务才能开始执行，这样系统的吞吐量就不够。


#### MySQL 5.7 的并行复制策略
提供参数slave-parallel-type来控制并行复制策略
- 配置为 DATABASE，表示使用 MySQL 5.6 版本的按库并行策略；
- 配置为 LOGICAL_CLOCK，表示的就是类似 MariaDB 的策略。不过，MySQL 5.7 这个策略，针对并行度做了优化。

![image](https://user-images.githubusercontent.com/32328586/130481015-9f83ec0e-e06f-48f2-abdb-ce2dbda20dbf.png)

其实，不用等到 commit 阶段，只要能够到达 redo log prepare 阶段，就表示事务已经通过锁冲突的检验了。

因此，MySQL 5.7 并行复制策略的思想是：
- 同事处于prepare状态的事务，在备库执行是可以并行的
- 处于prepare状态的事务，与处于commit状态的事务之间，在备库执行时也是可以并行的。


![image](https://user-images.githubusercontent.com/32328586/130481304-b0406ca4-6995-42bb-9227-fe94a54c8e14.png)


#### MySQL 5.7.22 的并行复制策略
新增了一个参数 binlog-transaction-dependency-tracking，用来控制是否启用这个新策略
- COMMIT_ORDER，表示的就是前面介绍的，根据同时进入 prepare 和 commit 来判断是否可以并行的策略。
- WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的 hash 值，组成集合 writeset。如果两个事务没有操作相同的行，也就是说它们的 writeset 没有交集，就可以并行。
- WRITESET_SESSION，是在 WRITESET 的基础上多了一个约束，即在主库上同一个线程先后执行的两个事务，在备库执行的时候，要保证相同的先后顺序。

对比自己实现的按行分发策略，优点如下：
- writeset 是在主库生成后直接写入到 binlog 里面的，这样在备库执行的时候，不需要解析 binlog 内容（event 里的行数据），节省了很多计算量；
- 不需要把整个事务的 binlog 都扫一遍才能决定分发到哪个 worker，更省内存；
- 由于备库的分发策略不依赖于 binlog 内容，所以 binlog 是 statement 格式也是可以的。

缺点：
对于“表上没主键”和“外键约束”的场景，WRITESET 策略也是没法并行的，也会暂时退化为单线程模型。

![image](https://user-images.githubusercontent.com/32328586/130483845-c5ccd2cc-7765-4373-a2d4-757488a80cca.png)



### 延迟读的解决办法
- 强制走主库方案
- sleep方案
- 判断主备无延迟方案
- 配合semi-sync方案
- 等主库位点方案
- 等GTID方案

#### 强制走主库方案
查询请求分类，分为走主库、走从库的请求。这个方案是用得最多的。

#### sleep方案
主库更新后，读从库前先sleep一下，具体方案，类似于执行一条select sleep(1)命令。

1、具体sleep多少秒，不确定
2、不保证精确性

#### 判断主备无延迟方案
show slave status 结果里的 seconds_behind_master 参数的值，可以用来衡量主备延迟时间的长短。

1、确保主备无延迟的方法：每次从库执行查询请求前，先判断seconds_behind_master是否已经等于0，如果不等于0，就必须等到这个参数变为0才能执行查询请求。参数单位是秒，不太准确，下面的两种方法比较准确。
2、对比位点确保主备无延迟。
- Master_Log_File和Read_Master_Log_Pos，表示读到的主库的最新位点；
- Relay_Master_Log_File和Exec_Master_Log_Pos表示备库执行的最新位点。
如果Master_Log_File 和 Relay_Master_Log_File、Read_Master_Log_Pos 和 Exec_Master_Log_Pos 这两组值完全相同，就表示接收到的日志已经同步完成。

![image](https://user-images.githubusercontent.com/32328586/131007842-844555d9-3727-4c20-a184-e73ff21be5d0.png)

3、对比GTID集合确保主备无延迟
- Auto_Position=1，表示这对主备关系使用了GTID协议。
- Retrieved_Gtid_Set，是备库收到的所有日志GTID集合；
- Executed_Gtid_Set，是备库所有已经执行完成的GTID集合。

如果两个集合相同，表示备库接收到的日志都已经同步完成。

#### 配合semi-sync
问题：
![image](https://user-images.githubusercontent.com/32328586/131008569-798fa307-0fbc-4041-9a72-59c5a296c64d.png)

解决：引入半同步复制，即semi-sync replication
1、事务提交时，主库把binlog发给从库；
2、从库收到binlog后，发回给主库一个ack，表示收到了（收到了就可以，因为有位点、GTID等判断收到后是否执行完成）；
3、主库收到这个ack以后，才能给客户端返回“事务完成”的确认。

缺点
1、一主多从的时候，在某些从库执行查询请求会存在过期读的现象；因为收到第一个ack后，就给客户端返回事务完成了。
2、在持续延迟的情况下，可能出现过度等待的问题。

#### 等主库位点方案

``` sql
select master_pos_wait(file, pos[,timeout]);
```
命令逻辑：
1、这条命令是在从库执行；
2、参数file和pos指主库上的文件名和位置。
3、timeout可选，设置为正整数N，表示这个函数最多等待N秒。

**这个命令正常返回的结果是一个正整数 M，表示从命令开始执行，到应用完 file 和 pos 表示的 binlog 位置，执行了多少事务。**

可以如下逻辑：
1、trx1事务更新完成后，马上执行show master status得到当前主库执行到的File和Position。
2、选定一个从库执行查询语句
3、在从库上执行select master_pos_wait(File,Position,1);
4、如果返回值是 >= 0的正整数，则在这个从库执行查询语句；
5、否则到主库执行查询语句。


#### GTID方案
``` sql
select wait_for_executed_gtid_set(gtid_set,1);
```
这条命令的逻辑是：
1、等待，直到这个库执行的事务中包含传入的 gtid_set，返回 0；
2、超时返回 1。

在前面等位点的方案中，我们执行完事务后，还要主动去主库执行 show master status。而 MySQL 5.7.6 版本开始，允许在执行完更新类事务后，把这个事务的 GTID 返回给客户端，这样等 GTID 的方案就可以减少一次查询。

所以逻辑变成了如下：
1、trx1事务，在主库完成更新后，从返回包直接获取这个事务的GTID，记为gtid1；
2、选定一个从库执行查询语句
3、在从库上执行select wait_for_executed_gtid_set(gtid1,1);
4、如果返回值是0，则在这个从库执行查询语句；
5、否则，到主库执行查询语句。


怎么能够让 MySQL 在执行事务后，返回包中带上 GTID 呢？
只需要将参数 session_track_gtids 设置为 OWN_GTID，然后通过 API 接口 mysql_session_track_get_first 从返回包解析出 GTID 的值即可。




### 判断表是否正常
#### select 1

并发连接和并发查询
- 并发连接：max_connection
- 并发查询：innodb_thread_concurrency

show processlist 的结果里，看到的几千个连接，指的就是并发连接。而“当前正在执行”的语句，才是我们所说的并发查询。
线程进入锁等待后，并发线程的计数会减一。

当同时在执行的语句超过了设置的 innodb_thread_concurrency 的值，这时候系统其实已经不行了，但是通过 select 1 来检测系统，会认为系统还是正常的。


#### 查表判断
在系统库创建一个表，例如命名为health_check，里面只放一行数据，然后定期执行
``` sql
select *from mysql.health_check;
```

该方法能检测由于并发线程过多导致的数据库不可用情况。
但存在一种可能，即磁盘空间满了后，更新事务要写binlog，但因为空间不足，导致更新语句和事务提交的commit语句被堵住。但是系统还是可以读数据。

疑问？像发现系统并发查询线程数不足后，这种要怎么处理？？？

#### 更新判断
常见做法是放一个 timestamp 字段，用来表示最后一次执行检测的时间。
``` sql
update mysql.health_check set t_modified=now();
```

假如主备关系为双M结构，可能会出现行冲突，可能会导致主备同步停止。
原因
> 如果表中只有一列t_modified，那么在主库修改t_modified事务还没有提交，而备库的修改t_modified的事务先提交并且binlog传给了主库，此时传给主库的binlog回放会被阻塞。等主库修改t_modified事务提交之后，binlog传给备库，以及备库的binlog应用的时候发现要更新的数据不能存在了，主备复制报1032的错误。导致主备复制停止 如果表中多存在一个主键列，那么上面的情况不会发生，但是主备的数据会不一致（这个是重点）。

为了让主备之间的更新不产生冲突，我们可以在 mysql.health_check 表上存入多行数据，并用 A、B 的 server_id 做主键。
``` sql
mysql> CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```

#### 内部统计
MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间。


### 全表扫描
#### server端
MySQL 是“边读边发的”。
服务端并不需要保存一个完整的结果集。取数据和发数据的流程是这样的：
- 获取一行，写到 net_buffer 中。这块内存的大小是由参数 net_buffer_length 定义的，默认是 16k。
- 重复获取行，直到 net_buffer 写满，调用网络接口发出去。
- 如果发送成功，就清空 net_buffer，然后继续取下一行，并写入 net_buffer。
- 如果发送函数返回 EAGAIN 或 WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满了，进入等待。直到网络栈重新可写，再继续发送。

mysql_sotre_result把结果保存在客户端本地，直接把所有结果读过来； mysql_use_result则是客户端一行一行的从Server读取数据，如果每行数据都有业务处理逻辑的话Server就要等待，会造成长事务。
假设有一个业务的逻辑比较复杂，每读一行数据以后要处理的逻辑如果很慢，而且返回数据集不大，建议使用mysql_store_result。否则容易导致客户端要过很久才会取下一行数据，导致服务端发送阻塞。

如果你在自己负责维护的 MySQL 里看到很多个线程都处于“Sending to client”这个状态，如果要快速减少处于这个状态的线程的话，将 net_buffer_length 参数设置为一个更大的值是一个可选方案。


#### 引擎端 InnoDB
change buffer 还有加速查询的作用，加速效果，依赖于一个重要的指标，即：内存命中率。可以通过 show engine innodb status

InnoDB Buffer Pool 的大小是由参数 innodb_buffer_pool_size 确定的，一般建议设置成可用物理内存的 60%~80%。

InnoDB 内存管理用的是最近最少使用 (Least Recently Used, LRU) 算法，这个算法的核心就是淘汰最久未使用的数据。

针对全表扫描冷数据，做了优化：在 InnoDB 实现上，按照 5:3 的比例把整个 LRU 链表分成了 young 区域和 old 区域。图中 LRU_old 指向的就是 old 区域的第一个位置，是整个链表的 5/8 处。也就是说，靠近链表头部的 5/8 是 young 区域，靠近链表尾部的 3/8 是 old 区域。


### join
``` sql
# 表结构

CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

#### Index Nested-Loop join
> select * from t1 straight_join t2 on (t1.a=t2.a);

由于驱动表走全表扫描，然后被驱动表走索引扫描。假设驱动表有N行数据，被驱动表有M行数据。
复杂度类似于N + N*2*log2M     （2是因为走了两次索引，一次a索引树，一次主键索引树）

两个结论：
- 使用join语句，性能比强行拆成多个单表执行SQL语句的性能要好。
- 如果使用join语句的话，需要让小表做驱动表。

#### Simple Nested-Loop Join
> select * from t1 straight_join t2 on (t1.a=t2.b);

由于表 t2 的字段 b 上没有索引，因此再用图 2 的执行流程时，每次到 t2 去匹配的时候，就要做一次全表扫描。

复杂度相当于M\*N，如果 t1 和 t2 都是 10 万行的表（当然了，这也还是属于小表的范围），就要扫描 100 亿行,过于笨重。

当然，MySQL 也没有使用这个 Simple Nested-Loop Join 算法，而是使用了另一个叫作“Block Nested-Loop Join”的算法，简称 BNL。


#### Block Nested-Loop Join
被驱动表上没有可用的索引，算法的流程是这样的：








