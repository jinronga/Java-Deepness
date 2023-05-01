# ☺ ThreadPoolExecutor 源码分析





#### 1、[线程池](https://so.csdn.net/so/search?q=%E7%BA%BF%E7%A8%8B%E6%B1%A0\&spm=1001.2101.3001.7020)状态： <a href="#cfekt" id="cfekt"></a>

五种状态：

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>



## 1、RUNNING <a href="#hcmjz" id="hcmjz"></a>

状态说明：在RUNNING状态下，线程池可以接收新的任务和执行已添加的任务。

线程池的初始化状态是RUNNING。换句话说，线程池被一旦被创建（比如调用Executors.newFixedThreadPool()或者使用ThreadPoolExecutor进行创建），就处于RUNNING状态，并且线程池中的任务数为0！线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理。

## 2、 SHUTDOWN <a href="#cgqzm" id="cgqzm"></a>

状态说明：线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务。

当一个线程池调用shutdown()方法时，线程池由RUNNING -> SHUTDOWN。\


## 3、STOP <a href="#lm1fn" id="lm1fn"></a>

状态说明：线程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在执行的任务。

\
调用线程池的shutdownNow()方法的时候，线程池由(RUNNING或者SHUTDOWN ) -> STOP。

## 4、TIDYING <a href="#qfvd6" id="qfvd6"></a>

\
状态说明：当所有的任务已终止，记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。

terminated()方法在ThreadPoolExecutor类中是空的，没有任何实现。



<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>



若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重写terminated()函数来实现。 当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 当线程池在STOP状态下，线程池中执行的任务为空时，会由STOP -> TIDYING。

## 5、 TERMINATED <a href="#lgemo" id="lgemo"></a>

\
状态说明：当钩子函数terminated()被执行完成之后，线程池彻底终止，就变成TERMINATED状态。

\
线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。

#### &#x20;总结

状态之间状态，和各个状态之间的的切换总结





<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

* 线程池的shutdown() 方法，将线程池由 RUNNING（运行状态）转换为 SHUTDOWN状态
* 线程池的shutdownNow()方法，将线程池由RUNNING 或 SHUTDOWN 状态转换为 STOP 状态。

注：SHUTDOWN 状态 和 STOP 状态 先会转变为 TIDYING 状态，最终都会变为 TERMINATED

\


### ThreadPoolExecutor构造函数： <a href="#avtpl" id="avtpl"></a>

ThreadPoolExecutor继承自AbstractExecutorService，而AbstractExecutorService实现了ExecutorService接口。





<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }




    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }


   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
