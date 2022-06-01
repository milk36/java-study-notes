思考题
===
## 1. A线程正在执行一个对象中的同步方法，B线程是否可以同时执行同一个对象中的非同步方法？
> 可以 一个线程在访问一个对象的同步方法时，另一个线程可以同时访问这个对象的非同步方法。
```java
public class SyncMethodOtherTest {
  public synchronized void m1() {//这里的锁对象为this
    HelperUtil.SleepHelper.secondsSleep(1);
    System.out.println("m1");
  }

  public void m2() {
    System.out.println("m2");
  }

  public static void main(String[] args) {
    SyncMethodOtherTest test = new SyncMethodOtherTest();
    Thread t1 = new Thread(test::m1);
    Thread t2 = new Thread(test::m2);
    t1.start();
    t2.start();
  }
}
```
## 2. 同上，B线程是否可以同时执行同一个对象中的另一个同步方法？
> 不行 
## 3. 线程抛出异常会释放锁吗？
> 会释放锁
```java
public class ErrorSyncMethodTest {
  public synchronized void m1() {
    try {
      System.out.println("m1:" + DateUtil.now());
      HelperUtil.SleepHelper.secondsSleep(5);
      int i = 1 / 0;
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  public synchronized void m2() {
    System.out.println("m2:" + DateUtil.now());
  }

  public static void main(String[] args) {
    ErrorSyncMethodTest test = new ErrorSyncMethodTest();
    Thread t1 = new Thread(test::m1);
    t1.start();

    Thread t2 = new Thread(test::m2);
    t2.start();
  }
}
```
## 4. volatile和synchronized区别？
* volatile 保证可见性和有序性
* synchronized 保证原子性和可见性
## 5. 写一个程序，证明AtomXXX类比synchronized更高效
* CAS适合场景 执行时间短 线程数少
* synchroinzed 适合场景 执行时间长 线程数多
```java
public class AtomVSSyncTest {

  private static final int TASK_COUNT = 100;
  private static final int MAX_COUNT = 100000;

  public static void main(String[] args) {
    ExecutorService threadPool = Executors.newFixedThreadPool(10);
    AtomicInteger atomicInteger = new AtomicInteger(0);
    CountDownLatch atomLatch = new CountDownLatch(TASK_COUNT);
    System.out.println("start AtomicInteger count test");
    TimeInterval timer = DateUtil.timer();
    for (int i = 0; i <TASK_COUNT ; i++) {
      threadPool.execute(()->{
        for (int j = 0; j < MAX_COUNT; j++) {
          atomicInteger.incrementAndGet();
        }
        atomLatch.countDown();
      });
    }
    try {
      atomLatch.await();
      System.out.println("AtomicInteger timer:" + timer.interval());
    } catch (InterruptedException ex) {
      ex.printStackTrace();
    }

    Object lock = new Object();
    CountDownLatch downLatch = new CountDownLatch(TASK_COUNT);
    System.out.println("start sync count test");
    timer = DateUtil.timer();
    for (int i = 0; i <TASK_COUNT ; i++) {
      threadPool.execute(()->{
        for (int j = 0; j <MAX_COUNT ; j++) {
          synchronized (lock) {
            atomicInteger.incrementAndGet();
          }
        }
        downLatch.countDown();
      });
    }
    try {
      downLatch.await();
      System.out.println("Sync timer:" + timer.interval());
    } catch (InterruptedException ex) {
      ex.printStackTrace();
    }
    threadPool.shutdown();
  }
}
```
## 6. AtomXXX类可以保证可见性吗？请写一个程序来证明
```java
public class AtomicVisibleTest {
  public static void main(String[] args) {
    AtomicBoolean ab = new AtomicBoolean(true);
    new Thread(()->{
      System.out.println("start observer");
      while (ab.get()) {

      }
      System.out.println("end observer");
    }).start();
    HelperUtil.SleepHelper.secondsSleep(2);
    System.out.println("stop observer");
    ab.getAndSet(false);
    HelperUtil.SleepHelper.secondsSleep(1);
    System.out.println("test finish");
  }
}
```
## 7. 写一个程序证明AtomXXX类的多个方法并不构成原子性
```java
public class AtomicMethodNotAtomTest {
  AtomicInteger num = new AtomicInteger(0);
  public void m1() {
    for (int i = 0; i < 100000; i++) {
      if (num.get() < 10000) {//Atomic类的方法没有锁操作
        num.incrementAndGet();//因为没有锁 上面这个条件在某一瞬间会失效 最终结果会 > 10000
      }
    }
  }
  public static void main(String[] args) throws Exception{
    AtomicMethodNotAtomTest test = new AtomicMethodNotAtomTest();
    List<Thread> list = new ArrayList();
    for (int i = 0; i < RuntimeUtil.getProcessorCount(); i++) {
      Thread t = new Thread(test::m1);
      list.add(t);
    }
    list.forEach(Thread::start);
    for (Thread thread : list) {
      thread.join();
    }
    System.out.println(test.num);
  }
}
```
## 8. 写一个程序模拟死锁
```java
public class DeathLockTest {
  public static void main(String[] args) {
    Object lock1 = new Object();
    Object lock2 = new Object();
    new Thread(()->{
      System.out.println("start t1");
      synchronized (lock1) {
        HelperUtil.SleepHelper.secondsSleep(1);
        synchronized (lock2) {
          System.out.println("t1 succeed");
        }
      }
    },"Thread-1").start();

    new Thread(()->{
      System.out.println("start t1");
      synchronized (lock2) {
        HelperUtil.SleepHelper.secondsSleep(1);
        synchronized (lock1) {
          System.out.println("t2 succeed");
        }
      }
    },"Thread-2").start();
  }
}
```
## 9. 写一个程序，在main线程中启动100个线程，100个线程完成后，主线程打印“完成”，使用join()和countdownlatch都可以完成，请比较异同。
## 10. 一个高效的游戏服务器应该如何设计架构？