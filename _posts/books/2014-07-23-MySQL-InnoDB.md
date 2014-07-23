---
layout: post
category: book
titile: 《MySQL技术内幕：InnoDB存储引擎》笔记
---

##Summary
这本书没有讲的太深，在掌握了MySQL的基本操作，对InnoDB有一个基础的了解之后，如果想对InnoDB原理性的一些东西了解的更多，从这本书入手我觉得是可以的。作者对这本书定位要高一些，不过目前我所吸收的东西给我的感觉是像前面说的这样。

---

##MySQL体系结构和存储引擎
数据库是文件的集合，是依照某种数据模型组织起来并存放于二级存储器中的数据集合；
数据库实例是程序，是位于用户和操作系统之间的一层数据管理软件，用户对数据库数据的任何操作，包括数据库定义、数据查询、数据维护数据库运行控制等都是在数据库实例下进行的，应用程序只有通过数据库实例才能和数据库打交道。

MySQL体系结构图：
![](http://francisnote.qiniudn.com/mysql-server-architecture.jpg)

MySQL由以下几个部分组成：

 - 连接池管理 Connectors
 - 管理服务和工具组件 Management Service & Utillties
 - SQL接口组件 SQL Interface
 - 查询分析器组件 Parser
 - 优化器组件 Optimizer
 - 缓冲组件 Cathes & Buffers
 - 插件式存储引擎 Pluggable Storage Engines
 - 物理文件 File System
具体各个部分的功能请参照[这里](http://www.searchdatabase.com.cn/showcontent_57918.htm)

MySQL区别于其他数据库很重要的一点就是其插件式的表存储引擎，MySQL的存储引擎是基于表的，而不是基于数据库的，也就是说同一个数据库里不同的表可以使用不同的存储引擎。


InnoDB存储引擎支持事务，其特点是行锁设计、支持外键、支持类似于Oracle的非锁点读，通过MVCC来获得高并发性，并且实现了SQL标准的四种隔离级别，默认为REPEATABLE，同时使用一种被称为next-key locking的策略来避免幻读。此外还提供插入缓冲、二次写、自适应哈希、预读等功能。

MyISAM不支持事务，表锁设计，支持全文索引，它的缓冲池只缓冲索引文件不缓冲数据文件。

Memory，数据放在内存中，MySQL数据库使用Memory存储引擎作为临时表来存放查询的中间结果集。

Archive，只支持INSERT和SELECT操作，非常适合存储归档信息，其设计目标主要是提供高速的插入和压缩功能。

##InnoDB存储引擎

总体架构如下：
![](http://images.51cto.com/files/uploadimg/20101210/122648495.jpg)

通过`SHOW ENGINE INNODB STATUS;`可查看innodb的状态。

后台线程主要有：

 - Master Thread
 主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性。
 - IO Thread
 用于处理write, read, insert buffer和log IO thread，读写的线程数可分别通过innodb\_read\_io\_threads和 innodb\_write\_io\_threads进行设置。
 - Purge Thread
 用于回收已经分配并使用的undo页。可在mysql配置文件中设置该类线程的数量
   innodb\_purge\_threads = 1;
 - Page Cleaner Thread
 用于刷新脏页

内存：

 - 缓冲池
 缓存的数据有：索引页，数据页，undo页，插入缓冲，自适应哈希索引，InnoDB存储的锁信息，数据字典等。buffer pool可以有多个，可在配置文件中配置innodb\_buffer\_pool\_instances。从5.6开始也可通过information\_schema架构下的INNODB\_BUFFER\_POOL\_STATS来查看缓冲池的状态。
 - LRU list, Free list和Flush list
 InnoDB对内存中数据的管理是通过LRU（最近最少使用）算法实现的，但是稍作修改，新访问的页并不是直接放到LRU list的首部，而是midpoint（innodb\_old\_blocks\_pct）处，这样可以避免将热点数据直接刷新出队列（比如数据扫描这样的操作，得到的结果比较大，而且操作频率不高）；同时引入innodb\_old\_blocks\_time，表示页被读取到midpoint位置后需要多久会被加入到LRU list的热端。

数据库刚启动时，LRU为空，所有的页都在Free List中。

LRU list中的页被修改之后称该页为脏页，数据库会通过CHECKPOINT机制刷新脏页回磁盘。脏页即存在与LRU列表中，也存在于Flush列表中，LRU list用来管理缓冲池中页的可用性，Flush list用于管理将页刷新回磁盘，二者互不影响。
 - 重做日志缓冲
 innodb\_log\_buffer\_size，默认8M，不需要太大，每秒都会将重做日志缓冲刷新到文件。
 - 额外的内存池
 存储锁，等待等信息 （不是很明白这个区域到底是做什么用的，不过innodb\_buffer\_pool增大的时候，这部分区域也需要跟着增大。）

###CheckPoint技术
主要是用于将脏页里面的数据刷新到磁盘上；在事务提交之前所有的操作都被记录到了redo log中，当数据库宕机之后，只需要将checkpoint之后的重做日志做处理，之前的页已经被刷新到磁盘上面了。对于InnoDB而言，是通过LSN(Log Sequence Number)来标记版本的。每个页，重做日志和checkpoint都有自己的LSN。
 
 > Log sequence number 74982049365
   Log flushed up to   74982049365
   Pages flushed up to 74982042549
   Last checkpoint at  74982042549
   
InnoDB中checkpoint有两种：

 - Sharp Checkpoint，发生在数据库关闭时将所有脏页刷新回磁盘。
 - Fuzzy Checkpoint，刷新部分脏页，分如下几种情况：
  - Master Thread Checkpoint, master thread以每秒和每10秒的速度将部分脏页刷新回磁盘
  - FLUSH\_LRU\_LIST Checkpoint，从LRU尾端移除的页，如果含有脏页那么进行Checkpoint
  - Async\Sync Flush Checkpoint，重做日志不可用的时候，强制将一些页刷新回磁盘（不是很明白到底是怎么回事）
  - Dirty Page too much Checkpoint, 脏页太多，大于innodb\_max\_dirty\_pages\_pct
  

###Master Thread
master thread的主要工作可以用下面的为代码来表示:

InnoDB1.2.×版本之前：
```
void master_thread() {
    goto loop;
    
    loop:
        for(int i = 0; i < 10; i++) {
            thread_sleep(1); //休眠一秒钟，当数据库负载比较大的时候，并非总是休眠一秒钟。
            do log buffer flush to disk
            if (last_one_second_ios < 5% innodb_io_capacity) {
                do merge 5% innodb_io_capacity insert buffer
            }
            
            if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
                do buffer pool flush 100% innodb_io_capacity dirty page
            } else if (enable adaptive flush) {
                do buffer pool flush desired amount dirty page
            }
            
            if (no user activity) {
                goto background loop
            }
        }
        
        if (last_ten_seconds_ios < innodb_io_capacity) {
            //系统认为这会儿IO压力不大，然后刷新脏页
            do buffer pool flush 100% innodb_io_capacity dirty page
        }
        do merge 5% innodb_io_capacity insert buffer
        do log buffer flush do disk
        do full purge
        if (buf_get_modified_ratio_pct > 70%) {
            do buffer pool flush 100% innodb_io_capacity dirty page
        } else {
            do buffer pool flush 10% innodb_io_capacity dirty page
        }
        goto loop
        
    background loop:
        do full purge
        do merge 100% innodb_io_capacity insert buffer
        if not idle
            goto loop
        else 
            goto flush loop
        
    flush loop:
        do buffer pool flush 100% innodb_io_capacity dirty page
        if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct)
            goto flush loop
        else 
            goto suspend loop
    
    suspend loop:
        suspend_thread();
        waiting event
        goto loop
}
```

基本上master thread所做的事情就是：刷新日志缓冲，刷新脏页，合并插入缓冲，删除无用的undo页这几个操作，只不过不同情况下刷新的页数之类的不同。

`innodb_io_capacity`: 表示磁盘的IO吞吐量，可调节。
`innodb_max_dirty_pages_pct`: 默认为75，Google测试的最佳数值是80
`innodb_purge_batch_size`: 每次 full purge回收的undo页
 
 InnoDB1.2.×的：
 
```
 if InnoDB is idle 
    srv_master_do_idle_tasks() //之前版本10秒一次的操作
 else 
    srv_master_do_active_tasks() //之前每秒的操作
```

另外将脏页的刷新操作分离到了Page Cleaner Thread中。
 
###InnoDB的关键特性

 - 插入缓冲（insert buffer）
 主要是将多次插入操作合并，减少随机IO。（对非聚簇索引的插入，先判断插入的非聚簇索引是否在缓冲池中，若在则直接插入，若不在则先放到一个insert buffer对象中。） 要使用insert buffer需要满足两个条件：1、 索引是辅助索引； 2、索引不是唯一的。（如果索引唯一，或者索引是聚簇索引，那么需要先判断数据的唯一性，这样需要去索引页查询，引入了随机读的操作，insert buffer就失去了意义。）

Insert Buffer有个升级版，Change Buffer，支持insert、update和delete操作，分别对应Insert Buffer, Purge buffer, Delete buffer。通过设置`innodb_change_buffering`来开启各种buffer的选项（changes表示insert和delete，all表示所有，none表示不开启）。`innodb_change_buffer_max_size`设置change buffer最大使用的内存（该参数有效值不超过50）。

insert buffer的内部实现是一个B+树，非叶节点存放search key（该页所在表（这个表应该是文件机构里面那个表，而非数据库table）的空间ID和偏移量offset），叶子节点是要插入的记录。启用insert buffer之后，会用insert buffer bitmap来标记每个辅助索引页的可用空间。

在三种情况下会merge insert buffer： 1、辅助索引页被读取到缓冲池； 2、Insert buffer bitmap可用空间不足； 3、Master Thread merge insert buffer

 - 两次写（double write）
 参考[这里](http://franciszhao.github.io/article/2014/07/03/InnoDB-Doublewrite/)
 - 自适应哈希索引（adaptive hash index）
 InnoDB自优化策略，无需DBA人为调整。通过缓冲池的B+树页构建而来，因此建立速度非常快，无需对整张表构建哈希索引。只能用来处理等值查询。
 - 异步IO(Async IO)
 需要内核级别的异步IO支持，默认打开
 - 刷新邻接页（Flush Neighbor Page）
 刷新一个脏页时，检测该页所在区的其他所有页，如果是脏页则一起刷新。通过`innodb_flush_neighbors`来控制，如果磁盘是固态硬盘之类的有超高IOPS性能的，建议不开启此特性。

启动、关闭和恢复

`innodb_fast_shutdown`:影响InnoDB关闭时的状况

 - 0: 完成所有full purge、merge insert buffer以及刷新脏页，需要时间可能较长
 - 1： 默认值，只刷新脏页
 - 2： 只将日志写入日志文件，下次启动的时候会进行recovery操作

`innodb_force_recovery`:影响InnoDB恢复的状况

可设置为不进行事物的回滚操作啊，不进行插入缓冲的合并操作等，有些时候用户知道如何手动恢复自己的数据库，比如alter很大的一张表的时候数据库宕机，重新启动的时候数据库自己会进行回滚操作，但是比较耗时，这时用户可设置`innodb_force_recovery`不让数据库进行事务回滚，自行重新操作。

---

##文件

`show variables like ''`

`select | [global | session] system_var_name`

`select [@@global. | @@session.] system_var_name`

作用应该是一样的，都是用来查询数据库的参数。
   
设置动态参数可以用SET: `set | [global | session] system_var_name=expr | [@@global. | @@session. | @@] system_var_name=expr`

###日志文件

**错误日志**

`SHOW VARIABLES LIKE 'log_error'`

**慢查询日志**

`SHOW VARIABLES LIKE 'slow_query_log'`

`SHOW VARIABLES LIKE 'slow_query_log_file'`

`SHOW VARIABLES LIKE 'long_query_time'`

`SHOW VARIABLES LIKE 'log_queries_not_using_indexes'` log没有用到索引的query

慢查询日志默认不开启，long\_query\_time的单位是秒，一般来说也只是在对数据库性能进行分析的时候会开启，并不会一直把慢查询日志开启。

除了输出到日志文件里，慢查询还可以输出到表中：

```
mysql> CREATE TABLE `slow_log` (
    `start_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `user_host` mediumtext NOT NULL,
    `query_time` time NOT NULL,
    `lock_time` time NOT NULL,
    `rows_sent` int(11) NOT NULL,
    `rows_examined` int(11) NOT NULL,
    `logical_reads` int(11) NOT NULL,
    `physical_reads` int(11) NOT NULL,
    `db` varchar(512) NOT NULL,
    `last_insert_id` int(11) NOT NULL,
    `insert_id` int(11) NOT NULL,
    `server_id` int(10) unsigned NOT NULL,
    `sql_text` mediumtext NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'

mysql> SET GLOBAL log_outout='TABLE';
```

>我自己尝试的时候没能成功的让log输出到table中，做的修改也就是新建table slow_log, 然后修改log_putput为table，然后 select sleep(5), 郁闷……

使用`mysqldumpslow`命令可以分析满查询日志， `mysqldumpslow --help`。

>另外可以通过配置额外的参数: `long_query_io`将超过指定逻辑IO次数的SQL也记录到slow log中。 参数`slow_query_type`用来表示启用slow log的方式，可选值为：
 - 0：不记录slow log； 
 - 1：根据运行时间记录；
 - 2：根据逻辑IO次数记录；
 - 3：根据时间和逻辑IO次数记录
这个是网易杭研自己实现的一个MySQL的分支InnoSQL所提供的一些功能。
 
**二进制日志**

二进制日志(binary log)记录了对MySQL数据库执行更改的所有操作，主要用于恢复，复制（主从复制）和审计（可以通过对二进制日志内容的审计来判断是否有对数据库的注入攻击）。

二进制日志也可在配置文件中关闭，主要参数如下：

```
mysql> show variables like '%binlog%';
+-----------------------------------------+----------------------+
| Variable_name                           | Value                |
+-----------------------------------------+----------------------+
//事务提交之前，所有未被提交的二进制日志都被会放到一个缓存中，该缓存大小由binlog_cache_size来控制，也就是每个事务会有一个自己的二进制日志缓存，所以该参数不用很大。
//如果一个事务的二进制日志大小超过binlog_cache_size，则会被先写到磁盘的一个临时文件上。可通过global status: binglog_cache_use, binlog_cache_disk_use来check这个大小设置的是否合理。（默认32k）
| binlog_cache_size                       | 32768                | 

| binlog_checksum                         | CRC32                |
| binlog_direct_non_transactional_updates | OFF                  |

//二进制日志格式，有：STATEMENT, ROW, MIXED。
//如果使用STATEMENT，当sql语句中有rand, uuid这样的函数的时候，可能会导致主从数据库不一致。
//ROW记录行的物理变化，这样会导致二进制日志文件比较大，而主从复制也是采用传输二进制日志的形式，文件大的话，对于复制的效率也有影响。
//MIED默认用STATEMENT记录，必要的时候用ROW记录。
| binlog_format                           | STATEMENT            |
| binlog_max_flush_queue_time             | 0                    |
| binlog_order_commits                    | ON                   |
| binlog_row_image                        | FULL                 |
| binlog_rows_query_log_events            | OFF                  |
| binlog_stmt_cache_size                  | 32768                |
| innodb_api_enable_binlog                | OFF                  |
| innodb_locks_unsafe_for_binlog          | OFF                  |
| max_binlog_cache_size                   | 18446744073709547520 |

//单个二进制日志文件的最大值，超过之后会生成新的文件，而且编号+1。
| max_binlog_size                         | 1073741824           |
| max_binlog_stmt_cache_size              | 18446744073709547520 |

//sync_binlog=[N]，表示每写N次缓冲就同步到磁盘。如果为1，那么每次写二进制日志都会同步到磁盘，如果为0，那么InnoDB不会主动调fync把数据同步到磁盘，而是依靠操作系统的文件系统自行同步。

//该参数的设置对于数据库的性能会有很大影响，设置为0和1，虽然性能差距不至于到一个数量级，但是还是会有好几倍。
| sync_binlog                             | 0                    |
+-----------------------------------------+----------------------+
15 rows in set (0.01 sec)
```

看到有脚本，能将binary log转化为sql script（format：STATEMENT），想想用处估计也比较有限，这里就不记录详细内容了，具体点击[这里](http://yaozb.blog.51cto.com/2762349/511896)。
用binary log恢复数据库可用`mysqlbinlog`，简单用法：

 > mysqlbinlog shdsh-zhaogx3-bin.000001 --start-datetime="2014-07-04 15:00:00" | mysql -uroot -p
 mysqlbinlog shdsh-zhaogx3-bin.000001 --start-position=200 | mysql -uroot -p
 
 _另外发现一个命令： `flush logs;`可用来刷新binary log，会重新生成一个二进制日志文件。点击[这里](http://www.xifenfei.com/245.html)。_
 
 ###InnoDB存储引擎文件
 
 **表空间文件**
 
 默认都使用ibdata1这个文件，设置了`innodb_file_per_table`之后，每个表自己用一个表空间文件，该文件用于存储表数据、索引和插入缓冲BITMAP等信息。
 
 **重做日志文件**
 
 默认情况下datedir下面会有两个重做日志文件（redo log）：ib\_logfile0和ib\_logfile1
 
 影响重做日志的几个参数：
 
  - innodb\_log\_file\_size: 每个重做日志文件最大大小
  - innodb\_log\_file\_in\_group：每个日志文件组中重做日志的数量（循环写入）
  - innodb\_mirrored\_log\_groups：日志镜像文件组的数量
  - innodb\_log\_group\_home\_dir：位置

重做日志文件不能太大也不能太小，太大恢复时间比较长，太小可能需要多次切换重做日志文件。（对InnoDB性能有很大影响）
  
二进制文件和重做日志文件的比较：
|Binary Log|Redo Log|
|----|----|
|记录所有对MySQL数据库的改动|记录InnoDB存储引擎自身的事务日志|
|记录格式可以是STATEMENT, ROW和MIXED | 记录的是每个页的物理变化 |
|仅在事务提交之前记录 | 在事务进行过程中不断记录|
|适用于point-in-time恢复| 适用于crash revocery恢复|
|不断生成新文件 | 循环写入|

基本上区别就这样吧，详细一点的可以看[这里](http://ask.chinaunix.net/question/772)。主要觉得二进制日志和重做日志很像，是因为都是记录事务的，都用作恢复，主要是要搞清楚这两种文件用在什么状况下的恢复。
能控制重做日志刷新的磁盘一个条件是master thread, 另一个是参数`innodb_flush_log_at_trx_commit`,该参数用于在commit时，控制处理重做日志的方式。

 - 0 不处理，等待master thread每秒刷新
 - 1 commit时同步到磁盘并且调用fsync
 - 2 commit时同步到磁盘，但是只是写入文件系统的缓存，由操作系统的文件系统保证是否写入到磁盘。

---

##文件

这一部分也是讲格式什么的比较多，就随便记录一些东西了。

创建存储过程：

```
delimiter //

create procedure load_t1(count int unsigned)
begin
declare s int unsigned default 1;
declare c varchar(20) default repeat('a', 20);
while s<= count do
insert into t1 select null, c;
set s=s+1;
end while;
end;
//

delimiter ;

call load_t1(20);

//show procedure status;
//call proc_name;
```

MySQL中varchar长度最大为65532，这里的长度其实是字节数，不是字符数，根据你用的编码不同，支持的最大字符长度也不同。其实这里的最大长度是指一个表中所有varchar类型的数据的总长度，如果一个表中创建的类型为varchar的列的总长度大于65535，也会报错。另外，对于char类型，如果存储的数据时多字节字符编码的（GBK,UTF-8之类），其实在存储引擎内部会被视为变长字符类型。

如果varchar列存储的数据过长，则会发生行逸出。（就是只有一段存储在当前数据页，剩余的存储在BLOB页上面。）主要是因为InnoDB存储引擎表是索引组织的，既B+树结构，最起码要确保一个数据页上面有两条记录，不然就成链表了。

InnoDB的行结构就算了，这个肯定会忘。

创建Trigger：

```
delimiter //

create trigger trg_usercash_update before update on usercash
for each row
begin
if new.cash - old.cash > 0 then
insert into usercash_err_log
select old.userid, old.cash, new.cash,USER(),NOW();
set new.cash = old.cash;
end if;
end;
//

delimiter ;

//show triggers;
```

MySQL里面trigger支持的动作有insert, update, delete, 触发时间：before,after, 并且只支持对于for each row的触发，不支持statement。

视图，在MySQL中视图是一种虚表，视图里面的数据没有物理存储（通过触发器，可以做到物化视图，Oracle支持物化视图）：

```
create view demo_haha 
as 
select demo.name as d_name, haha.name as h_name from demo as demo, haha as haha 
where demo.id > 15 and haha.id = 24 
with check option;
```

一般称可以进行更新操作的视图为可更新视图，这样的视图需要在视图定义的时候加上`with check option`。

###分区
MySQL支持的分区类型是水平分区（不同行记录在不同物理文件中，对应的有垂直分区）、局部索引分区（一个分区中既存放数据，又存放索引；而全局分区是指，数据存放在各个分区中，所有数据的索引存放在一个对象中）。

不管创建何种类型的分区，如果表中存在主键或者唯一索引时，分区列必须是唯一索引的一个组成部分。
分区类型有：

 - RANGE分区
 
    ```
    //create
    create table t1(
        id int
    ) engine=INNODB
    partition by range(id)(
        partition p0 values less then(10),
        partition p1 values less then(20),
        partition pdefault values less then maxvalue
    );
    
    //drop
    alter table t1 drop partition p0;
    
    //explain
    explain partitions select * from t1 where id < 10;
    ```
    
    RANGE分区多用于日期列的分区，比如按年，如果要删除某一年的数据，删除该年的分区即可；另外查询时，如果只需查该年的数据，则只会在该年的分区上进行查询，不会去遍历其他分区（称为：分区剪枝）。
    
    对于RANGE分区的查询，优化器只能对YEAR(), TO\_DAYS(), TO\_SECONDS(), UNIX\_TIMESTAMP()这类函数进行优化查询。(前三个函数是从‘0000-00-00 00:00:00’开始计算)
    
 - LIST分区

和RANGE分区类似，RANGE分区列的值是连续的，而LIST分区列的值是离散的

```
create table t2(
    a int,
    b int
) engine=innodb
partition by LIST(b) (
    partition p0 values in (1,3,5,7,9),
    partition p1 values in (0,2,4,6,8)
);
```

同时向LIST分区的表中插入多条数据的时候，如果有条数据不满足任何分区的要求，如果该表适用MyISAM引擎，则该数据之前插入的数据成功，之后的失败；如果是InnoDB引擎，则将其视为整个事务失败。

  - HASH分区

需要指明要进行hash计算的列或者表达式，同时需要指明分区个数

```
create table t3(
    a int,
    b DATETIME
)ENGINE=INNODB
partiton by hash(YEAR(b))
partitions 4;
```

hash函数就是取模，对于连续的值进行分区，数据分布会比较平均。
（MySQL还支持linear hash分区，类似于hash分区，但是分区算法更麻烦，与hash分区相比缺点在于各个分区间的数据分布可能不大均衡。）

- KEY分区

类似于hash分区，不同之处在于hash分区用自定义的函数进行分区，key分区使用数据库提供的函数分区。

```
create table t_key(
    a int,
    b datetime
)engine=innodb
partition by key(b)
partitions 4;
```

- COLUMNS分区

COLUMNS分区可以直接使用非整形的数据进行分区，分区根据类型直接比较而得，不需要转化为整形。支持的数据类型包括：INT, SMALLINT, TINYINT, BIGINT, DATE, DATETIME, CHAR, VARCHAR, BINARY, VARBINARY。上面的各种类型分区可以和COLUMNS分区相结合，如LIST COLUMNS, RANGE COLUMNS（支持多列）

子分区，对于特别大的表，可以在分区下面继续设置子分区，子分区只能是HASH或者KEY分区（这部分不做详细记录，知道有就行了）。

```
create table ts(
    a int,
    b datetime
)engine=innodb
partition by range(YEAR(b))
subpartition by hash(TO_DAYS(b))
subpartitions 2(
    partition p0 values less than(1990),
    partition p1 values less than(2000),
    partition p2 values less than maxvalue
);
```

处理null值，RANGE分区会将其放入最左边的分区，LIST分区需要指明null在那个分区(partition p0 values in (1,3,5,7,9,null)), 对于HASH和KEY分区，null值的分区函数运算结果为0.

###分区之后是否变快
对于查询来说，所谓的快慢跟IO有很大关系，能减少IO次数，查询的效率就会高，所以如果分区之后能减少查询的IO次数，则会变快。如果原来1000W的数据，B+树为3层，分10个区，每区100W，B+树2层，那么能加快查询速度（主键分区）。否则，意义可能不大。

另外，对于非分区列的查询，分区之后可能需要遍历所有分区，这样反倒增加了IO导致更慢！所以这也得看具体业务。

表和分区交换数据：`alter table {srcTable} exchage partition {parName} with table {newTable}`。执行这条命令要求也不少，比如数据结构必须一致，newTable不能有外键或者其他表对其的外键引用。

---

##索引

B+树索引，为了保持平衡，再插入新数据的时候不会做大量的拆分页操作，因为这样意味着磁盘的读写，所以采用类似与二叉树的旋转功能，当前页已满的情况下，尽可能将新插入的数据放到兄弟页，以此减少磁盘IO。B+树采用填充因子来控制树的变化，50%是填充因子最小值。

聚集索引和辅助索引（非聚集索引）

索引的创建：
`create index {index_name} on {table_name} ({columns})` 和 `alter table {table_name} add {index|key} {index_name} (columns)`

show index from table的时候需要注意一下 **Cardinality**这个值， Cardinality表示索引中不重复记录数量的预估值， Cardinality / rows\_in\_table应该尽可能的接近1，如果非常小，那么就需要考虑一下这个索引是否有必要了。索引应该尽量建在那些高选择性的字段上面，在低选择性的字段上面建立索引没什么必要。

要知道什么是联合索引什么是覆盖索引（即从辅助索引中就可以得到要查询的记录），强制指定索引force index。

两个优化Multi-Range Read（MRR）和Index Condition Pushdown(ICP)，MRR主要是减少了IO，将查询得到的辅助索引键值存放在一个缓存中，然后根据ROWID进行排序，然后读取，基本意思就是查询得到的辅助索引键值并不立即读取，先缓存起来，然后对缓存的这些键值排序，排序之后再进行读取，这样减少了随机读。启用MRR可以通过参数@@optimizer\_switch中的mrr和mrr\_cost\_based来控制（mrr=on即可，mrr\_cost\_based设置为off的话，表示总是开启mrr）。 

ICP主要是将where条件的过滤放到了存储引擎层，没有ICP的时候，当进行索引查询时，先根据索引来查找记录，然后MySQL层再根据where条件来过滤记录，现在有了ICP，在取出索引的同时，判断是否可以进行where条件过滤。

InnoDB的全文检索先忽略吧，从来没用到过，也跟其他的索引有点不一样，记不住。

---

##锁

个人觉得，锁其实就是一直确保数据一致性的机制。

意向锁，书中关于意向锁的表述：意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度上进行加锁，InnoDB存储引擎中意向锁即为表级别的锁，设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型。 如果是这样的话，感觉意向锁加锁很多会失败或者是阻塞……（意向锁也没见有什么特别的用处）

一致性非锁定读，就是通过MVCC的方式来控制读取，相对的有一致性锁定读，顾名思义读取数据的时候锁定该数据（select ... for update; select ... lock in share mode）

InnoDB默认会为外键建立索引，以免出现数据不一致的问题。

不同事务隔离机制下会发生的异常有这么几种：
    
 - 脏读： 读到了还没提交的数据。 read committed事务级别会发生。
 - 不可重复读： 在一个事务内多次读取同一个数据集合，在这个事务还没结束的时候，另一个事务也访问同一数据集合并且做了一些DML操作，这样在第一个事务中两次读取操作，由于第二个事务的修改，读取到的数据可能是不一样的。 read repeatable事务级别会解决该问题。
 - 幻读： 在一个事务内多次读取同一个数据集合，在这个事务还没结束的时候，另一个事务也访问同一数据集合并且做了添加或者删除操作，这样在第一个事务中两次读取操作，很有可能有一次读取到在另外一次读取操作中根本不存在的数据（个人理解）。

幻读个人觉得可以作为不可重复读的一种特殊情况，通常在read repeatable事务级别可以解决不可重复读的问题，而幻读需要在serizable级别解决，但是InnoDB通过next-key在read repeatable级别也可以避免幻读问题。另外，一般来说不可重复读的问题是可以接受的，因为读到的是已经提交的数据，因此很多数据库厂商默认将数据库隔离级别设置为read committed。

---

##事务

###事务的实现

redo log，这个可以参照前面关于redo log的一些内容，undo log是逻辑日志，只是将数据库逻辑的恢复到原来的样子，比如对于insert操作roll back的时候会是delete操作，如果是物理日志，那么需要恢复整个页，这样显然是有问题的。

> 删除数据可以用delete语句，语法灵活，可回滚； 也可以用truncate table {table_name}， 该语句直接删除所有数据，不可回滚。

read-uncommitted, read-committed, repeatable-read, serializable

##备份与恢复

 - 逻辑备份： 备份出来的文件是可读的，一般是文本文件，比如用mysqldump和select * into outfile，优点在于导出的文件内容可读，缺点在于恢复所需要的时间比较长。
 - 裸文件备份： 复制数据库物理文件
 - 冷备： 只需要备份数据库的frm文件，共享表空间文件，独立表空间文件ibd，重做日志文件。优点是速度快，简单，缺点在于通常文件比逻辑文件大，不总是可以跨平台。
 
 mysqldump可以用来做逻辑备份，但是无法导出view，`mysqldump --help`可以用来查看所有参数，部分重要参数如下：

 - --single-transaction：该参数只对innoDB有效，在备份开始前会先执行 start transaction
 - --lock-tables: 依次锁住schema下所有的表，并不能确保所有架构下表的一致性，一般会用于MyISAM。
 - --lock-all-tables：同时锁住所有表
 - --add-drop-databases: 在运行create database前先运行drop database，需要和 --all-databases或者--databases选项一起使用
 - --events：备份事件调度器
 - --routines：备份存储过程和函数
 - --triggers：备份触发器
 - --hex-blob：将binary, varbinary, blog, bit类型备份为16进制
 - --where='condition'：导出给定条件的数据

SELECT ... INTO 

``` 
SELECT [columns]
INTO
OUTFILE 'file_name'
[{fields | columns}
    [TERMINATED BY '每个列的分割符']
    [ENCLOSED BY '字符串的包含符']
    [ESCAPED BY '转义符']
]
[LINES
    [STARTING BY '每行的开始符号']
    [TERMINATED BY '每行的结束符号']
]
FROM {TABLE_NAME}
WHERE {CONDITION}
```

要注意Windows('\r\n')和Linux('\r')换行符的不同

逻辑备份的恢复：

`#mysql -uroot -p < test.sql`

`mysql> source test.sql`

``` 
LOAD DATA 
INTO TABLE {table_name}
[CHARACTER SET charset_name]
[{fields | columns}
    [TERMINATED BY '每个列的分割符']
    [ENCLOSED BY '字符串的包含符']
    [ESCAPED BY '转义符']
]
[LINES
    [STARTING BY '每行的开始符号']
    [TERMINATED BY '每行的结束符号']
]
[IGNORE number LINES]
[(col_name_or_user_var, ...)]
[SET col_name= expr, ...]
```

恢复数据的时候可以暂时忽略外键检查： `set @@foreign_key checks=0;`

热备：

 - ibbackup：InnoDB官方提供的热备工具，但是收费。
 - XtraBackup：实现了ibbackup的所有功能，开源，并且支持真正的增量备份

快照备份：这种备份依赖于文件系统的快照功能。

复制： replication的工作原理可分为三个步骤，1. master将数据更改记录到binlog中；2. slave把主服务器的binlog复制到自己的中继日志(relay log)中； 3. slave重做中继日志中的日志，把更改应用到自己的数据库上。

复制不可能是实时的，尤其是在master压力比较大的时候延迟更大，`show slave status; show master status`。 复制可用来做备份，但不仅限于备份： 1. 数据分布；2. 读取的负载平衡； 3. 备份； 4. 高可用性和故障转移。

为避免master的drop database或者 drop table这样的误操作，一个比较好的办法是在slave上面对数据库所在分区做快照。

##优化

判断当前数据库的内存是否已经达到瓶颈，可通过查看当前服务器的状态，比较物理磁盘的读取和内存读取的比例来判断缓冲池的命中率。（不应该低于99%）

```
mysql> show global status like 'innodb%read%';
+---------------------------------------+----------+
| Variable_name                         | Value    |
+---------------------------------------+----------+
| Innodb_buffer_pool_read_ahead_rnd     | 0        | 
| Innodb_buffer_pool_read_ahead         | 0        |//预读的次数
| Innodb_buffer_pool_read_ahead_evicted | 0        |//预读的页，但是没有被读取就从缓冲池被替换的页的数量
| Innodb_buffer_pool_read_requests      | 6931     |//从缓冲池中读取页的次数
| Innodb_buffer_pool_reads              | 454      |//从物理磁盘读取的次数
| Innodb_data_pending_reads             | 0        |
| Innodb_data_read                      | 11702272 |//总共读入的字节数
| Innodb_data_reads                     | 712      |//发起读取请求的次数，每次读取可能需要读取多个页
| Innodb_pages_read                     | 453      |
| Innodb_rows_read                      | 122      |
+---------------------------------------+----------+
10 rows in set (0.00 sec)
```

缓冲池命中率 = innodb\_buffer\_pool\_read\_requests / (innodb\_buffer\_pool\_read\_requests + Innodb\_buffer\_pool\_read\_ahead + Innodb\_buffer\_pool\_reads)
6931 / (6931 + 0 + 454) = 0.938

基准测试工具：sysbench和mysql-tpcc
