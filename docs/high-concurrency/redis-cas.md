## 面试题
redis 的并发竞争问题是什么？如何解决这个问题？了解 redis 事务的 CAS 方案吗？

## 面试官心理分析
这个也是线上非常常见的一个问题，就是**多客户端同时并发写**一个 key，可能本来应该先到的数据后到了，导致数据版本错了。或者是多客户端同时获取一个 key，修改值之后再写回去，只要顺序错了，数据就错了。

而且 redis 自己就有天然解决这个问题的 CAS 类的乐观锁方案。

## 面试题剖析
某个时刻，多个系统实例都去更新某个 key。可以基于 zookeeper 实现分布式锁。每个系统通过 zookeeper 获取分布式锁，确保同一时间，只能有一个系统实例在操作某个 key，别人都不允许读和写。

![zookeeper-distributed-lock](/img/zookeeper-distributed-lock.png)

你要写入缓存的数据，都是从 mysql 里查出来的，都得写入 mysql 中，写入 mysql 中的时候必须保存一个时间戳，从 mysql 查出来的时候，时间戳也查出来。

每次要**写之前，先判断**一下当前这个 value 的时间戳是否比缓存里的 value 的时间戳要新。如果是的话，那么可以写，否则，就不能用旧的数据覆盖新的数据。

Java并发问题--乐观锁与悲观锁以及乐观锁的一种实现方式-CAS

首先介绍一些乐观锁与悲观锁:

　　悲观锁：总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。再比如Java里面的同步原语synchronized关键字的实现也是悲观锁。

　　乐观锁：顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于write_condition机制，其实都是提供的乐观锁。在Java中java.util.concurrent.atomic包下面的原子变量类就是使用了乐观锁的一种实现方式CAS实现的。

 

乐观锁的一种实现方式-CAS(Compare and Swap 比较并交换)：

　　锁存在的问题:

　　　　Java在JDK1.5之前都是靠 synchronized关键字保证同步的，这种通过使用一致的锁定协议来协调对共享状态的访问，可以确保无论哪个线程持有共享变量的锁，都采用独占的方式来访问这些变量。这就是一种独占锁，独占锁其实就是一种悲观锁，所以可以说 synchronized 是悲观锁。

　　　　悲观锁机制存在以下问题：　　

　　　　　　1. 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。

　　　　　　2. 一个线程持有锁会导致其它所有需要此锁的线程挂起。

　　　　　　3. 如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险。

　　　　对比于悲观锁的这些问题，另一个更加有效的锁就是乐观锁。其实乐观锁就是：每次不加锁而是假设没有并发冲突而去完成某项操作，如果因为并发冲突失败就重试，直到成功为止。

　　乐观锁：

　　　　乐观锁（ Optimistic Locking ）在上文已经说过了，其实就是一种思想。相对悲观锁而言，乐观锁假设认为数据一般情况下不会产生并发冲突，所以在数据进行提交更新的时候，才会正式对数据是否产生并发冲突进行检测，如果发现并发冲突了，则让返回用户错误的信息，让用户决定如何去做。

　　　　上面提到的乐观锁的概念中其实已经阐述了它的具体实现细节：主要就是两个步骤：冲突检测和数据更新。其实现方式有一种比较典型的就是 Compare and Swap ( CAS )。

　　CAS：

　　　　CAS是乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。　　　

　　　　CAS 操作中包含三个操作数 —— 需要读写的内存位置（V）、进行比较的预期原值（A）和拟写入的新值(B)。如果内存位置V的值与预期原值A相匹配，那么处理器会自动将该位置值更新为新值B。否则处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。（在 CAS 的一些特殊情况下将仅返回 CAS 是否成功，而不提取当前值。）CAS 有效地说明了“ 我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。 ”这其实和乐观锁的冲突检查+数据更新的原理是一样的。

　　　　这里再强调一下，乐观锁是一种思想。CAS是这种思想的一种实现方式。

　　JAVA对CAS的支持：

　　　　在JDK1.5 中新增 java.util.concurrent (J.U.C)就是建立在CAS之上的。相对于对于 synchronized 这种阻塞算法，CAS是非阻塞算法的一种常见实现。所以J.U.C在性能上有了很大的提升。

