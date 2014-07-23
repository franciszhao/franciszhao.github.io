---
layout: post
title: 《Java并发编程实践》读书笔记
category: book
---


##Summary
这本书也就将近300页的样子，关于这本书多么的厉害不多说了，相比去看这本书的人心里都有数。

个人感觉这本书还是比较偏理论，站在比较高的高度来指导你如何写出健壮的并发程序，而非介绍API的（虽然也有大量的例子）。看这种书会有种感觉就是云里雾里的，不知道自己到底吸收了多少（目前我就是这种感受），还是得靠自己有一些并发编程的经验或者多加练习，这样感觉效果会好一点。

---
##对象的共享
主要是由于“重排序”的存在，在没有正确同步的多线程环境下，可能得不到数据正确的值。

加锁，不仅仅确保原子性，也可以确保可见性。 volatile变量也可以确保可见性，但是并不能保证原子性。
而且volatile变量适用于：

 - 对变量的写不需要依赖变量当前值，或者只有单线程修改变量值
 - 该变量不会与其他状态变量一起纳入不变性条件
 - 在访问变量时不需要加锁
    
发布：使对象能够在当前作用域之外的代码中使用。 逸出：当某个不该发布的对象被发布时，这种情况称之为逸出。
主要是说要正确的发布对象，对象发布之后不能确保其他线程如何使用该对象，所以如果不是安全发布，那么该对象的不变性条件无法得到保证。发布一个对象的时候，可能会间接的发布该对象的非私有变量所引用的其他对象。 **不要在构造过程中使this引用逸出。**

满足同步需求的另外一种做法是使用不可变对象，不可变对象是绝对线程安全的。

 - 对象创建以后其状态就不能修改
 - 对象的所有域都是final的
 - 对象是正确创建的

不可变对象内部仍可以用可变对象来管理状态，注意`不可变对象`和`不可变的对象引用`之间的区别。

```
//在可变对象基础上构建的不可变对象
public final calss ThreeStooges {
	private final Set<String> stooges = new HashSet<String>();

	public ThreeStooges() {
		stooges.add("Moe");
		stooges.add("Larry");
		stooges.add("Curly");
	}

	public boolean inStooge(String name) {
		return stooges.contains(name);
	}
}
```
通过将域声明为final，也相应的告诉其他人员，这些域是不会变化的。 除非需要某个域是可变的，否则应将其声明为final域。

安全发布的常用模式：

 - 在静态初始化函数中初始化一个对象引用
 - 将对象的引用保存到volatile类型的域或者AtomicReferance对象中
 - 将对象的引用保存到某个正确构造对象的final类型域中
 - 将对象的引用保存到一个由锁保护的域中

在并发程序中使用和共享对象时，可以使用一些实用的策略，包括：

 - 线程封闭 （线程封闭的对象只能由一个线程拥有，对象被封闭在该线程中，并且只能由该线程修改）
 - 只读共享 
 - 线程安全共享 （线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有借口来进行访问而不需要进一步同步）
 - 保护对象 （被保护的对象只能通过持有特定的锁来访问。 保护对象包括封装在其他线程安全对象中的对象，以及已发布的并且由某个特定锁保护的对象。）

> 感觉搞懂双重检查锁定的话，这一块的内容能了解大约一半吧。
  
---

##对象的组合

在设计线程安全类的过程中，需要包含以下三个基本要素：

 - 找出构成对象状态的所有变量
 - 找出约束状态变量的不变性条件
 - 建立对象状态的并发访问管理策略

> 个人理解就是，找出这个对象里面可以被其他线程访问的变量，确定这些变量的不变性条件（规则），然后确保在修改这个对象的时候，时刻都能保持这些不变性条件。

 - 通过**实例封闭**的方式来构建线程安全的类
将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而容易确保线程在访问数据时总能持有正确的锁。

基本意思就是通过加锁（私有锁对象或者是对象的内置锁）来同步对数据的访问（Java监视器模式）。

 - 通过**委托**来实现线程安全
