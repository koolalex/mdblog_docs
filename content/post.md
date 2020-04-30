
# Java多线程
## Java线程5个状态
**NRRBD(新可运阻死)**
- N:New 新建，如：Thread t = new MyThread();
- R:Runnable,当调用线程对象的start()方法（t.start();），线程即进入就绪状态,只是说明此线程已经做好了准备，随时等待CPU调度执行，并不是说执行了t.start()此线程立即就会执行；
- R:Running,只是说明此线程已经做好了准备，随时等待CPU调度执行，并不是说执行了t.start()此线程立即就会执行；（）
- B:Blocked,阻塞状态，由于某种原因，暂时放弃对cpu使用权，直到就绪才有机会被cpu调用进入运行状态。主要有三种原因阻塞：
    + 等待阻塞 - 调用了wait
    + 同步阻塞 - 获取synchronized同步锁失败，进入同步阻塞
    + 其他阻塞 - 通过sleep或join或发出IO请求
- D:Dead,死亡状态，运行完毕或异常退出。
- W:Waitting， 调用wait或者join但是没有超时时

- 线程对象的start()方法和run()方法的不同
    + start()得到的是异步的执行结果；
    + run()得到的是**同步**的执行结果；

## 线程安全
1) 相对线程安全如Vector，如果有个线程在遍历某个Vector、有个线程同时在add这个Vector，99%的情况下都会出现ConcurrentModificationException，也就是fail-fast机制。
2）变量不可变不需要考虑线程安全问题

## Volatile 
保证变量可见性，但无法保证其原子性，在读写变量的时候操作的是主存而不是Cpu cache。
- 高并发环境下，线程多谢变量的时候往往是读写CPU cache变量值，有可能不是最新的。
- 结合CAS实现原子性，比如java.util.concurrent.atomic包下的AtomicInteger.

## 锁机制
### 互斥锁 Mutex
- 同步块(synchronized block),synchronized锁住的是对象
- 对象锁(object.lock)
- 可重入锁(ReentrantLock), 已经获得了锁的线程再次要求同一个锁，可重入锁运行这种行为

### 信号量(Semaphore)
> 公平和非公平
作用是限制某段代码块的并发数,构造函数可以传入一个int型整数n，表示某段代码最多只有n个线程可以访问，如果超出了n，那么请等待，等到某个线程执行完毕这段代码块，下一个线程再进入。

### 乐观锁(CAS-Compare and Swap，即比较-替换)
**ABA问题，无锁堆栈**
> 取出1，+1得到2，compare and swap(1,2) 只有当原来的值还是1的时候，才将它替换为2，否则重新读取再+1。CAS是一个原子操作。

### AQS - AbstractQueuedSychronizer，翻译过来应该是抽象队列同步器
> 用于控制排队、唤醒的同用抽象工具
AQS涉及的几个概念：
- state（volatile int state）状态：这是AbstractQueuedSynchronizer里一个万能的属性，具体是什么含义，全看你的使用方式，比如在CountDownLatch里，它代表了当前到达后正在等待的线程数，在Semaphore里，它则表示当前进去后正在运行的线程数
- CAS: AQS里大量用了CAS（Compare and Swap）操作来修改state的值
- LockSupport: AQS里用了大量的LockSupport的park()和unpark()方法，来挂起和唤醒线程
- 同步队列和条件队列：sync queue and condition queue，弄清楚这两个队列的关系，AQS也就弄懂大半
- 公平和非公平：有线程竞争，就有公平和非公平的问题。锁释放的时候，刚好有个线程过来获取锁，但这时候线程等待队列里也有线程在等待，到底是给排队时间最久的线程呢(公平)，还是允许新来的线程参与竞争（不公平）？

如果说java.util.concurrent的基础是CAS的话，那么AQS就是整个Java并发包的核心了，ReentrantLock、CountDownLatch、Semaphore、CountDownLatch、CyclicBarrier等等都用到了它。AQS实际上以双向队列的形式连接所有的Entry，比方说ReentrantLock，所有等待的线程都被放在一个Entry中并连成双向队列，前面一个线程使用ReentrantLock好了，则双向队列实际上的第一个Entry开始运行。
AQS定义了对双向队列所有的操作，而只开放了tryLock和tryRelease方法给开发者使用，开发者可以根据自己的实现重写tryLock和tryRelease方法，以实现自己的并发功能。

## Java的并发工具
**同步容器和并发容器**

