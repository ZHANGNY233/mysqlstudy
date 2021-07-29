# Mysql事务相关



## 1.1 事务的基本要素（ACID）

### 1.1.1 原子性（Atomicity）

事务开始之后的所有操作，要么全部做完，要么全部不做，不可能停滞在中间环节。事务执行过程中如果出错，会回滚到事务开始之前的状态，所有的操作就像没有发生过一样。即事务是一个不可分割的工作单位，类似于化学中的原子概念，是物质构成的基本单位，不可再分。只有事务中的所有数据库操作都执行成功，才算整个事务成功。事务中任何一个SQL语句执行失败，已经执行成功的SQL语句也必须撤销，回到执行事务之前的状态。

### 1.1.2 一致性（Consistency）

事务会使数据库从一个一致性状态变换成另一个一致性状态。数据库的完整性约束不会被破坏。数据的完整性保持一致，即不会存在A向B转账，A扣款，B却还未收到的状态，这样就违反了一致性。存在的状态，只可能是两种：

A还未扣款，此时B还未收到； A已扣款，B已经收到。这两种状态都保持了数据的完整性/一致性。

### 1.1.3 隔离性（Isolation）

事务的隔离性是多个用户并发访问数据库时，数据库为每个用户开启的事务，不能被其他用户的操作所干扰，多个并发事务之间要相互隔离。

### 1.1.4 持久性（Durability）

事务一旦被提交，它对数据库中的数据改变就是永久性的，不可以再回退。

## 1.2 Mysql对ACID特性的实现

### A Mysql 如何实现原子性？

#### A.1 undo log（回滚日志）

**回滚日志是mysql实现原子性的关键，undo log 可以保证在发生回滚的时候，可以撤销所有已经执行成功的SQL。**

undo log属于**逻辑日志**。逻辑日志指的是，只是将数据库逻辑地恢复到原来的样子，并不能将数据库物理地恢复等到执行语句或者事务之前的样子。虽然所有的逻辑修改都被取消了，但是数据结构和页本身在回滚前后可能不一样了。

当事务对数据库进行修改的时候， InnoDB 会生成与之对应的 undo log，如果事务执行失败或者调用 rollback，导致事务需要回滚， InnoDB引擎会根据回滚日志中的记录，把数据回滚到之前的样子。在undo log中存储的是 SQL 语句执行相关的信息，事务中使用的每一条 INSERT 都对应了一条 DELETE，每一条 UPDATE 也都对应了一条还原旧值的 UPDATE 语句。如下图：

![image-20210727210938246](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727210938246.png)





当insert一条数据的时候，记录这条记录的主键，回滚的时候，根据主键执行delete操作。

在update语句执行undo log回滚时有可能会涉及到MVCC。这主要是为了保证在执行undo log的时候可以拿到正确版本。

MVCC 可以通过 undo log 形成一个事务执行过程中的版本链，每一个写操作会产生一个版本，数据库发生读的并发访问时，读操作访问版本链，返回最合适的结果直接返回。从而保证读写操作之间没有冲突。







### C Mysql 如何实现一致性？





### I  Mysql 如何实现隔离性？

锁机制

MVCC

对于隔离性，分两种情况讨论：

——一个事务中的写操作对另一个事务中的写操作的影响

——一个事务中的写操作对另一个事务中的读操作的影响

事务之间的写操作靠Mysql的锁机制来实现隔离，而事务间的写和读操作是靠MVCC机制来实现隔离的。

锁机制

快照读和当前读。





### D Mysql如何实现持久性？

#### D.1 redo log的产生背景

**简单来说Mysql为了解决磁盘 I/O 效率低下的问题，引入了缓冲池，又为了解决缓冲池带来的一致性问题，引入了 redo log ( 重做日志 ).**

##### D.1.1 缓冲池引入

Mysql的数据最终是存放在磁盘中，所以磁盘的容量大小决定数据库的容量大小。但如果对Mysql的操作都是通过读写磁盘来进行的话，磁盘的I/O会把效率大大拉低。所以 InnoDB为 Mysql提供的缓冲池（Buffer Pool），在读写数据时，把相应的数据和索引载入到内存中的缓冲池中，一定程度提高了数据读写的速度。