将当前类的线程安全性委托给一个已经确保是线程安全的类，基本意思是将当前类的状态保存在已知的线程安全类中。

当委托失效时（比如当前类有两个变量来标示状态，且不变性条件涉及到这两个变量），需要当前类自行构建线程安全策略。

###在现有的线程安全类中添加功能
 - 修改原有的类 （需要先理解代码中的同步策略，如果直接添加新方法到类中，那么意味着实现同步策略的所有代码仍处于同一个源文件中，从而更容易维护和理解。但是，源文件并不一定可修改）
 - 扩展这个类 （设计这个类时，并不一定考虑了可扩展性，父类的状态并不一定都向子类公开）
 - 客户端加锁 （主要问题在于需要知道被封装对象使用的是哪一个锁，即使知道了这样做还有一个问题，就是被封装类的加锁代码被放到了与与其完全无关的类中）
 - 组合 （新建的类通过内置锁为所有操作增加一层额外的锁，而不关心被组合类的安全策略。这样可能会造成性能损失，同时可能需要重写被组合类的所有方法。）

###将同步策略文档化
---
##基础构建模块
 BlockingQueue: `LinkedBlockingQueue`, `ArrayBlockingQueue`,`PriorityBlockingQueue` and `SynchronousQueue`
 BlockingDeque: `ArrayDeque` and `LinkedBlockingDeque`
###同步工具类
 - 闭锁 CountDownLatch (倒计时计数器)
 - FutureTask
 - 信号量 semaphore
 - 栅栏 CyclicBarrier（类似于闭锁，可重复利用；闭锁用于等待事件，而栅栏用于等待其他线程）
 - 栅栏 Exchanger (两方栅栏two-party，各方在栅栏位置上交换数据)
 
---
>  - 可变状态是至关重要的 （所有的并发问题都可以归结为如何协调对并发状态的访问。可变状态越少，就越容易确保线程安全性。）
 - 尽量将域声明为final类型，除非需要它们是可变的。
 - 不可变对象一定是线程安全的 （不可变对象能极大降低并发编程的复杂性。它们更为简单且安全，可以任意共享而无须使用加锁或保护性复制等机制。）
 - 封装有助于管理复杂性
 - 用锁来保护每一个可变变量
 - 当保护同一个不变性条件中的所有变量时，要使用同一个锁
 - 在执行符合操作期间，要持有锁
 - 如果从多个线程中访问同一个可变变量时没有同步机制，那么程序会出现问题
 - 不要故作聪明地推断出不需要使用同步
 - 在设计过程中要考虑线程安全，或者在文档中明确地指出它不是线程安全的
 - 将同步策略文档化

---

##任务执行

在线程中执行任务，一要搞清楚任务边界，二要确定现场调度策略

Executor Framework
 - Executor (interface)
 - ExecutorService (newFixThreadPool, newCachedThreadPool, newSingleThreadPool, newScheduledThreadPool, 这些pool都有一个线程池，然后默认情况下使用无界的LinkedBlockingQueue来保存待执行的任务，无界LinkedBlockingQueue的大小是Integer的最大值，这通常会有一定危险，任务执行速度小于生成速度的话，很有可能导致内存溢出。（这个地方有点类似于生产者消费者）)
 - Future
 - Callable
 - CompletionService (Executor & BlockingQueue)
 - invokeAll （执行一组任务，并且按照任务集合中迭代器的顺序将所有的Future添加到返回的集合中）

---

##取消和关闭

取消操作的原因：

 - 用户请求取消
 - 有时间限制的事件
 - 应用程序事件
 - 错误
 - 关闭
通常取消操作最合理的办法是中断。
`interrupt()`中断目标线程 （调用interrupt()并不意味着立即停止目标线程正在进行的工作，而只是传递了请求中断的消息）
`isInterrupt()`返回目标线程的中断状态
`interrupted()`清除当前线程的中断状态，并返回之前的值（始终没太搞清楚，清除中断状态和保存中断状态的意思）

