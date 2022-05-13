ThreadPoolExecute源码
===
## 源码解析
### 常量/变量解释
```java
// 1. `ctl`，可以看做一个int类型的数字，高3位表示线程池状态，低29位表示worker数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 2. `COUNT_BITS`，`Integer.SIZE`为32，所以`COUNT_BITS`为29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 3. `CAPACITY`，线程池允许的最大线程数。1左移29位，然后减1，即为 2^29 - 1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 4. 线程池有5种状态，按大小排序如下：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
private static final int RUNNING    = -1 << COUNT_BITS; //高3位: 111
private static final int SHUTDOWN   =  0 << COUNT_BITS; //高3位: 000 
private static final int STOP       =  1 << COUNT_BITS; //高3位: 001
private static final int TIDYING    =  2 << COUNT_BITS; //高3位: 010
private static final int TERMINATED =  3 << COUNT_BITS; //高3位: 011

// Packing and unpacking ctl
// 5. `runStateOf()`，获取线程池状态，通过按位与操作，低29位将全部变成0
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 6. `workerCountOf()`，获取线程池worker数量，通过按位与操作，高3位将全部变成0
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 7. `ctlOf()`，根据线程池状态和线程池worker数量，生成ctl值
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */
// 8. `runStateLessThan()`，线程池状态小于xx
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
// 9. `runStateAtLeast()`，线程池状态大于等于xx
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
```
* ctl 值

  高3位表示线程池状态，低29位表示worker数量
* 线程池的5中状态
  1. RUNNING：正常运行的；
  1. SHUTDOWN：调用了shutdown方法了进入了shutdown状态；
  1. STOP：调用了shutdownnow马上让他停止；
  1. TIDYING：调用了shutdown然后这个线程也执行完了，现在正在整理的这个过程叫TIDYING；
  1. TERMINATED：整个线程全部结束；
### 构造函数
```java
public ThreadPoolExecutor(int corePoolSize,
                        int maximumPoolSize,
                        long keepAliveTime,
                        TimeUnit unit,
                        BlockingQueue<Runnable> workQueue,
                        ThreadFactory threadFactory,
                        RejectedExecutionHandler handler) {
  // 基本类型参数校验
  if (corePoolSize < 0 ||
      maximumPoolSize <= 0 ||
      maximumPoolSize < corePoolSize ||
      keepAliveTime < 0)
      throw new IllegalArgumentException();
  // 空指针校验
  if (workQueue == null || threadFactory == null || handler == null)
      throw new NullPointerException();
  this.corePoolSize = corePoolSize;
  this.maximumPoolSize = maximumPoolSize;
  this.workQueue = workQueue;
  // 根据传入参数`unit`和`keepAliveTime`，将存活时间转换为纳秒存到变量`keepAliveTime `中
  this.keepAliveTime = unit.toNanos(keepAliveTime);
  this.threadFactory = threadFactory;
  this.handler = handler;
}
```
### execute 函数 提交执行task
```java
public void execute(Runnable command) {
  if (command == null)
      throw new NullPointerException();
  int c = ctl.get();
  // worker数量比核心线程数小，直接创建worker执行任务
  if (workerCountOf(c) < corePoolSize) {
      if (addWorker(command, true))
          return;
      c = ctl.get();
  }
  // worker数量超过核心线程数，任务直接进入队列
  if (isRunning(c) && workQueue.offer(command)) {
      int recheck = ctl.get();
      // 线程池状态不是RUNNING状态，说明执行过shutdown命令，需要对新加入的任务执行reject()操作。
      // 这儿为什么需要recheck，是因为任务入队列前后，线程池的状态可能会发生变化。
      if (! isRunning(recheck) && remove(command))
          reject(command);
      // 这儿为什么需要判断0值，主要是在线程池构造方法中，核心线程数允许为0
      else if (workerCountOf(recheck) == 0)
          addWorker(null, false);
  }
  // 如果线程池不是运行状态，或者任务进入队列失败，则尝试创建worker执行任务。
  // 这儿有3点需要注意：
  // 1. 线程池不是运行状态时，addWorker内部会判断线程池状态
  // 2. addWorker第2个参数表示是否创建核心线程
  // 3. addWorker返回false，则说明任务执行失败，需要执行reject操作
  else if (!addWorker(command, false))
      reject(command);
}
```
* 分3步逻辑执行:
  1. 核心线程数不够 `workerCount < corePoolSize`, 先启动核心线程
  1. 核心线程数够了添加到任务队列 `workQueue.offer(command)`
  1. 核心线程和任务队列都满 启动非核心线程工作 `addWorker(command, false)`
