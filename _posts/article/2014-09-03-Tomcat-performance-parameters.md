---
layout: post
title: Tomcat性能调优部分参数
category: article
---

临时兴起，就搜了一下相关的文章，看到一些普遍在说的参数，就记录一下。本人也没做很详细的测试，以后如果做了测试就再发出来吧。

首先，确保你的操作系统是64位的，32位操作系统可使用的内存大小是有限制的，理论上是2G，我的印象中之前一个朋友说他32位桌面操作系统占到了3.2G的内存，不管怎么说32位操作系统可管理的内存是比较小的。而64位可管理的内存虽然也有上限，但对于目前硬件来说足够了。所以，优化第一步，确保操作系统是64位的。

然后优化基本上集中在两个地方，一个是JVM配置参数的优化，一个是Tomcat配置参数的优化。

###JVM参数
一般来说都会配置一下-Xms, -Xmx这两个参数，配置在catalina.sh（Windows是catalina.bat）中。除此之外，还有很多：

```
export JAVA_OPTS="
	-server 
	-Xms1400M -Xmx1400M 
	-Xss512k 
	-XX:+AggressiveOpts 
	-XX:+UseBiasedLocking 
	-XX:PermSize=128M -XX:MaxPermSize=256M 
	-XX:+DisableExplicitGC 
	-XX:MaxTenuringThreshold=31 
	-XX:+UseConcMarkSweepGC -XX:+UseParNewGC  -XX:+CMSParallelRemarkEnabled 
	-XX:+UseCMSCompactAtFullCollection 
	-XX:LargePageSizeInBytes=128m  
	-XX:+UseFastAccessorMethods 
	-XX:+UseCMSInitiatingOccupancyOnly 
	-Djava.awt.headless=true"
```

这么一大堆参数，酷不酷！！酷到你想哭……
 一个一个来吧!

|param|value|description|
|----|----|----|
|-server||表明JVM是运行在server模式下，默认是运行在client模式下，server模式肯定比client模式有更多优化，更好的性能，所以没理由不把这个加上。|
|-Xms -Xmx||JVM内存最小值和最大值，一般来说设置的最小值会小于最大值，但也有地方说需要保持这两个值大小一致，原因是随着系统并发越来越高，内存使用情况逐渐上升，达到最高点之后开始回落，回落的代价是CPU高速运转进行垃圾回收，如果遇到内存大起大落的状况，严重情况下会造成系统的假死，因为JVM在进行垃圾回收。（个人感觉这个说法有待验证，现在的垃圾回收器已经比较智能了，不至于会对JVM中运行的线程造成很大影响。最大值最小值设置为一样的值，会使得JVM在GC之后不用为新的对象重新分配内存，倒是真的。另外，最小值设置的比最大值小，似乎也没什么好处。）|
|-Xss||设定每个线程的堆栈大小，依照实际情况而定。一般不易超过1M。（我看我们的生存环境设置为2M……）|
|-Xmn|官方推荐为整个堆得3/8|设置年轻代大小，增大年轻代将减少老年代的大小。整个堆的大小 = 年轻代大小+老年代大小。（这里说明一下-Xms -Xms设置的是堆内存大小。）|
|-XX:+AggressiveOpts||启用这个参数之后，每当JDK升级之后，JVM都会使用最近加入的优化技术。|
|-XX:+UseBiasedLocking||启用一个优化了的线程锁，这个优化使得app server内对线程处理自动进行最优调配。|
|-XX:PermSize -XX:MaxPermSize|PermSize默认物理内存1/64, MaxPermSize默认为物理内存1/4|PermSize设置非堆内存初始化值, MaxPermSize为非堆内存的最大值，非堆内存包括方法区和其他JVM内部处理或者优化需要的内存。|
|-XX:+DisableExplicitGC| | 不允许在程序中显式调用‘System.gc()’，如果调用了，强制进行内存回收，导致的问题跟 -Xms -Xmx一样（感觉有待验证）|
|-XX:+UseParNewGC| |对年轻代采用多线程并行回收。可与CMS收集同时使用，在serial基础上实现的多线程收集器|
|-XX:+UseParallelGC| |选择垃圾收集器为并行收集器。此配置仅对年轻代有效。可以同时并行多个垃圾收集线程，此时用户必须停止|
|-XX:+UseConcMarkSweepGC||即CMS gc，它使用的是gc估算触发和heap占用触发|
|-XX:MaxTenuringThreshold||设置垃圾最大年龄，如果为0，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代对象比较多的应用，这样可以提高效率。如果将此值设置的较大，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。|
|-XX:+CMSParallelRemarkEnabled||在使用UseParNewGC 的情况下, 尽量减少 mark 的时间|
|-XX:+UseCMSCompactAtFullCollection||在使用concurrent gc 的情况下, 防止 memoryfragmention, 对live object 进行整理, 使 memory 碎片减少。|
|-XX:LargePageSizeInBytes||指定 Java heap的分页页面大小|
|-XX:+UseFastAccessorMethods||get,set 方法转成本地代码|
|-XX:+UseCMSInitiatingOccupancyOnly||指示只有在 oldgeneration 在使用了初始化的比例后concurrent collector 启动收集|
|-XX:CMSInitiatingOccupancyFraction| 70 |CMSInitiatingOccupancyFraction，这个参数设置有很大技巧，基本上满足(Xmx-Xmn)*(100- CMSInitiatingOccupancyFraction)/100>=Xmn就不会出现promotion failed。在我的应用中Xmx是6000，Xmn是512，那么Xmx-Xmn是5488兆，也就是年老代有5488 兆，CMSInitiatingOccupancyFraction=90说明年老代到90%满的时候开始执行对年老代的并发垃圾回收（CMS），这时还 剩10%的空间是5488*10%=548兆，所以即使Xmn（也就是年轻代共512兆）里所有对象都搬到年老代里，548兆的空间也足够了，所以只要满 足上面的公式，就不会出现垃圾回收时的promotion failed；因此这个参数的设置必须与Xmn关联在一起。|

