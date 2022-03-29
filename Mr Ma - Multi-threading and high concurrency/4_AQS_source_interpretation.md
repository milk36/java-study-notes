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
  1. head 头节点拥有锁资源 thread 被置空 ; 且将节点 thread 设置到 exclusiveOwnerThread
* 公平/非公平 锁 原理
  1. AQS.hasQueuedPredecessors() `是否有排队的前置任务(线程)`

      查询等待队列里面是否有任何线程正在等待获取的时间长于 当前线程。

      * 公平锁 会先检查队列中是否有正在等待的线程 , 如果没有才去申请锁, 否则进入等待队列

      * 非公平锁 直接申请锁, 申请失败才会通过CAS操作添加到等待队列 **tail**
  1. ReentrantLock.Sync.nonfairTryAcquire()    
### ReentrantLock # NonfairSync  非公平锁
* NonfairSync 类图

  ![NonfairSync](https://i.imgur.com/xT5mn0b.png "NonfairSync")
* CLH Queue 双向队列

  ![CLH Queue](https://i.imgur.com/dryE3hI.png "CLH Queue")
#### ReentrantLock **acquire** 尝试获取锁 或 进入停止状态  
* acquire 获取锁流程

  ![Acquire Lock](https://i.imgur.com/K6CiFtJ.png "Acquire Lock")  
* nonfairTryAcquire 非公平尝试获取锁
  ```java
  final boolean nonfairTryAcquire(int acquires) {
      final Thread current = Thread.currentThread();
      int c = getState();//获取同步状态
      if (c == 0) {//无锁 则尝试获取锁
          if (compareAndSetState(0, acquires)) {// CAS操作尝试获取锁
              setExclusiveOwnerThread(current);//设置拥有独占线程 该线程获取锁资源
              return true;
          }
      }
      else if (current == getExclusiveOwnerThread()) {// 判断是否为同一线程 锁重入操作
          int nextc = c + acquires;//重入锁状态 + 1
          if (nextc < 0) // overflow
              throw new Error("Maximum lock count exceeded");
          setState(nextc);
          return true;
      }
      return false;
  }
  ```
* addWaiter 添加到等待队列
  ```java
  addWaiter(Node.EXCLUSIVE) //Node.EXCLUSIVE = null
  ...
  private Node addWaiter(Node mode) {
      Node node = new Node(mode);//创建新节点 绑定当前线程

      for (;;) {
          Node oldTail = tail;
          if (oldTail != null) {
              node.setPrevRelaxed(oldTail);//将旧尾节点添加到 新节点前面
              if (compareAndSetTail(oldTail, node)) {// CAS方式添加到 队列尾部
                  oldTail.next = node;
                  return node;
              }
          } else {
              initializeSyncQueue();//初始同步队列 ; head = tail = new Noed()
          }
      }
  }
  ```
* acquireQueued 
  ```java
  final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();//获取前节点
                if (p == head && tryAcquire(arg)) {//如果前节点为头节点 就尝试获取锁
                    setHead(node); //设置当前节点为头节点 prev thread 都置空 ; 因为此时这个节点的线程已经拥有锁
                    p.next = null; // help GC
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node))//获取失败后停顿 设置前节点 waitStatus = SIGNAL (-1)
                    interrupted |= parkAndCheckInterrupt(); //内部调用 LockSupport.park(this) 停止当前线程
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }
  ```  
#### ReentrantLock **release** 释放锁资源
* release 流程

  ![Release Lock](https://i.imgur.com/7kinOPs.png "Release Lock")
* ReentrantLock.unlock
  ```java
  public void unlock() {
        sync.release(1);
  }
  -> Sync->AQS.release(1)
    public final boolean release(int arg) {
          if (tryRelease(arg)) {
              Node h = head;
              //头节点非空 并等待状态不为0 
              //注意: 尾节点 waitStatus = 0
              if (h != null && h.waitStatus != 0)
                  unparkSuccessor(h);
              return true;
          }
          return false;
      }
  ``` 
* tryRelease 尝试释放锁
  ```java
  protected final boolean tryRelease(int releases) {
      int c = getState() - releases;//锁状态 - 1 ; 重入锁逻辑
      if (Thread.currentThread() != getExclusiveOwnerThread()) //判断释放锁线程是否为 独占线程
          throw new IllegalMonitorStateException();
      boolean free = false;
      if (c == 0) {// 锁状态清零
          free = true;
          setExclusiveOwnerThread(null);//置空独占线程
      }
      setState(c);//设置锁状态
      return free;
  }
  ```     
* unparkSuccessor  启动后续节点
  ```java
  private void unparkSuccessor(Node node) {        
        int ws = node.waitStatus;//节点等待状态
        if (ws < 0)//只有取消状态 CANCELLED = 1
            node.compareAndSetWaitStatus(ws, 0);

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒/启动 后续线程
    }
  ```