```



2.1）线程池工作原理：

* corePoolSize ：线程池中核心线程数的最大值
* maximumPoolSize ：线程池中能拥有最多线程数
* workQueue：用于缓存任务的阻塞队列

当调用线程池execute() 方法添加一个任务时，线程池会做如下判断：

1. 如果有空闲线程，则直接执行该任务；
2. 如果没有空闲线程，且当前运行的线程数少于corePoolSize，则创建新的线程执行该任务；
3. 如果没有空闲线程，且当前的线程数等于corePoolSize，同时阻塞队列未满，则将任务入队列，而不添加新的线程；
4. 如果没有空闲线程，且阻塞队列已满，同时池中的线程数小于maximumPoolSize ，则创建新的线程执行任务；
5. 如果没有空闲线程，且阻塞队列已满，同时池中的线程数等于maximumPoolSize ，则根据构造函数中的 handler 指定的策略来拒绝新的任务。







<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

）KeepAliveTime：

* keepAliveTime ：表示空闲线程的存活时间
* TimeUnit unit ：表示keepAliveTime的单位

当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小。

**注：**如果线程池设置了allowCoreThreadTimeout参数为true（默认false），那么当空闲线程超过keepaliveTime后直接停掉。（不会判断线程数是否大于corePoolSize）即：最终线程数会变为0。

2.3）workQueue 任务队列：

* workQueue ：它决定了缓存任务的排队策略

ThreadPoolExecutor线程池推荐了三种等待队列，它们是：SynchronousQueue 、LinkedBlockingQueue 和 ArrayBlockingQueue。

1）有界队列：

* SynchronousQueue ：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于 阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法 Executors.newCachedThreadPool 使用了这个队列。
* ArrayBlockingQueue：一个由数组支持的有界阻塞队列。此队列按 FIFO（先进先出）原则对元素进行排序。一旦创建了这样的缓存区，就不能再增加其容量。试图向已满队列中放入元素会导致操作受阻塞；试图从空队列中提取元素将导致类似阻塞。

2）无界队列：

* LinkedBlockingQueue：基于链表结构的无界阻塞队列，它可以指定容量也可以不指定容量（实际上任何无限容量的队列/栈都是有容量的，这个容量就是Integer.MAX\_VALUE）
* PriorityBlockingQueue：是一个按照优先级进行内部元素排序的无界阻塞队列。队列中的元素必须实现 Comparable 接口，这样才能通过实现compareTo()方法进行排序。优先级最高的元素将始终排在队列的头部；PriorityBlockingQueue 不会保证优先级一样的元素的排序。

[\
](https://blog.csdn.net/liuxiao723846/article/details/108026782)

**注意**：keepAliveTime和maximumPoolSize及BlockingQueue的类型均有关系。如果BlockingQueue是无界的，那么永远不会触发maximumPoolSize，自然keepAliveTime也就没有了意义。[\
](https://blog.csdn.net/liuxiao723846/article/details/108026782)2.4）threadFactory：

* threadFactory ：指定创建线程的工厂。（可以不指定）

如果不指定线程工厂时，ThreadPoolExecutor 会使用ThreadPoolExecutor.defaultThreadFactory 创建线程。默认工厂创建的线程：同属于相同的线程组，具有同为 Thread.NORM\_PRIORITY 的优先级，以及名为 “pool-XXX-thread-” 的线程名（XXX为创建线程时顺序序号），且创建的线程都是非守护进程。

2.5）handler 拒绝策略：

* handler ：表示当 workQueue 已满，且池中的线程数达到 maximumPoolSize 时，线程池拒绝添加新任务时采取的策略。（可以不指定）

如果不指定线程工厂时，ThreadPoolExecutor 会使用ThreadPoolExecutor.defaultThreadFactory 创建线程。默认工厂创建的线程：同属于相同的线程组，具有同为 Thread.NORM\_PRIORITY 的优先级，以及名为 “pool-XXX-thread-” 的线程名（XXX为创建线程时顺序序号），且创建的线程都是非守护进程。

2.5）handler 拒绝策略：

* handler ：表示当 workQueue 已满，且池中的线程数达到 maximumPoolSize 时，线程池拒绝添加新任务时采取的策略。（可以不指定）

**1.AbortPolicy**\
AbortPolicy

ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。

这是线程池默认的拒绝策略，在任务不能再提交的时候，抛出异常，及时反馈程序运行状态。如果是比较关键的业务，推荐使用此拒绝策略，这样子在系统不能承载更大的并发量的时候，能够及时的通过异常发现。

**2.DiscardPolicy**\
ThreadPoolExecutor.DiscardPolicy：丢弃任务，但是不抛出异常。如果线程队列已满，则后续提交的任务都会被丢弃，且是静默丢弃。

使用此策略，可能会使我们无法发现系统的异常状态。建议是一些无关紧要的业务采用此策略。例如。

**3.DiscardOldestPolicy**

ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新提交被拒绝的任务。

此拒绝策略，是一种喜新厌旧的拒绝策略。是否要采用此种拒绝策略，还得根据实际业务是否允许丢弃老任务来认真衡量。

**4.CallerRunsPolicy**

ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务\
\
最科学的的还是 AbortPolicy 提供的处理方式：抛出异常，由开发人员进行处理。

#### Executors类： <a href="#ravm8" id="ravm8"></a>

Executors类的底层实现便是ThreadPoolExecutor！ Executors 工厂方法有：

* Executors.newCachedThreadPool()：无界线程池，可以进行自动线程回收
* Executors.newFixedThreadPool(int)：固定大小线程池
* Executors.newSingleThreadExecutor()：单个后台线程

它们均为大多数使用场景预定义了设置。不过在阿里java文档中说明，尽量不要用该类创建线程池。

### 线程池相关接口介绍： <a href="#u36ih" id="u36ih"></a>

#### 1、ExecutorService接口： <a href="#z9sls" id="z9sls"></a>

该接口是真正的线程池接口。上面的ThreadPoolExecutor以及下面的ScheduledThreadPoolExecutor都是该接口的实现类。改接口常用方法：

* Future\<?> submit(Runnable task)：提交Runnable任务到线程池，返回Future对象，由于Runnable没有返回值，也就是说调用Future对象get()方法返回null；
* \<T> Future\<T> submit(Callable\<T> task)：提交Callable任务到线程池，返回Future对象，调用Future对象get()方法可以获取Callable的返回值；
* \<T> Future\<T> submit(Runnable task，T result)：提交Runnable任务到线程池，返回Future对象，调用Future对象get()方法可以获取Runnable的参数值；
* invokeAll(collection of tasks)/invokeAll(collection of tasks, long timeout, TimeUnit unit)：invokeAll会按照任务集合中的顺序将所有的Future添加到返回的集合中，该方法是一个阻塞的方法。只有当所有的任务都执行完毕时，或者调用线程被中断，又或者超出指定时限时，invokeAll方法才会返回。当invokeAll返回之后每个任务要么返回，要么取消，此时客户端可以调用get/isCancelled来判断具体是什么情况。
* invokeAny(collection of tasks)/invokeAny(collection of tasks, long timeout, TimeUnit unit)：阻塞的方法，不会返回 Future 对象，而是返回集合中某一个Callable 对象的结果，而且无法保证调用之后返回的结果是哪一个 Callable，如果一个任务运行完毕或者抛出异常，方法会取消其它的 Callable 的执行。和invokeAll区别是只要有一个任务执行完了，就把结果返回，并取消其他未执行完的任务；同样，也带有超时功能；
* shutdown()：在完成已提交的任务后关闭服务，不再接受新任；
* shutdownNow()：停止所有正在执行的任务并关闭服务；
* isTerminated()：测试是否所有任务都执行完毕了；
* isShutdown()：测试是否该ExecutorService已被关闭。



















