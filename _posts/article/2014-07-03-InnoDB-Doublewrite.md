---
layout: post
category: article
title: InnoDB Double Write
---

Doublewrite是InnoDB的一个关键特性，为数据页提供了更高的可靠性。

**什么是Doublewrite**

InnoDB将缓存中的脏页刷新到磁盘的时候，并不直接写磁盘，而是通过memcpy函数将脏页复制到内存中的doublewrite buffer，然后将doublewrite buffer中的内容写入InnoDB共享表空间的doublewrite物理磁盘上，然后马上调用fsync函数同步磁盘；之后再将doublewrite buffer中的页写入各个表空间。

**为什么需要Doublewrite**

InnoDB的页的大小一般为16KB，当刷新页到磁盘的时候很有可能只写了某页的一部分比如4KB，然后数据库宕机（停电或者OS crash），这种情况下页只写了一部分被称为部分写失效。

事物在提交之前会写redo log，如果发生上面所说的写失效，可能有人会想到用redo log来进行恢复。但是redo log里面记录的是对页的物理操作，比如offset 1000， write 'francis'，如果页本身已经损坏了，这种恢复毫无意义。这个时候需要页的一个副本，在用redo log恢复之前，需要先通过副本还原该页。

Doublewrite就提供了这样的副本，如果发生写失效，在恢复的时候InnoDB引擎会从共享表空间中的Doublewrite中找到该页的一个副本，先应用副本，然后再做恢复操作。

**性能影响**

Doublewrite写了两次，会不会对性能产生很大影响？（理论上感觉应该下降50%）

doublewrite由两部分组成，一部分是内存中的doublewrite buffer大小为2M，一部分是共享表空间里的doublewrite部分，是连续的128个页，大小也是2M。 当从buffer忘共享表空间写数据时，每次1M顺序写入，由于是连续写入所以这部分操作开销并不是很大。而从buffer写入表空间的时候（如果没doublewrite，就只有这部分操作），是离散的。所以总体来说性能影响很小（从脏页复制到doublewrite buffer可以忽略，都是内存操作），大概多了5%-10%的开销，但是为了数据一致性，这个代价是可以接受的。

**相关参数和状态**

``` 
 mysql> show variables like '%double%';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| innodb_doublewrite | ON    |
+--------------------+-------+
1 row in set (0.00 sec)

 mysql> show global status like 'innodb_dblwr%';
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| Innodb_dblwr_pages_written | 29360161 |
| Innodb_dblwr_writes        | 7304459  |
+----------------------------+----------+
2 rows in set (0.00 sec)
```

`Innodb_dblwr_pages_written`：The number of pages that have been written for doublewrite operations.

 `Innodb_dblwr_writes`：The number of doublewrite operations that have been performed.
 
 当Innodb\_dblwr\_pages\_written : Innodb\_dblwr\_writes比值比较小的时候，说明系统写入压力并不大。
 

 参考：

 [http://www.orczhou.com/index.php/2010/02/innodb-double-write/](http://www.orczhou.com/index.php/2010/02/innodb-double-write/)

 [http://www.mysqlperformanceblog.com/2006/08/04/innodb-double-write/](http://www.mysqlperformanceblog.com/2006/08/04/innodb-double-write/)

 《MySQL技术内幕：InnoDB存储引擎》

