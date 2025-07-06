# Mysql面试总结

## 1.MyISAM和InnoDB有什么区别？

1. MyISAM不支持行级锁，在高并发的场景下表现不如innoDB
2. 不支持事务，INNODB支持事务，有回滚和提交事务的能力
3. MyISAM不支持外键，INNODB支持外键，不过使用外键会让效率降低，所以一般是在业务代码里面实现数据的一致性。
4. MyISAM不支持数据库异常崩溃的安全恢复，而innodb支持使用redo log进行数据的恢复。

## 2.对mysql事务的隔离级别是否了解？

**READ-UNCOMMITTED(读取未提交)** ：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。

**READ-COMMITTED(读取已提交)** ：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。

**REPEATABLE-READ(可重复读)** ：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。

**SERIALIZABLE(可串行化)** ：最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。

## 3.mysq的日志你知道哪些？

**错误日志**：记录mysql服务器运行过程中的报错、警告、启动和关闭的日志

**查询日志**：记录所有的查询sql语句

**慢查询日志**：记录执行时间超过配置时间的SQL语句

配置：

```
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql_slow.log
long_query_time = 2  # 记录执行超过 2 秒的查询
```

**二进制日志**：记录对数据库进行修改的sql语句，主要是用来数据恢复和主从复制。

配置：

```
[mysqld]
log_bin = mysql-bin
binlog_format = ROW  # ROW/MIXED/STATEMENT
expire_logs_days = 7  # 设置 binlog 过期时间
```

**事务日志（Redo Log 和 Undo Log）**

- **Redo Log（重做日志）**：用于崩溃恢复，确保事务提交后即使崩溃也能恢复数据。
- **Undo Log（回滚日志）**：用于事务回滚和 MVCC（多版本并发控制）。

## 4.对mysql的mvcc有了解吗？

### **MVCC 的核心概念**

MVCC 主要依赖以下两个日志来管理数据的多个版本：

1. **Undo Log（回滚日志）**
   - 每次对数据进行 `UPDATE` 或 `DELETE` 时，旧版本的数据会被存入 **Undo Log**。
   - 这样，即使数据在事务中被修改，其他事务仍然可以通过 **Undo Log 看到之前的版本**。
2. **Read View（读视图）**
   - 事务读取数据时，InnoDB 会创建一个**快照（Snapshot）**，这个快照是基于 Undo Log 生成的。
   - 事务可以通过 **快照** 读取数据，而不会被其他事务的修改影响。

### **MVCC 的隔离级别**

MVCC 适用于 **READ COMMITTED**（读已提交）和 **REPEATABLE READ**（可重复读）两种隔离级别：

- **READ COMMITTED（读已提交）**
  - 事务每次查询时都会生成新的 **Read View**，因此可能会读到其他事务已经提交的最新数据。
  - **可能出现 "不可重复读"**，即同一个事务中，两次查询可能得到不同的结果。
- **REPEATABLE READ（可重复读，MySQL 默认）**
  - 事务在**第一次查询时生成 Read View**，之后的查询都会基于这个视图，不会受到其他事务提交的影响。
  - 这样保证了**事务期间的查询结果一致**，避免 "不可重复读"。

## 5.行级锁的使用有什么注意事项？

行级锁是针对索引字段加的锁，如果在sql语句的where中没有使用到索引的话会升级成表锁。

## 6.你是如何做sql优化的？

1、首先，先查一下慢sql日志，找到执行慢的sql语句，然后用explain看一下索引是否命中，如果没命中需要检查一下sql语句。索引中也可以加上查询的字段避免回表

2、然后再看一下是否用到外部排序，一般如果用到外部排序就是就是使用order的情况，这时候考虑加索引，如果有索引，再检查一下sql语句是否让索引没生效的地方，比如最左匹配原则，一个升序、一个降序。

3、如果sql语句本身没有问题，就要检查一下业务代码，是否由事务时间过长导致的。如果是事务过长导致，可以大事务拆分成小事务，可以把一部分的事务改成异步的，保证最终一致性即可。

**方案1：监控事务执行时间**

- **通过`information_schema.INNODB_TRX`表**

  sql

  复制

  ```
  -- 查询运行超过N秒的事务
  SELECT 
    trx_id, 
    trx_started, 
    TIMEDIFF(NOW(), trx_started) AS duration,
    trx_query  -- 当前正在执行的SQL（可能为空）
  FROM information_schema.INNODB_TRX
  WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;  -- 超过60秒的事务
  ```

