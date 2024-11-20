
## Quartz集群增强版\_02\.任务轮询及优化



> 转载请著名出处 [https://github.com/funnyzpc/p/18555665](https://github.com)


### 开源地址 [https://github.com/funnyzpc/quartz](https://github.com):[樱花宇宙官网](https://yzygzn.com)


    任务轮询的主要工作是按固定频度(时间`5s`)去执行项表捞未来5s内将要执行的任务，轮询这些任务待到执行时间点时将任务扔到线程池去执行。
    看似很简单其实也有各种各样的问题存在，这里不表 请往下看 \~
    另外，任务轮询的主要逻辑在：`QuartzSchedulerThread` ，读者有兴趣可以看看源码\~


### 轮询窗口内的任务


情况是这样子的，先看图:


* ![](https://img2024.cnblogs.com/blog/1161789/202411/1161789-20241119215406663-939273055.png)


    假使，现在有一个任务 `task1` ,他的执行时间是每`2秒`执行一次，但是记录执行项里面只会存一个`下一次执行时间`(`next_fire_time`)，落在上图就是`2s`的位置，这样在每`5秒`轮询一次的时候会漏掉一次执行(`4s`的位置)
    这个问题解决起来其实很简单，就是每次从`db`获取到的执行项再做计算，除当前次外 `5s` 内的执行的时间全部计算出来，这其中尤其要注意的是同一个时间项在当前次内有多次执行的一定要有`顺序`！
    在后续会有循环等待，但在特殊情况下，用上图说：由于同批次其他任务存在延迟（假如延迟大于等于`2s`） ，这时候`4s`时的这个任务可能早于 `2s` 时的任务执行，同时又由于 `4s` 时的任务的 参照时间是 `2s` 时的任务的时间(`pre_fire_time`) 😂（可能很难理解吧，建议看看后续`update`语句）
    在被扔到线程池前，数据库由于 `2s` 时的任务并没有执行，数据库里面存的是 `0s` 时的任务配置，从而就会导致`4s`时的任务不会执行（因为他竞争不到锁）(`2s`任务参照的是`0s`时的任务 `4s`参照的是`2s`时的任务)，这是很严重的问题； 如果任务是有序的且计算出来的`4s`时的任务总是排在 `2s` 时的任务之后，即使其他任务存在延迟，也会相应保证后续时间点儿任务正常执行，很大程度避免了任务丢失\~


### 获取执行权限（获取锁）


    因为存在集群并发的问题，所以一个任务同一时间必须只由一个`节点`来执行，同时也为了保证执行`顺序` 所以在任务被丢到线程池之前需要在数据库 做一个 `UPDATE` 的竞争操作，具体`SQL`语句如下：



```


|  | UPDATE |
| --- | --- |
|  | QRTZ_EXECUTE SET |
|  | PREV_FIRE_TIME =? , |
|  | NEXT_FIRE_TIME = ?, |
|  | TIME_TRIGGERED =?, |
|  | STATE =?, |
|  | HOST_IP =?, |
|  | HOST_NAME =?, |
|  | END_TIME =? |
|  | WHERE ID = ? |
|  | AND STATE = ? -- old STATE |
|  | AND PREV_FIRE_TIME = ? -- old PREV_FIRE_TIME |
|  | AND NEXT_FIRE_TIME = ? -- old NEXT_FIRE_TIME |
|  |  |


```

可以看到，必须是被更新记录必须是要对齐 `STATE`、 `PREV_FIRE_TIME`、 `NEXT_FIRE_TIME` 才可更新\~


### 使用动态线程池


    `Quartz` 一般使用的是 `SimpleThreadPool` 作为其任务的线程池，既然简单必然是： 内部使用固定线程处理
    一开始，我是准备就着源码做部分改动来着，后来发现没这边简单，原 `Quartz` 在获取锁的
时候会使用线程本地变量（`ThreadLocal`） 缓存 `执行线程` 以做并发控制，后来不得已将逻辑大部分推翻做重构，这是很大的变化; 现在，对于 `Quartz集群增强版` 来说,不再有 `ThreadLocal` 的困扰, 只需关注自身 执行线程池配置的实现逻辑即可，这就有了 `MeeThreadPool` 不仅有了线程分配控制也有了队列，这是一大变化，现在你可以使用 `MeeThreadPool` 也可以继续使用 `SimpleThreadPool` ～


    这是 `MeeThreadPool` 的主要逻辑：



```


|  |  |
| --- | --- |
|  | protected void createWorkerThreads(final int createCount) { |
|  | int cct = this.count = createCount<1? Runtime.getRuntime().availableProcessors() :createCount; |
|  | final MyThreadFactory myThreadFactory = new MyThreadFactory(this.getThreadNamePrefix(), this); |
|  | this.poolExecutor = new ThreadPoolExecutor(cct<=4?2:cct-2,cct+2,6L, TimeUnit.SECONDS, new LinkedBlockingDeque(cct+2),myThreadFactory); |
|  | } |
|  |  |
|  | private final class MyThreadFactory implements ThreadFactory { |
|  | final String threadPrefix ;//= schedulerInstanceName + "_QRTZ_"; |
|  | final MeeThreadPool meeThreadPool; |
|  | private final AtomicInteger threadNumber = new AtomicInteger(1); |
|  | public MyThreadFactory(final String threadPrefix,final MeeThreadPool meeThreadPool) { |
|  | this.threadPrefix = threadPrefix; |
|  | this.meeThreadPool = meeThreadPool; |
|  | } |
|  |  |
|  | @Override |
|  | public Thread newThread(Runnable r) { |
|  | WorkerThread wth = new WorkerThread( |
|  | meeThreadPool, |
|  | threadGroup, |
|  | threadPrefix + ((threadNumber.get())==count?threadNumber.getAndSet(1):threadNumber.getAndIncrement()), |
|  | getThreadPriority(), |
|  | isMakeThreadsDaemons(), |
|  | r); |
|  | if (isThreadsInheritContextClassLoaderOfInitializingThread()) { |
|  | wth.setContextClassLoader(Thread.currentThread().getContextClassLoader()); |
|  | } |
|  | return wth; |
|  | } |
|  | } |


```

    伸缩性以及可用性有了大大的提高，需要提一嘴的是 如果使用 `ThreadPoolExecutor` 开发 `Quartz` 线程池一定要注意：


* 核心线程打满之后 `task` 一定是先进入队列
* 队列满了之后才会依次创建线程直至最大线程数
* 一定要注意是否有线程被打满后的异常拒绝处理策略，如果不希望出现异常拒绝 那是否要考虑在提交任务之前判断线程池是否被打满
* 开发完成一定要进行广泛的测试，以符合预期
* 


### 轮询超时/执行超时问题


在`JVM`执行`GC`或者`DB`或者`网络`存在`故障`，亦或是主机`性能存在瓶颈`，或是线程池被打满 ... 等等，均会出现超时的问题，对于此类问题本 `Quartz集群增强版` 做了以下优化：


* 做了容忍度偏移，让任务不拘泥于几毫秒的差异提前执行



```


|  | //1.时间偏移(6毫秒) |
| --- | --- |
|  | long ww = executeList.size()-1000<0 ? 4L : ((executeList.size()-1000L)/2000L)+4L ; |
|  | ww= Math.min(ww, 8L); |
|  | while( !executeList.isEmpty() && (System.currentTimeMillis()-now)<=LOOP_INTERVAL ){ |
|  | long _et  = System.currentTimeMillis(); |
|  | QrtzExecute ce = null; // executeList.get(0); |
|  | for( int i = 0;i< executeList.size();i++ ){ |
|  | QrtzExecute el = executeList.get(i); |
|  | // 这是要马上执行的任务 |
|  | if( el.getNextFireTime()-_et <= ww){ |
|  | ce=el; |
|  | break; |
|  | } |
|  | if(i==0){ |
|  | ce=el; |
|  | continue; // 如果执行列表长度为一，则会直接进入下面sleep等待 |
|  | } |
|  | // 总是获取最近时间呢个 |
|  | if( el.getNextFireTime() <= ce.getNextFireTime() ){ |
|  | ce = el; |
|  | } |
|  | } |
|  | executeList.remove(ce); // 一定要移除，否则无法退出while循环!!! |
|  | // 延迟 |
|  | long w = 0; |
|  | if((w = (ce.getNextFireTime()-System.currentTimeMillis()-ww)) >0 ){ |
|  | try { |
|  | Thread.sleep(w); |
|  | }catch (Exception e){ |
|  | } |
|  | } |
|  | // 后续代码略 |
|  | } |


```

* 对于任务轮询，保证轮询时间间隔的同时也做了偏移修正



```


|  | // 延迟 |
| --- | --- |
|  | long st = 0; |
|  | if((st = (LOOP_INTERVAL-(System.currentTimeMillis()-now)-2)) >0 ){ |
|  | try { |
|  | Thread.sleep(st); |
|  | } catch (InterruptedException e) { |
|  | e.printStackTrace(); |
|  | } |
|  | } |
|  | if( st<-10 && st%5==0 ){ |
|  | LOG.error("当前次任务轮询超时:"+st); |
|  | } |
|  | // 防止因轮询超时的必要手段 |
|  | now = st<-1000? |
|  | System.currentTimeMillis()/1000*1000 : |
|  | System.currentTimeMillis()+(st<-10?st:0); |


```

* 对于事实的延迟做了任务修正


这个修正主要依赖于 `ClusterMisfireHandler` 的轮询处理，以保证后续中断的任务能及时恢复\~


    对于偏移，需要解释下： `偏移`是对于整个循环而言的，任务循环一次是 `5s` ，由于写表或任务提交可能造成整个循环会有 `几毫秒`或`几十毫秒`的偏差 ，这是`向后偏移`，如果任务提前执行完成 则整个循环可能不足 `5s` 这是`向前偏差` \~
不管是向前还是向后都是需要避免的\~


## 最后


    为了更清楚的了解 `Quartz集群增强版` 建议过一遍结构图：


* ![](https://img2024.cnblogs.com/blog/1161789/202411/1161789-20241119215428192-1663059381.png)
* ![](https://img2024.cnblogs.com/blog/1161789/202411/1161789-20241119215433119-415800458.png)


