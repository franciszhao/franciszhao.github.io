---
layout: post
title: MongoDB Sharding操作!
category: article
---

###为什么需要Shard
MongoDB的Shard是一种水平分片，主要目的无非就是提高总存储空间，减少单击压力这些，提高总负载这样。

Shard的基本介绍可以参照[官方文档](http://docs.mongodb.org/manual/core/sharding-introduction/)

###建立Shard集群

![](http://docs.mongodb.org/manual/_images/sharded-cluster-production-architecture.png)

MongoDB Shard集群的物理结构如上图所示，主要有三种节点（这几个中文名字是本人自己定义的）：路由节点（Routers），配置节点（config server），数据节点（shard）。三种节点的具体功能就不多少了，看完官方文档也差不多理解了。

> 上图中推荐的机器数目并非强制要求，主要是为了热备，提升性能，数据备份等。

Shard这一部分，上图所示是一个[replica set](http://docs.mongodb.org/manual/core/replication-introduction/)，就是多机数据备份集合，结构是这样的：一个primary node，多个secondary node，还有一个arbiter(裁决节点)，所有的读写操作都集中在primary node节点上，然后primary和 secondary之间会做数据同步。当primary宕机或者不可能之后，arbiter会从secondary中挑选一个成为新的primary node。arbiter不是必须的，甚至secondary也不是强制的。

 > Shard可以只是单台机器而非replica set

下面是本人操作的一些步骤，所有操作在一台Ubuntu虚拟机上完成：

 **1 启动数据节点**

```
./bin/mongod --port 27021 --dbpath data_27021 --fork --logpath logs/27021.log
./bin/mongod --port 27022 --dbpath data_27022 --replSet test27022 --fork --logpath logs/27021.log
./bin/mongod --port 27023 --dbpath data_27023 --replSet test27023 --fork --logpath logs/27021.log
```

这边我启动了三个数据节点，`--fork`表示已后台进程的方式运行， `--replSet`指定了这个节点所属replica set的名称，这里有test27022和test27023两个replica set，并且每个set只有一个节点。端口27021对应的实例留作演示shard为单台机器用。

 **2 启动config server**

```
./bin/mongod --configsvr --port 27024 --dbpath data_27024 --fork --logpath logs/27024.log
```
        
 **3 启动router**

```
./bin/mongos --configdb localhost:27024 --port 27025 --fork --logpath logs/27021.log 
```

启动的时候指明config server的地址，如果有多个以`,`分隔。

 **4 配置replica set**

```
./bin/mongo localhost:27022 #登录至replica set中任一机器

> use admin
switched to db admin

> rsconfig={_id:"test27022", members:[{_id:0, host:"localhost:27022",priority:1}]}
{
	"_id" : "test27022",
	"members" : [
		{
			"_id" : 0,
			"host" : "localhost:27022",
			"priority" : 1
		}
	]
}

> rs.initiate(rsconfig)  #设置replica set
{
	"info" : "Config now saved locally.  Should come online in about a minute.",
	"ok" : 1
}

> rs.status() # 查看设置结果
{
	"set" : "test27022",
	"date" : ISODate("2014-11-10T14:29:52Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "localhost:27022",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 507,
			"optime" : Timestamp(1415629406, 1),
			"optimeDate" : ISODate("2014-11-10T14:23:26Z"),
			"electionTime" : Timestamp(1415629407, 1),
			"electionDate" : ISODate("2014-11-10T14:23:27Z"),
			"self" : true
		}
	],
	"ok" : 1
}
```

重复上面的操作建立replica set "test27023"。

 **5 配置Sharding**
 
```
./bin/mongo localhost:27025 #登录至router

mongos> use admin #所有操作必须在admin下进行
switched to db admin

mongos> sh.addShard("test27022/localhost:27022"); #test27022是replica set的名字，localhost:27022是该set中primary node的地址
{ "shardAdded" : "test27022", "ok" : 1 }

mongos> sh.addShard("test27023/localhost:27023");
{ "shardAdded" : "test27023", "ok" : 1 }

mongos> db.runCommand({enableSharding:"uus_test"}) #设置允许shard的DB
{ "ok" : 1 }

mongos> db.runCommand({shardCollection:"uus_test.user", key:{"userId":1}}); 
#设置允许shard的collection，及其对应的shard key，这里"userId":1 表示选择userId作为shard key，值为1表示按范围分片。
{ "collectionsharded" : "uus_test.user", "ok" : 1 }

mongos> sh.status() 
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
  shards:
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022" }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27022	1
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : { "$maxKey" : 1 } } on : test27022 Timestamp(1, 0) 

```


添加shard还可以用另外的命令`db.runCommand({addShard:""})`，至此MongoDB Sharding集群搭建完毕。

关于Shard key的选择，MongoDB会为shard key这个字段建立索引，shard key的选择很关键，对于性能会有比较大的影响，尽量使用那些选择性比较大的字段（有点类似于关系型数据库建立索引）。如果查询包含shard key则只在一个shard里面查询即可，如果不包含则需要到所有shard里面查询（大多数时候。这样会比前一种情况慢）。这方面了解的不多，可以参考[这里](http://www.cnblogs.com/magialmoon/archive/2013/04/12/3017387.html) 和 [这里](http://my.oschina.net/costaxu/blog/196980#OSC_h1_5)

###修改chunk size

这是一个全局的参数。 默认是64MB。

小的chunk会让不同的shard数据量更均衡。 但会导致更多的Migration。

大的chunk会减少migration。不同的shard数据量不均衡。

这样修改chunk size。先连接上任意mongos

```
db.settings.save( { _id:"chunksize", value: 1 } )
```

单位是MB，这里修改为1M，方便插入数据之后看到多个chunk。另外可以通过在启动router的时候用参数 --chunkSize=1来指定chunk大小。

###插入数据

```
./bin/mongo localhost:27025 #登录到router

mongos> use uus_test;
switched to db uus_test

mongos> db.user.count();
0

mongos> for(i=0;i<100000;i++){db.user.insert({"userId":i,"name":"francis", "createDate": new Date()});}  # 插入10W数据
WriteResult({ "nInserted" : 1 })

mongos> db.user.count();
100000

mongos> db.user.stats() #查看user状态
{
	"sharded" : true,
	"systemFlags" : 1,
	"userFlags" : 1,
	"ns" : "uus_test.user",
	"count" : 100000,
	"numExtents" : 12,
	"size" : 11200000,
	"storageSize" : 22364160,
	"totalIndexSize" : 6066592,
	"indexSizes" : {
		"_id_" : 3254048,
		"userId_1" : 2812544
	},
	"avgObjSize" : 112,
	"nindexes" : 2,
	"nchunks" : 11,
	"shards" : {                            # sharding信息
		"test27022" : {
			"ns" : "uus_test.user",
			"count" : 36378,
			"size" : 4074336,
			"avgObjSize" : 112,
			"storageSize" : 11182080,
			"numExtents" : 6,
			"nindexes" : 2,
			"lastExtentSize" : 8388608,
			"paddingFactor" : 1,
			"systemFlags" : 1,
			"userFlags" : 1,
			"totalIndexSize" : 2207520,
			"indexSizes" : {
				"_id_" : 1185520,
				"userId_1" : 1022000
			},
			"ok" : 1
		},
		"test27023" : {
			"ns" : "uus_test.user",
			"count" : 63622,
			"size" : 7125664,
			"avgObjSize" : 112,
			"storageSize" : 11182080,
			"numExtents" : 6,
			"nindexes" : 2,
			"lastExtentSize" : 8388608,
			"paddingFactor" : 1,
			"systemFlags" : 1,
			"userFlags" : 1,
			"totalIndexSize" : 3859072,
			"indexSizes" : {
				"_id_" : 2068528,
				"userId_1" : 1790544
			},
			"ok" : 1
		}
	},
	"ok" : 1
}
```

从`db.user.stats()`的结果可以看出来，user这个collection被sharded，数据分布在test27022和test27023两个shard上面，test27022上面有36378条数据，大小为4.07MB；test27023上面有63622条数据，大小为7.12MB。


```
mongos> use admin
switched to db admin

mongos> sh.status()  #查看整个shard的信息
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
  shards:
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022" }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27022	5
				test27023	6
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27022 Timestamp(2, 1) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27022 Timestamp(1, 3) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27022 Timestamp(3, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27022 Timestamp(4, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27022 Timestamp(5, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(5, 1) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27023 Timestamp(3, 4) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(4, 2) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 

```

上面的结果中： "shards"表示当前集群有哪些shards；"databases"列出了所有的DB，"partitioned"表示是否可以shard， "primary"表示该DB的主分区，未被sharded的collection会存储在主分区上。

然后chunks这一部分是user这个collection中数据分布的情况，从上面可以看出 user的数据在test27022中有5个chunk，在test27023中有6个chunk。

`{ "userId" : 0 } -->> { "userId" : 4978 } on : test27022 Timestamp(1, 3)`表示userId在[0，4987)这个区间的，存储在test27022上面， Timestamp表示什么意思我也还没搞清楚。

`{ "$minKey" : 1 }`和`{ "$maxKey" : 1 }`表示极小值和极大值。

> 数据分布的并不是十分平均，range shard这种方式具体是怎么样的原理，还没搞清楚。

###新增shard

新增的shard可以不为空，但是不允许包含现有集群中已有的DB。这点没有具体尝试，本例中添加的是一个空的shard。

上面已经建立了一个shard集群，现在把最开始启动的localhost:27021这个实例添加进来，作为单台机器添加进来，至于replica set操作步骤一致。

```
./bin/mongo localhost:27025

mongos> use admin
switched to db admin

mongos> sh.addShard("localhost:27021");
{ "shardAdded" : "shard0000", "ok" : 1 }

mongos> sh.status();
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
  shards:
	{  "_id" : "shard0000",  "host" : "localhost:27021" }
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022" }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				shard0000	3
				test27022	4
				test27023	4
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : shard0000 Timestamp(7, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27022 Timestamp(7, 1) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27022 Timestamp(3, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27022 Timestamp(4, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27022 Timestamp(5, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : shard0000 Timestamp(6, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : shard0000 Timestamp(8, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 

```

新增shard很简单，另外也可以看到新增shard之后mongoDB自动就开始同步数据了。

> 当chunk过多的时候（大概是超过20个）, sh.status()的结果不会显示每个chunk的信息，若需要强制显示：sh.status(true);


之所以会自动同步数据是因为MongoDB有个balancer在运行，查看balancer信息：

```
mongos> use config
switched to db config

mongos> db.locks.find({_id: "balancer"}).pretty();
{
	"_id" : "balancer",
	"state" : 0,
	"who" : "user-VirtualBox:27025:1415629297:1804289383:Balancer:1681692777",
	"ts" : ObjectId("5460ddef20ab13caa089ccd7"),
	"process" : "user-VirtualBox:27025:1415629297:1804289383",
	"when" : ISODate("2014-11-10T15:46:55.249Z"),
	"why" : "doing balance round"
}

```

可以通过命令来设置balancer是否运行: sh.startBalancer()， sh.stopBalancer()。更多命令请参考[官方文档](http://docs.mongodb.org/manual/reference/method/)

config数据库下面有这些collection：

```
mongos> show collections;
changelog
chunks
collections
databases
lockpings
locks
mongos
settings
shards
system.indexes
tags
version
```

`chunks`可查看每个chunk的详细信息， `settings`可查看系统参数，如chunkSize


###移除shard

移除shard需要等待balancer将需要移除的shard上面的数据迁移到其他shards。如果移除的shard是某个DB的primary shard，则需要为该DB设置新的primary shard。

```
mongos> use admin
switched to db admin

mongos> db.runCommand({removeShard:"localhost:27021"})
{
	"msg" : "draining started successfully",
	"state" : "started",
	"shard" : "shard0000",
	"ok" : 1
}

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
  shards:
	{  "_id" : "shard0000",  "host" : "localhost:27021",  "draining" : true }
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022" }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27022	6
				test27023	5
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27022 Timestamp(9, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27022 Timestamp(7, 1) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27022 Timestamp(3, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27022 Timestamp(4, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27022 Timestamp(5, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(10, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27022 Timestamp(11, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 

mongos> db.runCommand({removeShard:"localhost:27021"})
{
	"msg" : "removeshard completed successfully",
	"state" : "completed",
	"shard" : "shard0000",
	"ok" : 1
}

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
  shards:
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022" }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27022	6
				test27023	5
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27022 Timestamp(9, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27022 Timestamp(7, 1) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27022 Timestamp(3, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27022 Timestamp(4, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27022 Timestamp(5, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(10, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27022 Timestamp(11, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 

```

首先`db.runCommand({removeShard:"localhost:27021"})`， 提示`"draining started successfully"`，这个时候开始迁移localhost:27021上面的数据。

这个时候查看shard信息会看到localhost:27021这个shard显示`"draining" : true `，表明正在迁移数据。当user只存在test27022和test27023这两个shard上面的时候，表明27021以完成数据迁移。

此时再次运行`db.runCommand({removeShard:"localhost:27021"})`，完成shard的移除。

下面尝试移除test27022，该分区是uus_test的primary shard：

```
mongos> db.runCommand({removeShard:"localhost:27022"})
{
	"msg" : "draining started successfully",
	"state" : "started",
	"shard" : "test27022",
	"note" : "you need to drop or movePrimary these databases",
	"dbsToMove" : [
		"uus_test"
	],
	"ok" : 1
}

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
  }
  shards:
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022",  "draining" : true }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27023	7
				test27022	4
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27023 Timestamp(12, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27023 Timestamp(13, 0) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27022 Timestamp(13, 1) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27022 Timestamp(4, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27022 Timestamp(5, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(10, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27022 Timestamp(11, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
  }
  shards:
	{  "_id" : "test27022",  "host" : "test27022/localhost:27022",  "draining" : true }
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27022" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27023	11
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27023 Timestamp(12, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27023 Timestamp(13, 0) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27023 Timestamp(14, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27023 Timestamp(15, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27023 Timestamp(16, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(10, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27023 Timestamp(17, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 

mongos> db.runCommand({moveprimary:"uus_test", to:"localhost:27023"})  #迁移至新的primary shard
{ "primary " : "test27023:test27023/localhost:27023", "ok" : 1 }

mongos> db.runCommand({removeShard:"localhost:27022"})
{
	"msg" : "removeshard completed successfully",
	"state" : "completed",
	"shard" : "test27022",
	"ok" : 1
}

mongos> sh.status();
--- Sharding Status --- 
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("5460c9f220ab13caa089c97c")
}
shards:
	{  "_id" : "test27023",  "host" : "test27023/localhost:27023" }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "uus_test",  "partitioned" : true,  "primary" : "test27023" }
		uus_test.user
			shard key: { "userId" : 1 }
			chunks:
				test27023	11
			{ "userId" : { "$minKey" : 1 } } -->> { "userId" : 0 } on : test27023 Timestamp(12, 0) 
			{ "userId" : 0 } -->> { "userId" : 4978 } on : test27023 Timestamp(13, 0) 
			{ "userId" : 4978 } -->> { "userId" : 15714 } on : test27023 Timestamp(14, 0) 
			{ "userId" : 15714 } -->> { "userId" : 26181 } on : test27023 Timestamp(15, 0) 
			{ "userId" : 26181 } -->> { "userId" : 36378 } on : test27023 Timestamp(16, 0) 
			{ "userId" : 36378 } -->> { "userId" : 47401 } on : test27023 Timestamp(10, 0) 
			{ "userId" : 47401 } -->> { "userId" : 57463 } on : test27023 Timestamp(17, 0) 
			{ "userId" : 57463 } -->> { "userId" : 69238 } on : test27023 Timestamp(8, 1) 
			{ "userId" : 69238 } -->> { "userId" : 80487 } on : test27023 Timestamp(4, 4) 
			{ "userId" : 80487 } -->> { "userId" : 90351 } on : test27023 Timestamp(5, 2) 
			{ "userId" : 90351 } -->> { "userId" : { "$maxKey" : 1 } } on : test27023 Timestamp(5, 3) 
```

操作步骤同移除普通的shard一样，多了移动uus_test到新的primary shard的操作`db.runCommand({moveprimary:"uus_test", to:"localhost:27023"})`。

###其他

 - 手动移动chunk
    
手动移动chunk的机会很少
```
mongos> db.adminCommand({moveChunk: "uus_test.user", find:{userId:10}, to:"localhost:27021"})
```

 - 故障恢复
当集群中某个分片宕掉以后，只要不涉及到该节点的操纵仍然能进行。当宕掉的节点重启后，集群能自动从故障中恢复过来。
