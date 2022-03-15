CAS Atomic /新线程同步机制
===
### CAS 
* CAS -> Compare And Swap 比较替换
  ```
  cas(V, Expected, NewValue)
  - if V == E
    V = NewValue
    otherwise try again or fail
  ```
  1. V 修改目标值
  1. Expected 期望的当前值,预期值
  1. NewValue 要设定的新值

  只有 V 目标值 == Expected 预期值, 才能让 V 目标值 = NewValue

  其他情况重试/失败
* ABA问题

  V 目标值 在操作之前已经被修改过多个版本, 可能已经从 1 到 10,然后又从10 到 1 

  当进行简单CAS操作时无法感知这个中间版本变化的过程, 解决方法 在其 V 目标值上加一个版本号

  java中可以使用 `AtomicStampedReference` 或者 `AtomicMarkableReference`

  ```java
  static AtomicStampedReference<Order> orderRef = new AtomicStampedReference<>(new Order(), 0);
  orderRef.compareAndSet(old, o, stamp, stamp + 1);
  ...
  static AtomicMarkableReference<Order> orderRef = new AtomicMarkableReference<>(new Order(), false);
  orderRef.compareAndSet(old, o, false, true);
  ```
* Unsafe
  
  Atomic内部操作都依赖于Unsafe

  Unsafe = C/C++ 的指针操作  
  * 直接操作内存

    allocateMemory putXX freeMemory pageSize
  * 直接生成类实例

    allocateInstance
  * 直接操作类或实例变量

    objectFieldOffset getInt getObject
  * CAS相关操作

    weakCompareAndSetObject Int Long
  * 模仿c/c++操作内存

    c -> malloc free c++ -> new delete
### 新线程同步机制
* AtomicLong

  CAS 自旋锁 

  比 `synchronized`快的原因: sync可能会升级到重量级锁, 所以效率偏低
* LongAdder

  CAS 自旋锁 分段锁

  线程数小了 循环数少了 未必有优势
* ReentrantLock

  可重入锁 与`synchronized `使用方式类似但功能更多 可以替代 `synchronized`

  * 注意:加锁 释放锁 操作需要在 `try{}finally{}` 代码快中执行
    ```java
    Lock lock = new ReentrantLock();
    try{
        lock.lock();
    }catch(InterruptedExcption e){
        e.printStackTrace();
    }finally{
        lock.unlock();
    }
    ```
  * tryLock() 进行尝试加锁
    ```java
    try{
      boolean locked = lock.tryLock(5,TimeUnit.SECONDS);//尝试加锁,等待5秒,返回是否加锁成功
      System.out.println(locked);
    }finally{
      lock.unlock();
    }
    ```
  * lockInterruptibly()  可以被打断的加锁 这是比sync要灵活的地方
    ```java
    Thread t2 = new Thread(()->{
      try{
        lock.lockInterruptibly();//对interrupt()方法做出响应
      }finally{
        lock.unlock();
      }
    }
    t2.interrupt();//中断线程操作
    ```
  * 公平锁 非公平锁设置

    公平锁 即需要在队列中排队等待 每次申请锁资源需要先进入等待队列
    
    非公平锁 即没有等待队列 大家一拥而上看谁速度快谁就拥有锁
    ```java
    Lock lock = new ReentrantLock(true);//参数为true表示为公平锁, 默认为非公平锁
    ``` 
* CountDownLatch 门栓 计数器 类似Thread.join()

  countDown await 两个方法配合使用
  ```java
  CountDownLatch latch = new CountDownLatch(100);
  for(100){
    new Thread(()->{
      latch.countDown();//计数器减一
    });
  }
  try{
    latch.await();//等待结束
  }catch(InterruptedException e){
    e.printStackTrace();
  }
  ```
* CyclicBarriar 循环栅栏  

  什么时候人满了 就把栅栏推倒 然后全部放出去; 之后栅栏又重新起来 再来人 满了之后又继续
  ```java
  CyclicBarrier barrier = new CyclicBarrier(20, () -> System.out.println("满人, 发车"));

  for(100){
    new Thread(()->{
      try{
        barrier.await();
      } catch (InterruptedException e) {
        e.printStackTrace();
      } catch (BrokenBarrierException e) {
        e.printStackTrace();
      }
    });
  }
  ```