* `reject(command) ` 拒绝/饱和 函数调用
### addWorker 函数
```java
private boolean addWorker(Runnable firstTask, boolean core) {
  retry:
  // 外层自旋
  for (;;) {
      int c = ctl.get();
      int rs = runStateOf(c);
      // 这个条件写得比较难懂，对其进行了调整，和下面的条件等价
      // (rs > SHUTDOWN) || 
      // (rs == SHUTDOWN && firstTask != null) || 
      // (rs == SHUTDOWN && workQueue.isEmpty())
      // 1. 线程池状态大于SHUTDOWN时，直接返回false SHUTDOWN = 0
      // 2. 线程池状态等于SHUTDOWN，且firstTask不为null，直接返回false
      // 3. 线程池状态等于SHUTDOWN，且队列为空，直接返回false
      // Check if queue empty only if necessary.
      if (rs >= SHUTDOWN &&
          ! (rs == SHUTDOWN &&
             firstTask == null &&
             ! workQueue.isEmpty()))
          return false;
      // 内层自旋
      for (;;) {
          int wc = workerCountOf(c);
          // worker数量超过容量，直接返回false
          if (wc >= CAPACITY ||
              wc >= (core ? corePoolSize : maximumPoolSize))
              return false;
          // 使用CAS的方式增加worker数量。
          // 若增加成功，则直接跳出外层循环进入到第二部分
          if (compareAndIncrementWorkerCount(c))
              break retry;
          c = ctl.get();  // Re-read ctl
          // 线程池状态发生变化，对外层循环进行自旋
          if (runStateOf(c) != rs)
              continue retry;
          // 其他情况，直接内层循环进行自旋即可
          // else CAS failed due to workerCount change; retry inner loop
      } 
  }
  boolean workerStarted = false;
  boolean workerAdded = false;
  Worker w = null;
  try {
      w = new Worker(firstTask);
      final Thread t = w.thread;
      if (t != null) {
          final ReentrantLock mainLock = this.mainLock;
          // worker的添加必须是串行的，因此需要加锁
          mainLock.lock();
          try {
              // Recheck while holding lock.
              // Back out on ThreadFactory failure or if
              // shut down before lock acquired.
              // 这儿需要重新检查线程池状态
              int rs = runStateOf(ctl.get());
              if (rs < SHUTDOWN ||
                  (rs == SHUTDOWN && firstTask == null)) {
                  // worker已经调用过了start()方法，则不再创建worker
                  if (t.isAlive()) // precheck that t is startable
                      throw new IllegalThreadStateException();
                  // worker创建并添加到workers成功
                  workers.add(w);
                  // 更新`largestPoolSize`变量
                  int s = workers.size();
                  if (s > largestPoolSize)
                      largestPoolSize = s;
                  workerAdded = true;
              }
          } finally {
              mainLock.unlock();
          }
          // 启动worker线程
          if (workerAdded) {
              t.start();
              workerStarted = true;
          }
      }
  } finally {
      // worker线程启动失败，说明线程池状态发生了变化（关闭操作被执行），需要进行shutdown相关操作
      if (! workerStarted)
          addWorkerFailed(w);
  }
  return workerStarted;
}
```
addWorker执行步骤:
  1. `compareAndIncrementWorkerCount` 使用CAS的方式增加worker数量。
  1. `t.start()` 然后启动Worker线程
