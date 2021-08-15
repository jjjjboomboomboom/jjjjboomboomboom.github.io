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






### explain
![image](https://user-images.githubusercontent.com/32328586/129395681-38e68a96-4d4e-4285-8421-45160c508ce0.png)

key="t_modified"表示的是，使用了 t_modified 这个索引；

Extra 字段的 Using index，表示的是使用了覆盖索引。