# MYSQL实战45讲

## 7.全局锁和表锁

### 7.1 全局锁：

​	对整个数据库实例进行加锁，对数据库实例加全局读锁的方法，命令是Flush tables with read lock (FTWRL)，使用完指令后数据库不再接受DML操作和DDL操作。

​	全局锁的典型使用场景是，做全库逻辑备份。也就是把整库每个表都select出来存成文本。如果不加全局读锁，那么在备份期间能够继续修改数据和表结构，导致备份的数据状态和实际的数据库状态不一致。

​	但是加全局的读锁有很明显的缺点：

- 如果在主数据库进行备份，那么数据库在备份期间无法处理业务
- 如果在从数据库进行备份，这期间从数据库无法同步主数据库的binlog

​	解决办法就是mysql官方自带的mysqldump逻辑备份工具，当mysqldump使用参数–single-transaction的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是可以正常更新的。

​	既然有了mysqldump为什么还需要Flush tables with read lock (FTWRL)，因为MYISAM这些引擎是不支持事务的。

​	如果只是要求全局只读的话为什么要不使用set global readonly=true指令：

- set global readonly=true一般有其他的业务逻辑，比如用来区分主数据库或者从数据库，使用这个指令可能会影响其他的业务逻辑判断
- Flush tables with read lock (FTWRL)指令如果在客户端异常断开时会自动释放锁数据库就回到正常支持更新的状态，但是set global readonly=true客户端异常断开后会导致数据库无法使用。

### 7.2表级锁

​	MYSQL的表级锁分为两种，一种是表锁，一种是元数据锁（锁表结构的）。

​	表锁的加锁方式，lock tables read或lock tables write，可以使用unlock tables...进行解锁，当客户端主动释放连接的时候也会主动释放锁。当线程加读锁时，其他的线程不允许对表进行DDL操作，但是可以进行读操作也就是重复加读锁，两个读线程之间不冲突。当线程加写锁的时候，其他线程不允许进行读写的操作。

​	元数据锁MDL，在读写表的时候会自动加上，是为了防止线程在读写表，其他的线程去修改表的结构。MYSQL5.5之后引入了MDL，当线程对表进行增删改查的操作时会加上MDL读锁，读读之间时不互斥的，当修改表结构的时候会加上MDL写锁，写锁和其他的锁都互斥，会阻塞其他的读、写线程。

​	对线上的数据库表修改表结构时要十分的小心，有这样一个场景，如果sessionA开启了一个事务，然后对查询表，这时会加上一个MDL读锁，这时候只加了读锁，sessionB的读操作还是能够进行，此时来了sessionC想要修改表结构，尝试加MDL写锁，这时被sessionA的MDL读锁阻塞，后续的查询请求尝试加读锁都会被写锁阻塞，一般客户端都有超时的重试机制，很快就会把数据库的线程占满，从而导致整个数据库都无法使用了。

​	如何解决给表加字段的问题，可以查询information_schema中的innodb_trx表查看是否有长事务，如果有可以把长事务kill掉，或者取消DDL操作。但是如果变更的是热点表，kill就未必管用，因为即使kill了，下一个请求也会很快到来。这时候就要给DDL操作加上超时时间。

命令语法：

ALTER TABLE tbl_name NOWAIT add column ...

ALTER TABLE tbl_name WAIT N add column ...

### 7.3行锁

**行锁的两阶段锁**，在innodb的事务中，行锁在需要的才会加上而不是事务的开始，行锁的释放是在事务的提交后。知道这个设定后对业务有什么帮助？

一个事务中并发冲突程度越大的语句越是应该往后放，从而减少行锁的持有时间，因此来提高业务的吞吐量。

但是这样操作无法解决所有的问题，如果一个事务中持有多把行锁，可能会和其他的事务进入循环等待陷入死锁。

如何解决死锁？

- 可以给加锁设置一个超时时间，通过参数innodb_lock_wait_timeout来设置，默认是50s
- 死锁检测，发现死锁检测后回滚造成死锁链的一个事务，将参数innodb_deadlock_detect设置为on，表示开启这个逻辑。

但是在业务中50s的超时等待是不可以接受的，但是如果设置太短，比如一秒，又会造成不必要的回滚，因为有可能是正常的锁等待。

所以一般都是使用第二种方案。

