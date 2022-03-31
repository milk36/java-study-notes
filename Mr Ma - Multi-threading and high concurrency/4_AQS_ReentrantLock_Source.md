AQS (AbstractQueuedSynchronizer) ReentrantLock 源码解读
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
  1. 如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，<big> 这个机制AQS是用 <u>CLH</u> 队列锁实现的，即将暂时获取不到锁的线程加入到队列中。</big>
* 关键点
  1. state (volatile) 共享锁数据 (加锁状态/重入次数 )
  1. 双向队列 存储等待线程
    
      CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。
      
      AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配。  
  1. LockSupport `park unpark  ` 暂停/唤醒 相应线程
  1. VarHandle(变量句柄) 替换 `unsafe.objectFieldOffset`(JDK 1.8) 偏移操作 实现对相应字段的CAS操作

      原理类似反射操作
      
      `AbstractQueuedSynchronizer` 中的VarHandle
      ```java
      private static final VarHandle STATE;
      private static final VarHandle HEAD;
      private static final VarHandle TAIL;

      static{
        try{
          MethodHandles.Lookup l = MethodHandles.lookup();
          STATE = l.findVarHandle(AbstractQueuedSynchronizer.class, "state", int.class);
          HEAD = l.findVarHandle(AbstractQueuedSynchronizer.class, "head", Node.class);
          TAIL = l.findVarHandle(AbstractQueuedSynchronizer.class, "tail", Node.class);
        } catch (ReflectiveOperationException e) {
          throw new ExceptionInInitializerError(e);
        }
      }
      ```
      `AbstractQueuedSynchronizer.Node` 中的VarHandle
      ```java    
      private static final VarHandle NEXT;
      private static final VarHandle PREV;
      private static final VarHandle THREAD;
      private static final VarHandle WAITSTATUS;

      static{
        try{
          MethodHandles.Lookup l = MethodHandles.lookup();
          NEXT = l.findVarHandle(Node.class, "next", Node.class);
          PREV = l.findVarHandle(Node.class, "prev", Node.class);
          THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
          WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
        } catch (ReflectiveOperationException e) {
          throw new ExceptionInInitializerError(e);
        }
      }
      ```