缓冲池占最大块内存，用来存放各种数据的缓存，包括有索引页、数据页、undo页、插入缓冲、自适应哈希索引、innodb存储的锁信息、数据字典信息等。工作方式是每次将数据库文件按页（每页16kb）读取到缓冲池，然后按照 LRU算法（最近最少使用）来保留在缓冲池中的数据。当从数据库读取数据的时候，先从Buffer Pool中读取数据，如果Buffer Pool中没有，则从磁盘读取后放入到Buffer Pool中。当向数据库写入数据的时候，会先写到Buffer Pool中，发生修改后的页即为dirty page( 脏页 )，然后再按照一定的频率把缓冲池的脏页刷新到磁盘中。（刷脏）

Buffer Pool可以提高 Mysql的读写效率，但是也存在问题。如果数据被更新到 Buffer Pool却还没 来得及刷新到磁盘中，这个时候突然宕机或者掉电，那么数据会丢失，无法保证事务的持久性。

##### D.1.2 redo log 引入

为了解决缓存带来的一致性问题，Mysql 引入了 redo log。当 Mysql 对缓存池的数据进行修改时，通过redo log记录这次操作，事务提交的时候通过 fsync 接口把 redo log进行刷盘。（即数据不是即时刷盘，但是对数据的操作会在是事务提交时刷盘）因为在事务提交时，redo log会被同步更新到磁盘，如若此时出现宕机或者掉电，可以在磁盘中读取redo log进行数据的恢复，这个时候就保证了事务的持久性。

redo log 采用 Write-Ahead Logging (日志预写)的方式，即在事务提交的时候先记录日志，再更新Buffer Pool，这样可以强行保证在redo log中更新了数据的情况下，这些数据最终都会被刷新和存储到磁盘中了。



#### D.2 redo log 工作方式

##### D.2.1 更新

redo log 分为两部分，一部分是内存中的 重做日志缓存( redo log buffer )，这部分容易丢失，二是磁盘上的重做日志文件( redo log file )，这部分是持久的。

**redo log 是如何更新的？**以 update操作为例子

![image-20210727211831331](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727211831331.png)

执行update操作。

( 若内存中没有 )先将原始数据从磁盘读取到内存，修改内存中的数据。

生成一条重做日志写入 redo log buffer，记录数据被修改后的值。

当事务提交时，需要先将 redo log buffer中的内容刷新到 redo log file。

事务提交后，也会将内存中修改数据的值写入磁盘。



对于第④步 内存中的 redo log buffer 刷新到硬盘上的 redo log file 这一步，更详细的过程如下：

![image-20210729022459414](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210729022459414.png)



InnoDB 通过 force log at commit 技术来实现事务的持久化特性。为了保证每次 redo log 都能写入磁盘上的日志文件中，每次将内存中的 redo log buffer 内容同步磁盘时都会调用一次 fsync。对操作系统来说 write 和 fsync 是不同的操作，通常情况下 write 只是到了磁盘 IO 缓冲区，何时同步到磁盘由操作系统控制。

**为了确保每次日志都写入重做日志文件，在事务提交时，InnoDB存储引擎会强制调用一次 fsync 操作。**



**问题1：redo log 刷新到磁盘 和 数据刷脏都是写磁盘，为什么要用redo log而不是直接刷脏？**



##### D.2.2 I/O 性能

为了保证 redo log 可以有较好的 I/O 性能，InnoDB的 redo log 的设计有以下几个特点：

1.尽量保持 redo log 存储在一段连续的空间上。在系统第一次启动的时候就把日志文件的空间完全分配，此后以顺序追加的方式记录 redo log。

2.批量写入日志。日志并不是直接写入文件，而是先写入 redo log buffer，然后以特定的频率或者在被强制调用同步时将 buffer中的数据一并写入磁盘。

3.并发的事务共享 redo log 的存储空间，他们的 redo log 按语句的执行顺序，依次交替地记录在一起，以减少日志占用的空间。

4.redo log 上只进行顺序追加的操作，当一个事务需要回滚的时候，它的 redo log 记录也不会被删掉或者撤销。



**解答1：主要原因是redo log的更新比刷脏快很多。**

——redo log是追加操作日志，是**顺序I/O**； 而刷脏是**随机I/O**，显然顺序I/O效率高很多。

——刷脏是**以数据页为单位**的，即每次最少从磁盘中读取一页数据到内存，或者最少刷一页数据到磁盘。Mysql的默认页大小是16KB，对于一页的修改，整个页都要刷到磁盘中；而redo log只包含真正需要写入磁盘的操作日志。



##### D.2.3 存储格式和内容

redo log 和 binlog二进制日志的不同，binlog主要是主从复制和进行 Point in time的恢复。



Mysql还有一个记录操作的日志 binlog. redo log 和 binlog的区别如下：