但是死锁的检测不是没有消耗，如果事务进入了等待，就要看看它所依赖的线程有没有被别人锁住，如此循环，最后判断是否出现了循环等待，也就是死锁。这个是一个O(n)级别的操作，假设有1000个线程同时要更新一行数据，这个检测操作就是100万级别的。是一个非常消耗CPU的操作。所以会有CUP利用率100%，但是每秒只能执行不到几个事务的情况。

那么如何解决这种热点更新的情况？

- 如果确定业务不会出现死锁可以临时关闭死锁检测，但是这种操作本身带有一定的风险，因为业务设计的时候一般不会把死锁当做一个严重错误，毕竟出现死锁了，就回滚，然后通过业务重试一般就没问题了，这是业务无损的。而关掉死锁检测意味着可能会出现大量的超时，这是业务有损的。
- 另一种思路就是控制并发度，比如让同一行最多只有10个线程在更新，这样死锁检测的成本就很低，一个直接的想法就是，在客户端做并发控制。但是，你会很快发现这个方法不太可行，因为客户端很多。我见过一个应用，有600个客户端，这样即使每个客户端控制到只有5个并发线程，汇总到数据库服务端以后，峰值并发数也可能要达到3000。
- 所以最优的解决方案就是使用中间件，让要修改同一行的线程排队控制并发度，如果公司有DBA也可以让DBA修改MYSQL源码让排队做在引擎层。或者是用分治的思想，比如把一行库存拆成10个，每行占据1/10的库存，这样并发度分摊到10行中去。

## 8.mysq是如何实现事务之间数据的隔离性的以及如何回滚数据的?

### 	8.1**事务的隔离性**

依靠的是read view，在可重复读的隔离级别下，每次事务都会生成一个read view，read view中会记录当前活跃的事务id，还有其他中的最小值，和下一次要分配的事务id（事务id是数据库实例全局自增的）。

​	如何通过read view判断当时事务对其他事务的数据是否可见？

**情况一：select语句**

​	创建事务时会记录一个事务活动id列表，从最小的活跃事务id到下一次要分配的事务id，即分配给当前事务的id。最小的活跃事务id

为低水位，当前事务id为高水位。如果小于低水位则是已经提交的是事务，可见。当前事务id为高水位，如果大于高水位则是未来的事务，不可见。所以，反过来说，在【低水位，高水位】之间的事务都是未提交的，对当前事务是不可见的。

**情况二：update语句**

update的情况不同，update的操作实际分为两部，先读在写，update的读是当前读，永远是读最新的数据版本。举个例子，

假设当前数据库只有一个活跃事务id 90，k初始为1，sessionA为手动提交事务，with consistent snapshot是为了一开始就开启事务，默认情况下是在执行第一条语句时开启事务，sessionB为自动提交事务

| 时刻 | sessionA                                   |            sessionB            |
| :--: | ------------------------------------------ | :----------------------------: |
|  1   | start transaction with consistent snapshot |                                |
|  2   |                                            | update t set k=k+1 where id=1; |
|  3   | update t set k=k+1 where id=1;             |                                |
|  4   | select k from t where id=1；               |                                |
|  5   | commit                                     |                                |

时刻1 sessionA开启了事务，分配事务id为99，那么此时sessionA的活跃事务id列表应该为【90,99】，到了时刻二sessionnB开启事务分配事务id为100，sessionB的活跃事务列表为【90,99,100】，自动提交完成后此时k=2, 到时刻3, sessionA尝试将k+1，按道理来说sessionA只能看到k=1而看不到sessionB中的k=2，但是update永远是当前读，获取到最新的read view，所以时刻4看到的k应该等于3。

### 8.2**事务的回滚**

依靠是undo log，每执行一条DDL语句就会产生一条undo log来记录当前的事务id、数据内容、执行更早一条的undo log指针，当发生回滚时会按照后进先出的顺序来恢复数据。

InnoDB 的 **Purge 线程** 会在满足以下 **两个条件** 时删除 Undo Log：

1. **没有活跃事务依赖该 Undo Log**
   - 即该 Undo Log 对应的事务已提交，并且：
     - 没有活跃的 Read View 需要访问该 Undo Log（用于 MVCC）
     - 没有其他事务的 `roll_ptr` 指向该 Undo Log（即没有更晚的事务基于此版本修改数据）
2. **该 Undo Log 所在的事务的提交顺序早于系统当前最老的活跃事务**
   - InnoDB 会维护一个 **`history list`**，按事务提交顺序链接 Undo Log。
   - Purge 线程只能清理 **已提交且位于 `history list` 最前端** 的 Undo Log（即最早提交的事务产生的 Undo Log）。



