AQS (AbstractQueuedSynchronizer) CountDownLatch 源码解读
===
### CountDownLatch await 等待获取锁
![await](https://i.imgur.com/bbIiU4r.png "await")
* `CountDownLatch.await`
  ```java
  public void await() throws InterruptedException {
      sync.acquireSharedInterruptibly(1);
  }
  ```
* `AQS.acquireSharedInterruptibly` 以共享方式请求锁 可以被打断
  ```java
  public final void acquireSharedInterruptibly(int arg)
      throws InterruptedException {
  if (Thread.interrupted())//判断当前线程是否中断
      throw new InterruptedException();
  if (tryAcquireShared(arg) < 0)//尝试获取共享锁
      doAcquireSharedInterruptibly(arg);//以共享可中断模式获取 或停止
  }
  ```  
* `CountDownLatch.Sync.tryAcquireShared` 尝试获取共享锁

  返回 1 可以获取锁 ; 返回 -1 阻塞
  ```java
  protected int tryAcquireShared(int acquires) {
      return (getState() == 0) ? 1 : -1;//获取失败 返回 -1
  }
  ```
* `AQS.doAcquireSharedInterruptibly` 
  
  以共享可中断模式获取
  
  失败则进入等待状态

  这里与`ReentrantLock`不同的是 head 在被唤醒后 会向后传递 将后面的 netx 继续唤醒
  ```java
  private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);//以共享的方式添加节点到 队列尾部
    try {
        for (;;) {
            final Node p = node.predecessor();//获取当前 prev
            if (p == head) {//prev 为 head
                int r = tryAcquireShared(arg);//尝试获取共享锁 state == 0 ?
                if (r >= 0) {//获取成功
                    setHeadAndPropagate(node, r);//设置head 并传导锁释放
                    p.next = null; // help GC
                    return;
                }
            }
            //获取失败后停止当前线程 设置 prev.waitStatus = SIGNAL (-1)
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
  }
  ```
* `AQS.setHeadAndPropagate`  

  成功唤醒共享锁后 设置队列的头，并检查后续队列是否有在共享模式下等待的 Node 
  ```java
  private void setHeadAndPropagate(Node node, int propagate) {
      Node h = head; // 记录旧 head 用于下面检查
      setHead(node);//将当前Node 设置为Head
      
      if (propagate > 0 || h == null || h.waitStatus < 0 ||
          (h = head) == null || h.waitStatus < 0) {
          Node s = node.next;
          if (s == null || s.isShared())//next 为空 或者为共享模式锁 唤醒共享锁
              doReleaseShared();
      }
  }
  ```
* `AQS.doReleaseShared`  

  成功唤醒共享锁后 处理后续等待节点唤醒
  ```java
  private void doReleaseShared() {      
      for (;;) {
          Node h = head;
          if (h != null && h != tail) {//判断是否有后续节点需要唤醒
              int ws = h.waitStatus;
              if (ws == Node.SIGNAL) {
                  if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))//CAS 将head.waitStatus = 0
                      continue;            
                  unparkSuccessor(h);//唤醒 head.next节点
              }
              else if (ws == 0 &&
                       !h.compareAndSetWaitStatus(0, Node.PROPAGATE))//PROPAGATE == -3 以便能传导后续节点唤醒操作
                  continue;                
          }
          if (h == head)//没有后续节点等待                   
              break;
      }
  }
  ```
### CountDownLatch countDown 减少门闩计数释放锁  
![countDown](https://i.imgur.com/nph9YA6.png "countDown")
* `CountDownLatch.countDown`
  ```java
  public void countDown() {
      sync.releaseShared(1);//计数器减一
  }
  ```
* `AQS.releaseShared`
  ```java
  public final boolean releaseShared(int arg) {
      if (tryReleaseShared(arg)) {//计数器减一 并判断能否释放锁
          doReleaseShared();//唤醒共享锁 并处理后续等待节点唤醒
          return true;
      }
      return false;
  }
  ```
* `CountDownLatch.Sync.tryReleaseShared`

  计数器减一 
  
  返回能否释放锁
  ```java
  protected boolean tryReleaseShared(int releases) {
      // Decrement count; signal when transition to zero
      //因为释放锁可能同时被多个线程调用 需要循环CAS操作 直到成功
      for (;;) {
          int c = getState();
          if (c == 0)
              return false;
          int nextc = c - 1;
          if (compareAndSetState(c, nextc))//CAS 设置 state - 1 
              return nextc == 0;
      }
  }
  ```