响应中断一般有两种策略：

 - 传递异常 (从而使当前方法也成为可中断的阻塞方法)
 - 恢复中断状态 （从而使调用栈中的上层代码能够对其进行处理）

可以用Future来实现取消， `Future.cancel`。

处理不可中断的阻塞：

 - Java.io包中的同步Socket I/O （通过关闭底层的套接字，可以使得由于执行read或write方法而被阻塞的线程抛出一个SocketException）
 - Java.io包中的同步I/O (中断一个在InterruptibleChannel上等待的线程时，将抛出ClosedByInterruptException并关闭链路（这还会使得其他在这条链路上阻塞的线程同样抛出ClosedByInterruptException）。当关闭一个InterruptibleChannel时，将导致所有在链路操作上阻塞的线程都跑出AsynchronousException。)
 - Selector的异步I/O （如果一个线程在调用Selector.select方法时阻塞了，那么调用close或wakeup方法会使线程抛出ClosedSelectorException）
 - 获取某个锁 （如果一个线程由于等待某个内置锁而阻塞，那么将无法响应中断，因为线程认为它肯定会获得锁）
 
可以采用newTaskFor来封装非标准的取消 (处理不可阻塞中断这一块没遇到过，理解的一般)
 
###停止基于线程的服务

 - 关闭ExecutorService有两种方法：`shutdown()` and `shutdownNow()`
 - 毒丸对象

Thread API中提供了UncaughtExceptionHandler，设置Handler来处理未捕获异常。

守护进程 （不影响JVM的关闭）

---

##线程池的使用

在默认的一些线程池无法满足需求的时候，自行配置线程池（注意使用有界队列或者是无界队列）

```
public ThreadPoolExecutor(int corePoolSize,
                          int maxPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BolckingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {...}
```

考虑线程池配置时候需要注意的一些：

 - 线程运行时间
 - 任务是计算密集型还是I/O密集型
 - 注意线程的存活时间
 - 注意设置用来保存等待执行的任务的队列
 - 设置饱和策略 
   - AbortPolicy -- 中止 （默认的饱和策略，饱和时将会抛出RejectedExecutionException）
   - CallerRunsPolicy -- 调用者运行 (将某些任务退回给调用者线程去执行，由于执行需要时间，调用者线程在一段时间内不会提交任务，线程池得以有时间执行任务)
   - DiscardPolicy -- 抛弃
   - DiscardOldestPolicy -- 抛弃最老的

线程工厂，很多情况下都需要使用定制的线程工厂。例如，希望为线程池中的线程制定一个UncaughtExceptionHandler，或者实例化一个定制的Thread类用于执行调试信息的记录，或者希望给线程起个有意义的名字。

扩展ThreadPoolExecutor，它提供了几个可以在子类化中改写的方法：beforeExecute、afterExecute和terminated。
      
---

##避免活跃性危险

死锁：

 - 锁顺序死锁
 - 动态锁顺序死锁
 - 写作对象之间发生的死锁
 - 资源死锁 （两个线程分别以不同次序请求持有两个不同的资源）

开放调用，如果在调用某个方法时不需要持有锁，那么这种调用被称为开放调用。依赖于开放调用的类，通常能表现出更好的行为。这种通过开放调用来避免死锁的方法，类似于采用封装机制来提供线程安全的方法。

```
class Taxi {
    private Point location, destination;
    private final Dispatcher dispatcher;
    
    public synchronized Point getLocation() {
        return location;
    }
    
    public void setLocation(Point location) {
        boolean reachedDestination;
        synchronized (this) {
            this.location = location;
            reachedDestination = location.equals(destination);
        }
        if (reachedDestination) {
            dispatcher.notifyAvailable(this);
        }
    }
}


class Dispatcher {
    private final Set<Taxi> taxis;
    private final Set<Taxi> availableTaxis;
    
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxi.add(taxi);
    }
    
    public Image getImage() {
        Set<Taxi> copy;
        synchronized (this) {
            copy = new HashSet<Taxi>(taxis);
        }
        Image image = new Image();
        for (Taxi t : copy) {
            image.drawMarker(t.getLocation());
        }
        return image;
    }
}
```