## 9.普通索引和唯一索引怎么选？

### 9.1 查询

假设，执行查询的语句是 select id fromT where k=5。这个查询语句在索引树上查找的过程，先是通过B+树从树根开始，按层搜索到叶子节点，也就是图中右下角的这个数据页，然后可以认为数据页内部通过二分法来定位记录。

- 对于普通索引来说，查找到满足条件的第一个记录(5,500)后，需要查找下一个记录，直到碰到第一个不满足k=5条件的记录。
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。

两者之间的差距是微乎其微的，因为mysql读取数据是按页读取数据的，一个页默认为16kB大概可以存放几千个key，只有当查询的数据是页上的最后一个数据时，普通索引才会多做一次磁盘IO，概率是很小的，平均的时间复杂度几乎相同。

### 9.2 修改

要说明普通索引和唯一索引之间对插入操作的影响，要先说明一下change buffer，当一个数据要更新时，如果数据在内存中，那么直接更新，如果不在内存中，则把修改操作写入change buffer，等到下次查询该数据时再应用change buffer中的操作修改数据，这个过程称为merge。使用change buffer可以显著的减少IO的次数，不过对于唯一索引来说，change buffer是不起作用的，因为唯一索引需要做唯一的检测索引每次都要把整个数据页载入内存。

## 10.为什么MYSQL有时候会选错索引?

### 10.1 选择索引的影响因素

因为影响索引选择的因素有很多，包括扫描行数、是否需要回表、是否需要进行排序、是否需要创建临时表等等，有时候可能选择索引a需要扫描的行数更少，但是结果需要根据字段b来进行排序，那么就有可能选择索引b来避免对结果进行排序。或者使用索引要回表的时间查找的时间，有可能也会使用全表扫描。

### 10.2 如何判断优化器是否选择了最优的执行计划

优化器选择的执行计划也不是任何时候都是最优的，如何判断优化器是否选择了最优的执行计划，可以用explain看一下优化器的预估执行计划是否符合预期，如果不符合，可以把慢sql时间设成0，然后一条使用force index和不使用force index的语句做一个对照，然后在慢日志中对比一下哪个执行时间最短。

### 10.3 如何使用最优的执行计划

#### 10.3.1 analyze table

如果优化器确实没有选择最佳的索引，可以使用analyze table t命令来重新统计一下索引信息，表的统计信息是通过抽样的方式进行统计的，采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过1/M的时候，会自动触发重新做一次索引统计。

在MySQL中，有两种存储索引统计的方式，可以通过设置参数innodb_stats_persistent的值来选择：

- 设置为on的时候，表示统计信息会持久化存储。这时，默认的N是20，M是10。
- 设置为off的时候，表示统计信息只存储在内存中。这时，默认的N是8，M是16。

#### 10.3.2 诱导优化器

假设有一条查询语句select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1；

这条语句会因为需要对b排序而不使用只需要扫描1000行的索引a，而是使用要扫描50000行的索引b，但是可以通过修改语句为

select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b**,a** limit 1；

让索引选择a，间接的达到目的。

## 11.字符串的索引如何选取？

#### 11.1 选择完整的字符串

**优点：**查询时是区分度高，而且不用回表

**缺点：**索引占用的空间会变大，会增加IO的次数

#### 11.2 前缀索引

**优点：**占用的空间小，IO次数少

**缺点：**需要回表查出完整的字符串做对比。

**索引的长度如何选择？**

根据区分度来选择，区分度越高需要回表的次数也会越少。

例子：