作用——redo log 是用来记录更新缓存的，为了保证Mysql就算宕机也不会影响事务的持久性；binlog是用来记录什么时间操作了什么，有时间点，可以保证将数据恢复到某个时间点，或者可以用于主从同步数据。

层次——redo log是存储引擎 InnoDB实现的，（MyISAM没有redo log），而binlog是在Mysql服务器层面存在的，任何其他存储引擎也有binlog。redo log是物理日志，基于磁盘的数据页，binlog是逻辑日志，存储一条执行的SQL的信息。

时机——redo log在默认情况下是在事务提交时进行刷盘的；可以通过参数：innodb_flush_log_at_trx_commit 来改变策略，可以不用等到事务提交时才进行刷盘。如：可以设置为每秒刷新一次。

binlog是在事务提交时写入。



二者写入方式对比如下:

![image-20210728194636607](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210728194636607.png)



可以看出 binlog 是每次事务提交写入，所以每个事务仅会有一条日志。逻辑日志，记录的是逻辑上的修改，即SQL语句层面。

而 redo log 则是在事务一开始就写入，*T1 表示的是事务提交。 物理日志，即它记录的是物理页上的修改。

redo log 默认是以 block（块）的方式为单位进行存储，每个块是512字节。由于redo log 是和数据库引擎有关，它的格式由数据库引擎来决定，InnoDB的存储管理是基于页的，redo log 也是。它的格式如下：

![image-20210728195241615](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210728195241615.png)

四个字段：

redo_log_type 重做日志类型

space 表空间的ID

page_no 页的偏移量

redo_log_body存储内容

执行一条插入语句，它的redo log 可能为：



```sql
INSERT INTO user SELECT 1,2;
           |
page(2,3), offset 32, value 1,2 # 主键索引
page(2,4), offset 64, value 2   # 辅助索引
```

##### D.2.4 恢复机制

![image-20210728200422289](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210728200422289.png)



LSN( Log Sequence Number )日志序列号，InnoDB里，LSN单调递增，代表的含义有：重做日志写入的总量、checkpoint的位置、页的版本。

假设在 LSN = 10000 的时候数据库出现故障，磁盘中checkpoint 为10000，表示磁盘已经刷新数据到10000这个序列号，而当前 redo log的 checkpoint 是13000，则代表需要恢复10000-130000序号中的数据。





undo log 和 redo log 本身是分开的。Innodb引擎的 undo log 是记录在数据文件中的，而且 innodb 是将 undo log 的内容看作是数据，因此对 undo log 本身的操作（比如向undo log中插入一条新的undo记录 等），也会记录在 redo log中，undo log 可以不必立即刷新到磁盘上，即使丢失了，也可以通过redo log将其恢复。当在数据库插入一条记录的时候：

先向 undo log 中插入一条 undo log记录（对应的删除记录部分）

再向 redo log 中插入一条“ 插入undo log记录 ” 的 redo log 记录。

插入数据。

最后向 redo log 中插入一条 “ insert ”相关的 redo log 记录。





<u>只靠 undo log 行么？</u>
<u>只有 undo log 的情况下就必须保证事务提交前，脏页必须刷回磁盘，否则宕机时这部分数据修改就在内存中丢失了，破坏了持久性，但是如上文提到的，这种办法性能太差，必须改进。</u>

<u>只靠 redo log 行么？</u>
<u>只有 redo log 的情况下就不能在事务提交前刷脏，我们假设有个大事务更新数据量很大，更新完一部分必须刷一部分脏页回磁盘然后再 load 其他的数据进内存操作，此时如果更新到一半宕机了，那么数据库里存在一部分新数据一部分旧数据，而事务又没有提交，因此应该回滚，只有 redo log 的情况下无法完成。</u>















## 2.1 事务的四种隔离级别

### Read Uncommitted (读未提交)

事务可以看到其他未提交事务的执行结果。这种读取未提交的数据的行为，也被称为脏读（Dirty Read）。该级别使用很少。

### Read Committed (读已提交)

这是大多数数据库系统使用的默认隔离级别。（但不是mysql默认的，可以查查资料举例）一个事务只能看见已提交事务所做的改变，即事务提交之前对其余事务不可见。



### Repeatable Read (可重复读)

### Serializable（串行化）

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 读已提交（read-committed）   | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |







mysql默认隔离级别为可重复读，mysql的可重复读是如何实现的？是否真的可以防止幻读？需要代码验证一下。

Mysql 可重复读隔离级别之所以可以防止幻读现象出现，是因为可重复读这个隔离级别使用了GAP锁（间隙锁)。

https://www.cnblogs.com/laipimei/p/10483677.html