### ReentrantLock # NonfairSync  非公平锁
* NonfairSync 类图

  ![NonfairSync](https://i.imgur.com/xT5mn0b.png "NonfairSync")
* CLH Queue 双向队列

  ![CLH Queue](https://i.imgur.com/dryE3hI.png "CLH Queue")
#### ReentrantLock **acquire** 尝试获取锁 或 进入停止状态  
* `AQS.acquire` 获取锁流程

  ![Acquire Lock](https://i.imgur.com/K6CiFtJ.png "Acquire Lock")  
  ```java
  public final void acquire(int arg) {
      if (!tryAcquire(arg) && //尝试获取锁
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //添加到等待队列尾 在队列中等待 并尝试获取锁
          selfInterrupt();
  }
  ```
* Node nextWaiter 标识 独占 / 共享  
  ```java
  //标记，表示节点在共享模式下等待
  static final Node SHARED = new Node();
  //标记, 表示节点在独占模式下等待
  static final Node EXCLUSIVE = null;
  //上面两个标记赋值给nextWaiter
  Node nextWaiter;
  ```
* 公平/非公平 锁 原理
  * **公平锁** `AQS.hasQueuedPredecessors()` 

      * hasQueuedPredecessors 查询队列中是否有处于等待状态的

        是否有排队的前置任务(线程) 
      
        查询等待队列里面是否有任何线程正在等待获取的时间长于 当前线程。        
        ```java
        public final boolean hasQueuedPredecessors() {
            Node h, s;// s = h.next
            if ((h = head) != null) {
                if ((s = h.next) == null || s.waitStatus > 0) {//如果 next 为空 或者等待状态为 取消, 就跳到下一个 继续判断
                    s = null; // traverse in case of concurrent cancellation
                    for (Node p = tail; p != h && p != null; p = p.prev) {
                        if (p.waitStatus <= 0)
                            s = p;
                    }
                }
                if (s != null && s.thread != Thread.currentThread())//确认下一个等待任务非空 并不是当前线程 
                    return true;
            }
            return false;
        }
        ```        
      > ### 会先检查队列中是否有正在等待的线程 , 如果没有才去申请锁, 否则进入等待队列
      * `ReentrantLock.FairSync.tryAcquire` 公平尝试获取锁
        ```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() && //判断队列是否有前置的等待线程 这里是公平锁与非公平锁的区别
                    compareAndSetState(0, acquires)) { //CAS操作尝试获取锁 state 从 0 -> 1
                    setExclusiveOwnerThread(current);//设置拥有独占线程 该线程获取锁资源
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {// 判断是否为同一线程 锁重入操作
                int nextc = c + acquires;//重入锁 state + 1
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
        ```       
  * **非公平锁** `ReentrantLock.Sync.nonfairTryAcquire()`  
  
    > ### 直接申请锁, 申请失败才会通过CAS操作添加到等待队列 **tail**
    * `ReentrantLock.Sync.nonfairTryAcquire` 非公平尝试获取锁
      ```java
      final boolean nonfairTryAcquire(int acquires) {
          final Thread current = Thread.currentThread();
          int c = getState();//获取同步状态
          if (c == 0) {//无锁 则尝试获取锁
              if (compareAndSetState(0, acquires)) {// CAS操作尝试获取锁 state 从 0 -> 1
                  setExclusiveOwnerThread(current);//设置拥有独占线程 该线程获取锁资源
                  return true;
              }
          }
          else if (current == getExclusiveOwnerThread()) {// 判断是否为同一线程 锁重入操作
              int nextc = c + acquires;//重入锁 state + 1
              if (nextc < 0) // overflow
                  throw new Error("Maximum lock count exceeded");
              setState(nextc);
              return true;
          }
          return false;
      }
      ``` 
* `AQS.addWaiter` 添加到等待队列
  ```java
  addWaiter(Node.EXCLUSIVE) //Node.EXCLUSIVE = null
  ...
  private Node addWaiter(Node mode) {
      Node node = new Node(mode);//创建新节点 绑定当前线程

      for (;;) {
          Node oldTail = tail;//尾队列
          if (oldTail != null) {
              node.setPrevRelaxed(oldTail);//把新节点的 prev (前节点) 设置成旧尾节点 
              if (compareAndSetTail(oldTail, node)) {// CAS方式添加到 队列尾部 如果失败则重来一遍
                  oldTail.next = node;
                  return node;
              }
          } else {
              initializeSyncQueue();//初始同步队列 ; head = tail = new Noed()
          }
      }
  }
  ```
* `AQS.acquireQueued` 尝试从队列中获取锁 排队

  `tryAcquire()` 且将节点 thread 设置到 `exclusiveOwnerThread` => 独占线程

  `setHead()` head 头节点拥有锁资源 thread prev 都被置空 

  `Thread.interrupted()` 查询当前线程是否被打断过, **并重置打断标记** 自动复位成false 
  ```java
  final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();//获取前节点 双向队列特性
                if (p == head && tryAcquire(arg)) {//如果前节点为头节点 就尝试获取锁 可能前节点以释放锁
                    setHead(node); //设置当前节点为头节点 prev thread 都置空 ; 因为此时这个节点的线程已经拥有锁
                    p.next = null; // help GC
                    return interrupted;
                }
                //获取失败后停止当前线程 设置 prev.waitStatus = SIGNAL (-1) 
                //这样以来 prev释放锁的时候就知道是否有后续的等待任务需要被唤醒
                if (shouldParkAfterFailedAcquire(p, node))
                    //内部调用 LockSupport.park(this) 停止当前线程 
                    //唤醒后返回 Thread.interrupted() 测试当前线程是否已中断 ; 而且该方法可以清除线程的中断状态
                    interrupted |= parkAndCheckInterrupt(); 
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }
  ```
  waitStatus 对应的几个状态
  ```java
  //表示线程已取消
  static final int CANCELLED =  1;
  //表示后续线程等待unpark信号
  static final int SIGNAL    = -1;
  //表示线程在等待条件
  static final int CONDITION = -2;
  //表示下一次获取的共享应该无条件传播 ???
  static final int PROPAGATE = -3;
  ```  
#### ReentrantLock **release** 释放锁资源
* `AQS.release` 流程

  ![Release Lock](https://i.imgur.com/7kinOPs.png "Release Lock")
* `ReentrantLock.unlock`
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
                  unparkSuccessor(h);//启动后续线程 唤醒 head.next 节点
              return true;
          }
          return false;
      }
  ``` 
* `ReentrantLock.tryRelease` 尝试释放锁
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
* `AQS.unparkSuccessor`  启动后续节点
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