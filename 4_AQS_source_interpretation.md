AQS (AbstractQueuedSynchronizer) 源码解读
===
### 阅读源码的原则
1. 跑不起来的不读
1. 解决问题就好 -- 目的性
1. 一条线索到底
1. 无关细节略过
### AQS源码
* [AQS源码解读参考](https://pdai.tech/md/java/thread/java-thread-x-lock-AbstractQueuedSynchronizer.html)
* AQS核心思想
  1. AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。
  1. 如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用 <u>CLH</u> 队列锁实现的，即将暂时获取不到锁的线程加入到队列中。
* 关键点
  1. sate 共享锁数据 (加锁状态/重入次数 )
  1. 双向队列 存储等待线程
  1. LockSupport park unpark
  1. CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配。
* 公平/非公平 锁 原理
  1. AQS.hasQueuedPredecessors() `是否有排队的前置任务(线程)`

      查询等待队列里面是否有任何线程正在等待获取的时间长于 当前线程。

      公平锁 会先检查队列中是否有正在等待的线程 , 如果没有才去申请锁, 否则进入等待队列

      非公平锁 直接申请锁, 申请失败才会通过CAS操作添加到等待队列 **tail**
  1. ReentrantLock.Sync.nonfairTryAcquire()    