通过可定时的锁可以避免死锁，通过线程转储信息可以分析死锁。

其他活跃性危险

 - 饥饿 （线程由于无法访问它所需要的资源而不能继续执行时，就发生了“饥饿”。如果在Java应用程序中对优先级使用不当，或者在持有锁时执行一些无法结束的结构，就有可能导致“饥饿”）
 - 糟糕的响应性 （某个线程长时间占用锁）
 - 活锁 （该问题尽管不会阻塞线程，但是也不能继续执行，因为线程将不断重复执行相同的操作，而且总会失败。比较像：两个很有礼貌的人，相互给对方让路，然后在另一条路上相遇，因此他们反复地避让下去。）（解决办法：在重试的时候加入随机性。）

> 最常见的活跃性问题就是锁顺序死锁，在设计程序时应该避免产生锁顺序死锁：确保线程在获取多个所时采用一致的顺序，最好的解决办法是在 *程序中始终使用开放性调用* 。

---

##性能与可伸缩性
衡量程序的性能一般可以从这两个方面：运行速度和处理能力

 - 运行速度：服务时间、延迟时间这些指标，主要指程序运行有多快
 - 处理能力：生产量、吞吐量这些指标，主要是指在计算机资源一定的情况下，能完成多少任务

在进行程序优化的时候（不管是“多块”还是“多少”），首先使程序正确，然后再提升性能，提升性能的时候要搞清楚想要提升的是哪方面的指标，而且要以测试为基准，不要妄加猜测。

关于多快，最终还是要受限于任务中有多少串行的部分，最高加速比为 1/ (F + (1-F)/N), 当N（处理器个数）趋近于无限大的时候，加速比趋近于1/F（串行部分的比例）。

 > 所以理想情况下，所有可以并行的地方都并行执行且相对于串行部分，时间可以忽略不计，那么任务的运行时间就约等于任务中串行部分需要的时间。
 
 关于多少，基本意思就是在增加计算资源（CPU，内存，存储容量，带宽）的时候，程序的吞吐量或者处理能力能够得到相应地增加。就是可扩展性。
  
线程引入的开销：
  
   - 上下文切换
   - 内容同步 （synchronized和volatile提供的可见性保证中可能会使用内存栅栏，内存栅栏会刷新缓存，是缓存无效。）（在无竞争的情况下JVM对于synchronized做了一些优化，比如如果只有一个线程访问当前锁对象，那JVM就可以通过优化去掉这个锁获取操作。还有粗化锁粒度啊之类的优化。）
   - 阻塞 （阻塞也有一点优化，就是阻塞之前先自旋，如果自旋一会就能获得锁那就不用挂起了，如果不能那在一定时间之后挂起。自旋的时间会根据之前自旋的状况调节，如果之前自旋一直不成功会直接挂起。）
   
减少锁的竞争，在并发程序中，对可伸缩性的最主要的威胁就是独占方式的资源锁：

 - 缩小锁的范围，快进快出
 - 缩小锁的粒度，细化锁
 - 锁分段（ConcurrentHashMap）
 - 避免热点域
 - 一些替代锁的方法（并发容器，读写锁，不可变对象以及原子变量）
 - 检测CPU的利用率（CPU利用率不高可能是：负载不足，I/O密集，外部限制，锁竞争）
 
---

##并发程序的测试

并发测试大致分为两类：安全性测试（不发生任何错误行为）和活跃性测试（某个良好的行为终究会发生）

> 这部分主要还是需要结合书中提供的例子