### Worker 工作线程单元
```java
private final class Worker
  extends AbstractQueuedSynchronizer //本身就是AQS锁
  implements Runnable
{
  /** 工作线程,如果线程工厂初始失败则为空*/
  final Thread thread;
  /** 初始运行任务,可能为空 */
  Runnable firstTask;
  /** 每个线程任务计数器(已完成) */
  volatile long completedTasks;

  /**
   * 从ThreadFactory创建给定的第一个任务和线程。
   * @param firstTask the first task (null if none)
   */
  Worker(Runnable firstTask) {
      setState(-1); // inhibit interrupts until runWorker
      this.firstTask = firstTask;
      //线程工厂创建了一个线程。传入的参数为当前worker
      this.thread = getThreadFactory().newThread(this);
  }
}
```
### Worker.runworker 工作线程run函数
```java
final void runWorker(Worker w) {
  Thread wt = Thread.currentThread();
  Runnable task = w.firstTask;
  w.firstTask = null;
  // 调用unlock()是为了让外部可以中断
  w.unlock(); // allow interrupts
  // 这个变量用于判断是否进入过自旋（while循环）
  boolean completedAbruptly = true;
  try {
      // 这儿是自旋
      // 1. 如果firstTask不为null，则执行firstTask；
      // 2. 如果firstTask为null，则调用getTask()从队列获取任务。
      // 3. 阻塞队列的特性就是：当队列为空时，当前线程会被阻塞等待
      // 4. 当getTask()阻塞等待的时间超过 keepAliveTime 工作线程退出while循环 停止工作
      while (task != null || (task = getTask()) != null) {
          // 这儿对worker进行加锁，是为了达到下面的目的
          // 1. 降低锁范围，提升性能
          // 2. 保证每个worker执行的任务是串行的
          w.lock();
          // If pool is stopping, ensure thread is interrupted;
          // if not, ensure thread is not interrupted.  This
          // requires a recheck in second case to deal with
          // shutdownNow race while clearing interrupt
          // 如果线程池正在停止，则对当前线程进行中断操作
          if ((runStateAtLeast(ctl.get(), STOP) ||
               (Thread.interrupted() &&
                runStateAtLeast(ctl.get(), STOP))) &&
              !wt.isInterrupted())
              wt.interrupt();
          // 执行任务，且在执行前后通过`beforeExecute()`和`afterExecute()`来扩展其功能。
          // 这两个方法在当前类里面为空实现。
          try {
              beforeExecute(wt, task);
              Throwable thrown = null;
              try {
                  task.run();
              } catch (RuntimeException x) {
                  thrown = x; throw x;
              } catch (Error x) {
                  thrown = x; throw x;
              } catch (Throwable x) {
                  thrown = x; throw new Error(x);
              } finally {
                  afterExecute(task, thrown);
              }
          } finally {
              // 帮助gc
              task = null;
              // 已完成任务数加一 
              w.completedTasks++;
              w.unlock();
          }
      }
      completedAbruptly = false;
  } finally {
      // 自旋操作被退出，说明线程池正在结束
      processWorkerExit(w, completedAbruptly);
  }
}
```
* 当`getTask()`阻塞等待的时间超过 keepAliveTime 工作线程退出while循环 停止工作
* 核心线程与工作线程在`getTask()`区别
  ```java
  private Runnable getTask() {
    //非核心线程需要判断keepAliveTime超时逻辑
    boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
    ...
    try {
        Runnable r = timed ?
            //工作线程 判断keepAliveTime超时逻辑
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            //核心线程将阻塞等待任务
            workQueue.take();
        if (r != null)
            return r;
        timedOut = true;
    } catch (InterruptedException retry) {
        timedOut = false;
    }
  }
  ```

  1. 核心线程 `corePool` (合同工) 
  
      获取任务执行的是 `workQueue.take()` 将一直阻塞获取任务

  1. 工作线程(临时工) 
  
      则是`workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)` 
  
      超时返回 null 这个工作线程结束执行