### 非线程安全
ArrayList、LinkedList、HashMap (List、Set、Queue、Map等)

### 同步容器
Vector
Stack
HashTable
Collections.synchronized 方法生成的容器

### 并发容器
ConcurrentHashMap --> HashMap
CopyOnWriteArrayList --> ArrayList
CopyOnWriteArraySet -->HashSet
ConcurrentSkipListMap --> TreeMap
ConcurrentSkipListSet --> TreeSet
ConcurrentLinkedQueue --> Queue
LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue -->BlockingQueue

### 线程池
> 使用线程池可以避免频繁创建和销毁同时，可以通过控制线程数量
ThreadPoolExecutor

### 线程通讯方法
> 通讯的桥梁一般是共享的那个object，也就是下面的一些方法都是通过共享的object去call的

- wait()
该方法用来将当前线程置入休眠状态，直到接到通知或被中断为止。在调用wait（）之前，线程必须要获得该对象的对象级别锁【所以要用那个对象去call】

- notify
该方法也要在同步方法或同步块中调用，即在调用前，线程也必须要获得该对象的对象级别锁

- wait(long), wait(long,int)
等待超时

- thread.yield()
Thread.yield()不是用来作线程通讯的,这个方法的作用是让当前线程放弃CPU，然后和其他线程一起竞争CPU，这个方法用的很少，因为只有非常清楚程序要干啥的程序员才会直接进行CPU级别的资源调度。

- Semaphore
信号量，控制并发数，举例蹲坑，10个坑满了其他人就得等着

- CountDownLatch
为0时唤醒，距离参加饭局，10个人到齐开饭，没到一个人`CountDownLatch-1`代表到了一个人

- BlockingQueue

## sleep和wait区别
sleep方法和wait方法都可以用来放弃CPU一定的时间，不同点在于如果线程持有某个对象的监视器，sleep方法不会放弃这个对象的监视器，wait方法会放弃这个对象的监视器

## ThreadLocal
简单说ThreadLocal就是一种以空间换时间的做法，在每个Thread里面维护了一个以开地址法实现的ThreadLocal.ThreadLocalMap，把数据进行隔离，数据不共享，自然就没有线程安全方面的问题了

## synchronized和ReentrantLock的区别
synchronized是和if、else、for、while一样的关键字，ReentrantLock是类，这是二者的本质区别。既然ReentrantLock是类，那么它就提供了比synchronized更多更灵活的特性，可以被继承、可以有方法、可以有各种各样的类变量，ReentrantLock比synchronized的扩展性体现在几点上：
（1）ReentrantLock可以对获取锁的等待时间进行设置，这样就避免了死锁
（2）ReentrantLock可以获取各种锁的信息
（3）ReentrantLock可以灵活地实现多路通知

## ReadWriteLock
ReadWriteLock是一个读写锁接口，ReentrantReadWriteLock是ReadWriteLock接口的一个具体实现，实现了读写的分离，读锁是共享的，写锁是独占的，读和读之间不会互斥，读和写、写和读、写和写之间才会互斥，提升了读写的性能。

## FutureTask
这个其实前面有提到过，FutureTask表示一个异步运算的任务。FutureTask里面可以传入一个Callable的具体实现类，可以对这个异步运算的任务的结果进行等待获取、判断是否已经完成、取消任务等操作。当然，由于FutureTask也是Runnable接口的实现类，所以FutureTask也可以放入线程池中。

## 不可变对象
不可变对象在多线程中意味着不需要担心其会改变而做出控制

## 自旋
就是空转，用在一些非常短的非阻塞等待场景中，因为线程阻塞涉及到用户态和内核态切换的问题；

## Java内存模型
（1）Java内存模型将内存分为了主内存和工作内存。类的状态，也就是类之间共享的变量，是存储在主内存中的，每次Java线程用到这些主内存中的变量的时候，会读一次主内存中的变量，并让这些内存在自己的工作内存中有一份拷贝，运行自己线程代码的时候，用到这些变量，操作的都是自己工作内存中的那一份。在线程代码执行完毕之后，会将最新的值更新到主内存中去
（2）定义了几个原子操作，用于操作主内存和工作内存中的变量
（3）定义了volatile变量的使用规则
（4）happens-before，即先行发生原则，定义了操作A必然先行发生于操作B的一些规则，比如在同一个线程内控制流前面的代码一定先行发生于控制流后面的代码、一个释放锁unlock的动作一定先行发生于后面对于同一个锁进行锁定lock的动作等等，只要符合这些规则，则不需要额外做同步措施，如果某段代码不符合所有的happens-before规则，则这段代码一定是线程非安全的