　　　　以 java.util.concurrent 中的 AtomicInteger 为例，看一下在不使用锁的情况下是如何保证线程安全的。主要理解 getAndIncrement 方法，该方法的作用相当于 ++i 操作。

复制代码
 1 public class AtomicInteger extends Number implements java.io.Serializable {  
 2     private volatile int value; 
 3 
 4     public final int get() {  
 5         return value;  
 6     }  
 7 
 8     public final int getAndIncrement() {  
 9         for (;;) {  
10             int current = get();  
11             int next = current + 1;  
12             if (compareAndSet(current, next))  
13                 return current;  
14         }  
15     }  
16 
17     public final boolean compareAndSet(int expect, int update) {  
18         return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  
19     }  
20 }
复制代码
　　　　在没有锁的机制下,字段value要借助volatile原语，保证线程间的数据是可见性。这样在获取变量的值的时候才能直接读取。然后来看看 ++i 是怎么做到的。

　　　  getAndIncrement 采用了CAS操作，每次从内存中读取数据然后将此数据和 +1 后的结果进行CAS操作，如果成功就返回结果，否则重试直到成功为止。

　　　  而 compareAndSet 利用JNI（Java Native Interface）来完成CPU指令的操作：

1 public final boolean compareAndSet(int expect, int update) {   
2     return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
3 }　　　
　　　　其中unsafe.compareAndSwapInt(this, valueOffset, expect, update);类似如下逻辑：

1 if (this == expect) {
2     this = update
3     return true;
4 } else {
5     return false;
6 }
　　　　那么比较this == expect，替换this = update，compareAndSwapInt实现这两个步骤的原子性呢？ 参考CAS的原理

　　CAS原理：

　　　　CAS通过调用JNI的代码实现的。而compareAndSwapInt就是借助C来调用CPU底层指令实现的。

　　　　  下面从分析比较常用的CPU（intel x86）来解释CAS的实现原理。

 　　　  下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：

1 public final native boolean compareAndSwapInt(Object o, long offset,
2                                               int expected,
3                                               int x);
　　　　 可以看到这是个本地方法调用。这个本地方法在JDK中依次调用的C++代码为：

复制代码
 1 #define LOCK_IF_MP(mp) __asm cmp mp, 0  \
 2                        __asm je L0      \
 3                        __asm _emit 0xF0 \
 4                        __asm L0:
 5 
 6 inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
 7   // alternative for InterlockedCompareExchange
 8   int mp = os::is_MP();
 9   __asm {
10     mov edx, dest
11     mov ecx, exchange_value
12     mov eax, compare_value
13     LOCK_IF_MP(mp)
14     cmpxchg dword ptr [edx], ecx
15   }
16 }
复制代码
　　　　如上面源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

 

　　CAS缺点：

　　　　　1. ABA问题：

　　　　　　　比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且two进行了一些操作变成了B，然后two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后one操作成功。尽管线程one的CAS操作成功，但可能存在潜藏的问题。如下所示：

　　　　　　　

　　　　　　　现有一个用单向链表实现的堆栈，栈顶为A，这时线程T1已经知道A.next为B，然后希望用CAS将栈顶替换为B：

　　　　　　　　　　head.compareAndSet(A,B);

　　　　　　　　在T1执行上面这条指令之前，线程T2介入，将A、B出栈，再pushD、C、A，此时堆栈结构如下图，而对象B此时处于游离状态：

　　　　　　　

　　　　　　　此时轮到线程T1执行CAS操作，检测发现栈顶仍为A，所以CAS成功，栈顶变为B，但实际上B.next为null，所以此时的情况变为：

　　　　　　　

　　　　　　　其中堆栈中只有B一个元素，C和D组成的链表不再存在于堆栈中，平白无故就把C、D丢掉了。

　　　　　　　从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

复制代码
1 public boolean compareAndSet(
2                V      expectedReference,//预期引用
3 
4                V      newReference,//更新后的引用
5 
6               int    expectedStamp, //预期标志
7 
8               int    newStamp //更新后的标志
9 ) 
复制代码
　　　　　　　　实际应用代码：