###正确性测试
 - 基本的单元测试 （类似于在串行上下文中执行的测试，也就是把当前的程序当成串行的测试一下）
 - 对阻塞的测试 （测试阻塞可以用中断，当然这要求被测试程序响应中断）
 - 安全性测试 （比如测试生产者消费者模式，可以通过一个对顺序敏感或者不敏感的校验和函数来测试。生产者生产随机数，然后计算总和；消费者拿到生产的随机数，然后计算总和。）
 - 资源管理的测试 （通常是测试是否做了不该做的事情，比如在不需要的时候应该销毁对对象的引用）
 - 使用回调 （通常回调函数的执行是在对象生命周期的一些已知位置上，并且这些位置非常适合判断不变性条件是否被破坏）
 - 产生更多交替行为 （Thread.yield）

###性能测试
 - 增加计时功能 （书中给的例子是给CyclicBarrier设置栅栏动作，the command to execute when the barrier is tripped。）
 - 多种算法的比较 （LinkedBlockingQueue比ArrayBlockingQueue的可伸缩性要好，主要是因为链表队列拥有更好的内存分配和GC等开销，而且与数组队列相比，链表队列的put和take等方法支持并发性更高的访问，因为一些优化后的链表队列算法能将头节点的更新操作和尾节点的更新操作分离开来。）
 - 响应性衡量 （主要是了解服务时间的变化情况。非公平的信号量通常能实现更好的吞吐量，而公平的信号量则实现更低的变动性。）
 
###避免性能测试的陷阱
 - 垃圾回收 （垃圾回收可能对某一次测试的时间产生很大影响，但是垃圾回收器可能在任何时刻运行，两种防止的策略：1，确保垃圾回收不执行；2，确保垃圾回收执行多次，这通常需要比较长的测试时间。）
 - 动态编译 （在某个时刻如果一个方法的运行次数足够多，那么动态编译器会将它编译为机器代码，当编译完成后，代码的执行方式将从解释执行变成直接执行。防止策略：程序运行足够长时间。在HotSpot中，如果在运行程序时使用命令行选项-xx:+PrintCompilation，那么当动态编译运行时，将输出一条信息。）
 - 对代码路径的不真实采样 （需要尽量覆盖代码路径）
 - 不真实的竞争程度 
 - 无用代码的消除 （优化编译器会找出并消除那些不会对输出结果产生任何影响的无用代码（Dead Code），优化之后对程序性能的影响无法预测。防止策略：计算某个派生对象的hashcode，并且将其与一个任意值比较。）
 
###其他测试办法
 - code review
 - 静态分析工具 （FindBugs）
 - 面向方面的测试技术
 - 分析和检测工具
 
---

##显式锁 -- ReentrantLock

ReentrantLock实现了Lock接口，并提供了与synchronized相同的互斥行和内存可见性。
ReentrantLock提供了更加灵活的功能：定时锁(tryLock())，轮询锁（tryLock()），可中断的锁获取操作(tryLock(timeout, unit))，非块结构的加锁，提供公平锁；而且性能比内置锁更好。

与显式锁相比，内置锁有自己的优势，内置锁为开发人员所熟悉，并且简介紧凑，现有许多程序都已经使用了内置锁；显式锁更危险，因为需要明确的unlock，内置锁自动释放锁；内置锁在线程转储中能给出哪些调用帧获得了哪些锁；内置锁是JVM的内置属性(synchronized)，它能执行一些优化（粗化锁粒度，单线程访问消除锁之类的），而且未来更有可能提升内置锁的性能。

###读写锁 ReadWriteLock
读写锁的一些可选实现包括：

 - 释放优先 （当一个写锁释放锁的时候，如果队列中同时存在读锁和写锁，应该优先选择哪个？还是最先发出请求的线程？）
 - 读线程插队 （如果锁由读线程持有，但有写线程在等待，那么新到达的读线程能否理解获得访问权？）
 - 重入性 （读锁和写锁是否是可重入的？）
 - 降级 （如果一个写线程持有锁，那么它是否可以在不释放锁的情况下获取读锁？）
 - 升级 （读锁能否优先于其他正在等待的读线程和写线程而升级为一个写入锁？）