## 线程终止
> 判断线程停止的方法this.isInterrupted()
1) 抛异常
2) interrupt- sleep的时候，被主函数调用了interrupt()方法停止该线程，那么会跳过sleep后的代码，进入catch代码
3) 正常return
4) Thread.yield()方法来放弃当前CPU资源

## 线程优先级
高优先级的程序总是大部分先执行完，但并不代表高优先级的程序全部先执行完

## 守护线程
守护线程有陪伴的意思，当进程中不存在非守护线程后，守护线程自动销毁。
aThreadObject.setDaemon(true); //通过该方法来将线程对象设置为守护线程

## 线程池调优
- ThreadPoolTaskExecutor
```java
<!-- 通用的 TaskExecutor，适合执行一些小功能，并发低，效率高的任务-->  
<bean id="commonTaskExecutor"  
    class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">  
    <!-- 线程名称前缀 -->  
    <property name="threadNamePrefix" value="commonTaskExecutor" />  
    <!-- 核心线程数，默认为1 -->  
    <property name="corePoolSize" value="10" />  
    <!-- 最大线程数，默认为Integer.MAX_VALUE -->  
    <property name="maxPoolSize" value="200" />  
    <!-- 队列最大长度，一般需要设置值>=notifyScheduledMainExecutor.maxNum；默认为Integer.MAX_VALUE -->  
    <property name="queueCapacity" value="1000" />  
    <!-- 线程池维护线程所允许的空闲时间，默认为60s -->  
    <property name="keepAliveSeconds" value="300" />  
    <!-- 容器停止时是否等待job执行完，默认为false -->  
    <property name="waitForTasksToCompleteOnShutdown" value="true" />  
    <!-- 容器停止时等待job执行的秒数，默认为0 -->  
    <property name="awaitTerminationSeconds" value="30" />  
    <!-- 线程池对拒绝任务（无线程可用）的处理策略，目前只支持AbortPolicy、CallerRunsPolicy；默认为后者 -->  
    <property name="rejectedExecutionHandler">  
        <!-- AbortPolicy:直接抛出java.util.concurrent.RejectedExecutionException异常 -->  
        <!-- CallerRunsPolicy:主线程直接执行该任务，执行完之后尝试添加下一个任务到线程池中，可以有效降低向线程池内添加任务的速度 -->  
        <!-- DiscardOldestPolicy:抛弃旧的任务、暂不支持；会导致被丢弃的任务无法再次被执行 -->  
        <!-- DiscardPolicy:抛弃当前任务、暂不支持；会导致被丢弃的任务无法再次被执行 -->  
        <bean class="java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy" />  
    </property>  
</bean>  
```
- 假如是业务时间长集中在IO操作上，也就是IO密集型的任务，因为IO操作并不占用CPU，所以不要让所有的CPU闲下来，可以适当加大线程池中的线程数目，让CPU处理更多的业务。一个经验值为:"最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目"
- 假如是业务时间长集中在计算操作上，也就是计算密集型任务，这个就没办法了，和1一样，线程池中的线程数设置得少一些，减少线程上下文的切换。

## 多线程编程模式
> 常用的多线程设计模式包括：Future模式、Master-Worker模式、Guarded Suspeionsion模式、不变模式和生产者-消费者模式等。

### Future模式
FUTURE模式的目的在于在执行耗时任务（如远程RPC调用）的时候，主线程不需要等待其返回结果而可以继续执行其他逻辑，等到主线程拿到返回值时再对其进行处理。

### Master-Worker模式
一个Master线程，调用多个worker线程，并且协调它们共同执行一项任务。

### Guarded Suspension模式
与生产者-消费者模型的区别，Guarded Suspension模式并不限制缓冲区（即请求队列）的长度。
涉及的重点知识是wait和notify（ALL）方法，这两个方法用来使当前线程等待或者唤醒等待的线程。

---

# Spring&SpringBoot
- @Component 作用就相当于 XML配置的作用
- @Bean 需要在配置类中使用，即类上需要加上@Configuration注解,@Bean可以将第三方库手动装配到Spring容器中
```java
@Configuration
public class WebSocketConfig {
    @Bean
    public Student student(){
        return new Student();
    }
}
```