1 private static AtomicStampedReference<Integer> atomicStampedRef = new AtomicStampedReference<Integer>(100, 0);
2 
3 ........
4 
5 atomicStampedRef.compareAndSet(100, 101, stamp, stamp + 1);
 

 

 　　　　2. 循环时间长开销大：

　　　　　　自旋CAS（不成功，就一直循环执行，直到成功）如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

　　　　

　　　　3. 只能保证一个共享变量的原子操作：

　　　　　　当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

 

　　CAS与Synchronized的使用情景：　　　

　　　　1、对于资源竞争较少（线程冲突较轻）的情况，使用synchronized同步锁进行线程阻塞和唤醒切换以及用户态内核态间的切换操作额外浪费消耗cpu资源；而CAS基于硬件实现，不需要进入内核，不需要切换线程，操作自旋几率较少，因此可以获得更高的性能。

　　　 2、对于资源竞争严重（线程冲突严重）的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源，效率低于synchronized。

　　　补充： synchronized在jdk1.6之后，已经改进优化。synchronized的底层实现主要依靠Lock-Free的队列，基本思路是自旋后阻塞，竞争切换后继续竞争锁，稍微牺牲了公平性，但获得了高吞吐量。在线程冲突较少的情况下，可以获得和CAS类似的性能；而线程冲突严重的情况下，性能远高于CAS。

 

　　concurrent包的实现：

　　　　由于java的CAS同时具有 volatile 读和volatile写的内存语义，因此Java线程之间的通信现在有了下面四种方式：

　　　　　　1. A线程写volatile变量，随后B线程读这个volatile变量。

　　　　　　2. A线程写volatile变量，随后B线程用CAS更新这个volatile变量。

　　　　　　3. A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。

　　　　　　4. A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

　　　　Java的CAS会使用现代处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机器，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。如果我们仔细分析concurrent包的源代码实现，会发现一个通用化的实现模式：

　　　　　　1. 首先，声明共享变量为volatile；　　

　　　　　　2. 然后，使用CAS的原子条件更新来实现线程之间的同步；

　　　　　　3. 同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

　　　　AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。从整体来看，concurrent包的实现示意图如下：

　　　　　　

 

　　JVM中的CAS（堆中对象的分配）：　

　　　　Java调用new object()会创建一个对象，这个对象会被分配到JVM的堆中。那么这个对象到底是怎么在堆中保存的呢？

　　　　首先，new object()执行的时候，这个对象需要多大的空间，其实是已经确定的，因为java中的各种数据类型，占用多大的空间都是固定的（对其原理不清楚的请自行Google）。那么接下来的工作就是在堆中找出那么一块空间用于存放这个对象。 
　　　　在单线程的情况下，一般有两种分配策略：

　　　　　　1. 指针碰撞：这种一般适用于内存是绝对规整的（内存是否规整取决于内存回收策略），分配空间的工作只是将指针像空闲内存一侧移动对象大小的距离即可。

　　　　　　2. 空闲列表：这种适用于内存非规整的情况，这种情况下JVM会维护一个内存列表，记录哪些内存区域是空闲的，大小是多少。给对象分配空间的时候去空闲列表里查询到合适的区域然后进行分配即可。

　　　　但是JVM不可能一直在单线程状态下运行，那样效率太差了。由于再给一个对象分配内存的时候不是原子性的操作，至少需要以下几步：查找空闲列表、分配内存、修改空闲列表等等，这是不安全的。解决并发时的安全问题也有两种策略：

　　　　　　1. CAS：实际上虚拟机采用CAS配合上失败重试的方式保证更新操作的原子性，原理和上面讲的一样。

　　　　　　2. TLAB：如果使用CAS其实对性能还是会有影响的，所以JVM又提出了一种更高级的优化策略：每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲区（TLAB），线程内部需要分配内存时直接在TLAB上分配就行，避免了线程冲突。只有当缓冲区的内存用光需要重新分配内存的时候才会进行CAS操作分配更大的内存空间。 
　　　　　　虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来进行配置（jdk5及以后的版本默认是启用TLAB的）。
      
[1] https://www.cnblogs.com/qjjazry/p/6581568.html