###Tomcat容器内优化

```
conf/server.xml

<Connector port="8080" 
		  protocol="HTTP/1.1"
          URIEncoding="UTF-8"  
          minSpareThreads="25" 
          maxSpareThreads="75"
          enableLookups="false" 
          disableUploadTimeout="true" 
          connectionTimeout="20000"
          acceptCount="300"  
          maxThreads="300" 
          maxProcessors="1000" 
          minProcessors="5"
          useURIValidationHack="false"
          compression="on" 
          compressionMinSize="2048"
          compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain"
          redirectPort="8443"
/>
```

|param|value|description|
|----|----|----|
|URIEncoding=”UTF-8”||使得tomcat可以解析含有中文名的文件的url，真方便，不像apache里还有搞个mod_encoding，还要手工编译|
|maxSpareThreads||maxSpareThreads 的意思就是如果空闲状态的线程数多于设置的数目，则将这些线程中止，减少这个池中的线程总数。|
|minSpareThreads||最小备用线程数，tomcat启动时的初始化的线程数。|
|enableLookups||这个功效和Apache中的HostnameLookups一样，设为关闭。|
|connectionTimeout||connectionTimeout为网络连接超时时间毫秒数。|
|maxThreads||maxThreads Tomcat使用线程来处理接收的每个请求。这个值表示Tomcat可创建的最大的线程数，即最大并发数。|
|acceptCount||acceptCount是当线程数达到maxThreads后，后续请求会被放入一个等待队列，这个acceptCount是这个队列的大小，如果这个队列也满了，就直接refuse connection。|
|maxProcessors与minProcessors||在 Java中线程是程序运行时的路径，是在一个程序中与其它控制线程无关的、能够独立运行的代码段。它们共享相同的地址空间。多线程帮助程序员写出CPU最 大利用率的高效程序，使空闲时间保持最低，从而接受更多的请求。通常Windows是1000个左右，Linux是2000个左右。|
|useURIValidationHack||把useURIValidationHack设成"false"，可以减少它对一些url的不必要的检查从而减省开销。|
|enableLookups="false"||为了消除DNS查询对性能的影响我们可以关闭DNS查询，方式是修改server.xml文件中的enableLookups参数值。|
|disableUploadTimeout||类似于Apache中的keeyalive一样|
|compression|on|打开压缩功能|
|compressionMinSize|2048|启用压缩的输出内容大小，这里面默认为2KB|
|noCompressionUserAgents|gozilla, traviata| 对于以下的浏览器，不启用压缩|
|compressableMimeType|text/html,text/xml|压缩类型|
|protocol||可以启用其他协议，例如：protocol="org.apache.coyote.http11.Http11AprProtocol"， [APR](http://tomcat.apache.org/tomcat-6.0-doc/apr.html)的作用是使用JNI的方式来读取文件以及进行网络传输，这样做的好处是可以大大提示tomcat对静态文件的处理能力。|

> 同样的8443端口也可以这样配置。

基本上是参照[这里](http://blog.csdn.net/lifetragedy/article/details/7708724)的。