> 这些可选实现是在自己实现ReadWriteLock的时候可以选择实现的功能，ReentrantReadWriteLock为这两种锁都提供了可重入的加锁语义，而且在构造的时候可以选择是一个公平的锁还是非公平的。
 
##构建自定义的同步工具
状态依赖性，条件队列，条件谓词
Lock -- 内置锁， Condition -- 内置条件队列
AbstractQueuedSynchronizer

> 这一章节前面都是一些理论，消化的一般；后面在介绍AQS，concurrent包里面很多同步工具都是基于AQS构建的。


CAS -- compare and swap

在竞争程度较高的情况下（线程本地计算较少），锁的性能要比原子变量好；但是在更真实的情况下，原子变量的性能将超过锁（竞争程度适中）。 另外，在这两种情况下，ThreadLocal的性能都是最好的。

> 不太明白为什么这里竞争程度适中的情况会更加真实，大概是因为书中来计较的时候，竞争程度较高的这种情况下，除了竞争锁或原子变量，什么都不做。

如果在某种算法中，一个线程的失败或者挂起不会导致其他线程也失败或者挂起，那么这种算法就被称为非阻塞算法。
如果在算法的每个步骤中都存在某个线程能够执行下去，那么这种算法被称为无锁算法。
如果在算法中仅将CAS用于协调线程之间的操作，并且能正确地实现，那么它既是一种无阻塞算法，也是一种无锁算法。

```
private static class Node <E> {
        public final E item;
        public Node<E> next;
        
        public Node(E item) {
            this.item = item;
        }
    }
    
    private static class NodeQ <E> {
        public final E item;
        public AtomicReference<NodeQ<E>> next;
        
        public NodeQ(E item, NodeQ<E> next) {
            this.item = item;
            this.next = new AtomicReference<NodeQ<E>>(next);
        }
    }
    
    //非阻塞栈
    public class ConcurrentStack <E> {
        AtomicReference<Node<E>> top = new AtomicReference<Test.Node<E>>();
        
        public void push(E item) {
            Node<E> newHead = new Node<E>(item);
            Node<E> oldHead;
            do {
                oldHead = top.get();
                newHead.next = oldHead;
            } while (!top.compareAndSet(oldHead, newHead));
        }
        
        public E pop() {
            Node<E> oldHead;
            Node<E> newHead;
            do {
                oldHead = top.get();
                if (null == oldHead) {
                    return null;
                }
                newHead = oldHead.next;
            } while (!top.compareAndSet(oldHead, newHead));
            return oldHead.item;
        }
    }
    
    //非阻塞链表
    public class LinkedQueue <E> {
        private final NodeQ<E> dummy = new NodeQ<E>(null, null);
        private final AtomicReference<NodeQ<E>> head = new AtomicReference<NodeQ<E>>(dummy);
        private final AtomicReference<NodeQ<E>> tail = new AtomicReference<NodeQ<E>>(dummy);
        
        public boolean put(E item) {
            NodeQ<E> newNodeQ = new NodeQ<E>(item, null);
            while (true) {
                NodeQ<E> currentTail = tail.get();
                NodeQ<E> tailNext = currentTail.next.get();
                if (currentTail == tail.get()) {
                    if (tailNext != null) {
                        //队列处于中间状态，推进尾节点
                        tail.compareAndSet(currentTail, tailNext);
                    } else {
                        //队列处于稳定状态，尝试插入新节点
                        if (currentTail.next.compareAndSet(null, newNodeQ)) {
                            //插入操作成功，尝试推进尾节点；此时如果推进尾节点不成功，可由其他线程帮其推进。
                            tail.compareAndSet(currentTail, newNodeQ);
                            return true;
                        }
                    }
                }
            }
        }
    }
     
```

CAS中的ABA问题，AtomicStampedReference（添加版本号）和AtomciMarkableReference（将节点保留在队列中的同时，将其标记为“已删除的节点”）通过不同的手段可以避免ABA问题。

---

##Java内存模型

参阅[ifeve.com](http://ifeve.com)
