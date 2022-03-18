面试解题
===
### 面试题1

  实现一个容器，提供两个方法add、size，写两个线程：

  线程1，添加10个元素到容器中

  线程2，实时监控元素个数，当个数到5个时，线程2给出提示并结束

  > volatile一定要尽量去修饰普通的值，不要去修饰引用值，因为volatile修饰引用类型，这个引用对象指向的是另外一个new出来的对象对象，如果这个对象里边的成员变量的值改变了，是无法观察到的

* 测试数据结构
  ```java
      static abstract class TestList {

          private List<Integer> list = Lists.newArrayList();

          void add(int i) {
              list.add(i);
          }

          int size() {
              return list.size();
          }
      }
  ```
  
* Wait Notify 解题版本

  ```java
  TestList list = new TestList();
  Object lock = new Object();
  new Thread(()->{
    synchronized(lock){
      //if(size()==5){
        try{
          //启动后就等待
          lock.wait();
          System.out.println("size:"+list.size());
          //通知T2继续执行
          lock.notify();
        }catch(InterruptedException e){
          e.printStackTrace();
        }
      //}
      
    }
  },"T1").start();

  new Thread(()->{
    synchronized(lock){
      for(int i=0 ;i<10;i++){
        list.add(i);
        System.out.println("add:"+i);
        if(list.size() == 5){
          //通知T1执行
          lock.notify();
          try{
            //让出当前锁资源 否则T1需要等待T2执行完毕才能获得锁            
            lock.wait();
          }catch(InterruptedException e){
            e.printStackTrace();
          }
        }

      }
    }
  },"T2").start();
  ```
* CountDownLatch  门闩解题实现 

  需要初始化两把门闩 配合

  ```java
  TestList list = new TestList();
  CountDownLatch latchA = new CountDownLatch(1);
  CountDownLatch latchB = new CountDownLatch(1);
  new Thread(()->{
      try{
          //启动后就等待
          latchA.await();
          System.out.println("size:"+list.size());
          //通知T2继续执行后续打印
          latchB.countDown();
      }catch(InterruptedException e){
        e.printStackTrace();
      }
      
  },"T1").start();

  new Thread(()->{
      for(int i=0 ;i<10;i++){
        list.add(i);
        System.out.println("add:"+i);
        if(list.size() == 5){
          //通知T1执行
          latchA.countDown();
          try{
            //等待T1执行打印输出 得到通知后 T2再继续执行输出            
            latchB.await();
          }catch(InterruptedException e){
            e.printStackTrace();
          }
        }
    }
  },"T2").start();
  ```
* LockSupport `park unpark` 解题实现
  ```java
  static Thread t1,t2;
  ...

  TestList list = new TestList();  
  t1 = new Thread(()->{
      //启动后就等待
      LockSupport.park();
      System.out.println("size:"+list.size());
      //通知T2继续执行后续打印
      LockSupport.unpark(t2);
  },"T1");

  t2 = new Thread(()->{
      for(int i=0 ;i<10;i++){
        list.add(i);
        System.out.println("add:"+i);
        if(list.size() == 5){
        //通知T1执行
        LockSupport.unpark(t1);
        //等待T1执行打印输出 得到通知后 T2再继续执行输出            
        LockSupport.park();          
        }
    }
  },"T2");

  t1.start();
  t2.start();
  ```
* Semaphore 信号量 解题实现 
  ```java
  static Thread t1,t2;
  ...

  TestList list = new TestList();
  Semaphore s1 = new Semaphore(1);
  Semaphore s2 = new Semaphore(1);
  t1 = new Thread(()->{
    try{
      //获得s1许可
      s1.acquire();
      t2.start();
      for(int i = 0 ;i<10 ;i++){
        list.add(i);
        System.out.println("add: "+i);
        if(list.size()==5){
          //释放s1信号
          s1.release();
          //等待t2释放s2信号 完成后续打印
          s2.acquire();
        }
      }
    }catch(InterruptedException e){
      e.printStackTrace();
    }
  });
  t2 = new Thread(()->{
    try{
      //启动后立即获得s2许可证
      s2.acquire();
      System.out.println("waiting for s1");      
      //阻塞等待s1许可
      s1.acquire();//启动后这里会阻塞
      System.out.println("size:"+list.size());
      //释放s2信号
      s2.release();
    }catch(InterruptedException e){
      e.printStackTrace();
    }
  });
  t1.start();
  ```
### 面试题2
 写一个固定容量同步容器，拥有`put`和`get`方法，以及`getCount`方法，
 
 能够支持**2个生产者线程**以及**10个消费者线程**的阻塞调用。  

* `synchronized wait() notifyAll()` 解题思路(常规 需记忆)
  ```java
  private static final int MAX = 10;
  //LinkedList采用链表结构 对于频繁插入和删除数据效率比ArrayList高
  private LinkedList<T> lists = new LinkedList();
  private int count =0;

  public synchronized int getCount(){ return count; }

  public synchronized void put(T t){
    //判断容器是否已满 则进入等待
    // 使用while是防止在下次唤醒时count被其他生产线程修改(有新增)
    while(getCount()==MAX){
      try{
        //当前线程进入等待
        this.wait();
      }catch(InterruptedException e){
        e.printStackTrace();
      }
    }
    lists.addLast(t);
    ++count;
    this.notifyAll();//唤醒所有等待线程 
  }

  public synchronized T get(){
    //判断容器是否为空 则进入等待
    //使用while是防止唤醒后count被其他消费线程修改(减少)
    while(getCount() == 0){
      try{
        this.wait();
      }catch(InterruptedException e){
        e.printStackTrace();
      }
    }
    T t = lists.removeFirst();
    --count;
    this.notifyAll();
    return t;
  }
  ```
* ReentrantLock `Condition` 解题思路 

  > 使用两个 `Condition` 条件分别对 comsumer (消费) producer (生产 )线程加锁 提高加锁性能

  ```java
  private static final int MAX = 10;
  private LinkedList<T> lists = new LinkedList();
  private Lock lock = new ReentrantLock();
  private Condition proC = lock.newCondition();
  private Condition conC = lock.newCondition();

  private int count = 0;

  public int getCount(){ 
    return count;
  }

  public void put(T t){
    try{
      lock.lock();
      while(getCount() == MAX){
        //当前生产线程等待 producer
        proC.await();
      }
      ++count;
      lists.addLast(t);
      //通知唤醒所有消费线程 comsumer
      conC.signalAll();
    }catch(InterruptedException e){
      e.printStackTrace();
    }finally{
      lock.unlock();
    }
  }

  public T get(){
    T t = null;
    try{
      lock.lock();
      while(getCount() == 0){
        //当前消费线程等待 comsumer
        conC.await();
      }
      --count;
      t = lists.removeFirst();
      //通知唤醒所有生产线程 producer
      proC.signalAll();
    }catch(InterruptedException e){
      e.printStackTrace();
    }finally{
      lock.unlock();
    }
    return t;
  }
  ```