select count(distinct left(email,4)）as L4, count(distinct left(email,5)）as L5, count(distinct left(email,6)）as L6,

count(distinct left(email,7)）as L7,

from SUser;

#### 11.3倒序存储

当遇到前缀的区分度不高的时，比如一个市的居民库，身份证的前几位代表的是城市，如果选择前缀索引的话区分度会很低，可以把身份证号倒序存储，这样就可以提高区分度，缺点是每次读写时都要加一层reverse函数。

#### 11.4hash存储

当字符串前缀区分度不高时，可以加一列CRC32字段，在hash列上添加索引只需要4个字节，而且冲突的概率很小，只有在冲突时才需要对比整个字符串。

## 12. 为什么MYSQL会突然抖动一下？

在使用MYSQL数据库时经常会碰到一个问题就是为什么突然一个查询会变得的很慢，但是是随机出现的。这是因为MYSQL在查询的过程中触发了脏页置换。那时候会发生脏页置换呢？

### **场景一：**redo log满了

redo log满了，redo log本质上是一个循环链表，当write pos追上check pos表示没有写的空间了，如果这时候需要进行写操作，则要把check pos的位置往前移动，就是要把一部分的redo log记录的脏页写入磁盘。这种情况下系统无法再接收任何的写操作，所有的写操作都会被阻塞。TPS会直接降成0。

### **场景二：内存不够用了，最久未被使用的数据页需要被置换出去。**

内存不够用了，最久未被使用的数据页需要被置换出去。刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

- 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
- 日志写满，更新全部堵住，写性能跌为0，这种情况对敏感业务来说，是不能接受的。

所以要通过控制刷脏页的速度来防止这种抖动情况。

刷脏页的速度由两个因素来控制：

- 脏页的比例
- redo log的写盘速度

MYSQL会根据这两个两个因素分别算出两个数字，取其中最大的然后乘以innodb_io_capacity，innodb_io_capacity是IO读写能力，可以通过fio来得出。其实，因为没能正确地设置innodb_io_capacity参数，而导致的性能问题也比比皆是。MySQL的写入速度很慢，TPS很低，但是数据库主机的IO压力并不大。经过一番排查，发现罪魁祸首就是这个参数的设置出了问题。他的主机磁盘用的是SSD，但是innodb_io_capacity的值设置的是300。于是，InnoDB认为这个系统的能力就这么差，所以刷脏页刷得特别慢，甚至比脏页生成的速度还慢，这样就造成了脏页累积，影响了查询和更新性能。

## 13. count（*）很慢如何优化?

### 13.1 count(*)的实现方式：

- **MYISAM：**有个计数器持久化在磁盘中。
- **INNODB：**遍历行做统计。

**为什么两者的实现方式不同？**

这是因为即使是在同一个时刻的多个查询，由于多版本并发控制（MVCC）的原因，InnoDB表“应该返回多少行”也是不确定的。而MYSIAM是不支持事务的。

**进行count(*)的时候索引如何选择**

InnoDB是索引组织表，主键索引树的叶子节点是数据，而普通索引树的叶子节点是主键值。所以，普通索引树比主键索引树小很多。遍历不同的索引树只要逻辑正确就会选择最小的那一个，保证逻辑正确的前提下，尽可能的少扫描行数。

### 13.2 如何进行优化？

**redis**：将计数器存放在redis中， 但是redis和mysql是两个服务不支持事务，假设有这样一个查询请求，查询操作记录的总数还有最近的100条操作记录，那么，这个逻辑就需要先到Redis里面取出计数，再到数据表里面取数据记录。

我们是这么定义不精确的：

1. 一种是，查到的100行结果里面有最新插入记录，而Redis的计数里还没加1；

2. 另一种是，查到的100行结果里没有最新插入的记录，而Redis的计数里已经加了1。

**存储在mysql中：**可以放在单独的一张计数器表中。首先解决的崩溃重启的问题，因为INNODB是支持crash-safe的。第二INNODB支持事务，在可重复读的隔离级别下是不会出现计数器和记录数不一致的情况的。

### 13.3 不同的count()之间的区别

这里，首先你要弄清楚count()的语义。count()是一个聚合函数，对于返回的结果集，一行行地判断，如果count函数的参数不是NULL，累计值就加1，否则不加。最后返回累计值。所以，count(*)、count(主键id)和count(1) 都表示返回满足条件的结果集的总行数；而count(字段），则表示返回满足条件的数据行里面，参数“字段”不为NULL的总个数。

**count(字段)：**

1. 如果这个“字段”是定义为not null的话，一行行地从记录里面读出这个字段，判断不能为null，按行累加；

2. 如果这个“字段”定义允许为null，那么执行的时候，判断到有可能是null，还要把值取出来再判断一下，不是null才累加。

**count(主键id)**：InnoDB引擎会遍历整张表，把每一行的id值都取出来，返回给server层。server层拿到id后，判断是不可能为空的，就按行累加。

**count(1)：**InnoDB引擎遍历整张表，但不取值。server层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。

**count(*)**: mysql的优化器做过专门的优化

按照效率排序的话，count(字段)<count(主键id)<count(1)≈count(*)，所以我建议你，尽量使用count(*)。

## 14.如何查询随机消息？

**场景**：英语学习App首页有一个随机显示单词的功能，也就是根据每个用户的级别有一个单词表，然后这个用户每次访问首页的时候，都会随机滚动显示三个单词。他们发现随着单词表变大，选单词这个逻辑变得越来越慢，甚至影响到了首页的打开速度。

**sql：**select word from words order by rand() limit 3；

**问题：**使用explain发现Extra字段显示 Using temporary，表示使用临时表；use filesort，表示是需要执行排序操作。

**优化**：

方法一：查出最大max（id）和min（id），取最大值和最小值之间的一个随机数，公式：x = (M-N)*rand() + N，

select word from words where id > x limit 1；但是如果id之间有空洞，比如1，2,40001,40002，那么每次去平均数只会取到40001，甚至可以算bug了。

方法二：统计单词个数，C = count（*），Y = C * rand(),  select word from words limit Y，1; 总共要扫描 C+Y+1行。如果Y随机的很大的话，就接近于2C，不过还是会比方法一快，因为扫描时是根据rowid的索引，不用创建临时表和排序。随机3个单词，即就是

select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行

select * from t limit @Y2，1；

select * from t limit @Y3，1

不过可以进一步优化，后续查询语句根据前一条查出的id，加一个条件id>x,避免做重复的扫描。

**监控是否使用磁盘临时表**：

SET optimizer_trace='enabled=on'；

select word from words order by rand() limit 3;

SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

## 15.为什么有的sql语句逻辑相同但是效率却大不相同？

**案例一：查询条件加函数操作导致索引失效**

```
//建表语句
mysql> CREATE TABLE `tradelog` (
`id` int(11) NOT NULL,
`tradeid` varchar(32) DEFAULT NULL,
`operator` int(11) DEFAULT NULL,
`t_modified` datetime DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `tradeid` (`tradeid`),
KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

```
select * from tradelog where month(t_monified) = 7;
```

**问题**：索引失效了，走的全表扫描

**原因**：加了month函数后，传入的查询值为7,而索引是根据全日期来建立索引的，所以索引失效走的全表扫描。

**解决办法**：

```
mysql> select count(*) from tradelog where
-> (t_modified >= '2016-7-1' and t_modified<'2016-8-1') or
-> (t_modified >= '2017-7-1' and t_modified<'2017-8-1') or
-> (t_modified >= '2018-7-1' and t_modified<'2018-8-1')
```

**案例二：隐式类型转换**

```
mysql> select * from tradelog where tradeid=110717;
```

**问题:** 其中tradeid是字符串类型，而给出的查询字段是int类型，所以先做类型转换后再做比较，字符串类型和整型进行比较，字符串类型会被先转换为整型，即CAST(tradeid as signed  int)，触发了对索引字段的函数操作，所以优化器会放弃使用索引。

**案例三：隐式字符串编码转换**

```
//新增一张交易细节表
mysql> CREATE TABLE `trade_detail` (
`id` int(11) NOT NULL,
`tradeid` varchar(32) DEFAULT NULL,
`trade_step` int(11) DEFAULT NULL, /*操作步骤*/
`step_info` varchar(32) DEFAULT NULL, /*步骤信息*/
PRIMARY KEY (`id`),
KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
insert into tradelog values(1, 'aaaaaaaa', 1000, now());
insert into tradelog values(2, 'aaaaaaab', 1000, now());
insert into tradelog values(3, 'aaaaaaac', 1000, now());
insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```

```
//查询语句
select d.* from trade_log l, trade_detail where d.tradeid = l.tradeid and l.id = 2
```

**问题**：查询语句先去tradelog中根据id=2查询tradeid，然后再根据tradeid查询trade_detail，当查询trade_detail时索引失效。

**原因**：因为两张表使用的字符集不相同，这时候tradelog为驱动表，trade_detail为被驱动表，当需要比较时，不同字符集需要转成相同的字符集后才能进行比较，而tradelog表使用的字符集为utf8mb4是trade_detail使用的utf8的超集，于是需要把trade_detail中的tradeid字段转换为utfmb4，相当于使用函数CONVERT(tradeid USING utfmb8)，在索引列上使用函数导致，索引失效，被驱动表走全表扫描。

**优化手段**：

方法一：把trade_detail字符集升级成utfmb8，不过是DDL操作，如果数据量比较大的话，会影响业务。

方法二：手动地把l.tradeid转换为utf8的字符集，即CONVERT(l.tradeid USING utf8)。



