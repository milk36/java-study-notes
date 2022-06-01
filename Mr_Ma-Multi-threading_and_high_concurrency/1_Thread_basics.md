线程基础知识
===
### 基础知识
1. 代码仓库:
    * [2020版多线程，包括第一版的代码 代码仓库](http://git.mashibing.com/mashibing/juc2020)
    * [JUC](http://git.mashibing.com/bjmashibing/JUC)
    * [多线程第二版资料 黄老师 马老师](http://git.mashibing.com/kuaile/duoxianchengkaifa-dierban)
    * [多线程第三版资料](http://git.mashibing.com/kuaile/duocianchengdisanban)
1. 压榨CPU性能的历史
    * 单进程 人工切换纸带机
    * 多进程批处理 多个任务批量执行
    * 多进程并行处理 程序写在不同内存位置来回切换
    * 多线程 一个程序内部不同任务的来回切换
    * 纤程/协程 golang
1. 进程/线程的定义
    * 进程 资源分配的基本单位
    * 线程 调度执行的基本单位 不同的执行路径
1. 设置多少线程数量最合适
    ```
    //N_cpu 处理器核数量, 可以通过 Runtime.getRuntime().availableProcessors()获取
    //U_cpu 期望的cpu利用率
    //W/C是等待时间与计算时间的比率
    N_threads = N_cpu * U_cpu *(1+W/C)

    ```
    > 实际情况中需要进行性能分析后才能得出 W/C的比率, 比如使用JProfiler
    
### 线程
1. 创建线程5种方式
    * 常见方式
        ```java
        //1. 继承Thread 父类 class MyThread extends Thread
        new MyThread().start();
        //2. 实现Runnable接口 class MyRun implements Runnable
        new Thread(new MyRun()).start();
        //3. lambda表达式
        new Thread(() -> {
            System.out.println("Hello Lambda!");
        }).start();
        //4. 线程池方式
        ExecutorService service = Executors.newCachedThreadPool();
        service.execute(() -> {
            System.out.println("Hello ThreadPool");
        });

        ```
    *  Callable接口实现 与Runnable不同的是有返回值
        ```java
        class MyCall implements Callable<String> {
            @Override
            public String call() {
                System.out.println("Hello MyCall");
                return "success";
            }
        }
        //线程池调用Callable
        Future<String> f = service.submit(new MyCall());
        String s = f.get();
        System.out.println(s);

        //FutureTask包装后 可以用Thread执行Callable
        FutureTask<String> task = new FutureTask<>(new MyCall());
        Thread t = new Thread(task);
        t.start();
        System.out.println(task.get());

        ```
1. 线程基本方法 sleep yield join
    * **sleep** 睡眠 让出CPU资源
    * **yield** 返回就绪状态 谦让退出一会 进入等待队列里 可能很快又回到执行状态
    * **join** 等待另外一个线程的结束 再运行
1. 线程6个状态
    * New：新建状态 -> Thread.start()
    * Runnable：可运行状态 -> ready(挂起/就绪) running / thread.yield()
        * yield 屈服 让步
    * Waiting：等待状态 -> Lock.lock 可重入锁 公平锁 等
    * TimedWaiting：超时等待状态 -> Thread.sleep
    * Blocked：阻塞状态 -> synchronized 等待进入同步代码块的锁
    * Terminated：终止状态
    * 获取当前线程的状态 `t.getState()`
    * 状态图:
    
        ![image](https://i.imgur.com/bmVkDY5.png)
1. 线程打断 interrupt
    * interrupt()
        * 打断某个线程(设置中断标志位)
    * isInterrupted()
        * 查询某线程是否被打断过(查询标志位)
    * **static** interrupted()
        查询当前线程是否被打断过, **并重置打断标记** 自动复位成false
        
        测试当前线程是否已中断。通过该方法可以清除线程的中断状态。
        
        换句话说，<big>如果这个方法被连续调用两次，第二个调用将返回false</big> (除非当前线程再次被中断，在第一个调用清除其中断状态之后，在第二个调用检查它之前)。

    * 设置中断标志位,是结束线程的一种方式(比较优雅)
    * InterruptedException 异常处理, 使用`sleep wait join`方法时 需要捕捉这个异常
        ```java
        Thread t = new Thread(() -> {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                System.out.println("Thread is interrupted")
                //try catch后候中断状态已经被自动复位成false, 以避免出现混乱情况
                System.out.println(Thread.currentThread().isInterrupted());
            }
        });
        t.start();
        SleepHelper.sleepSeconds(5);
        t.interrupt();
        ```
    * 如果加锁过程中 想执行中断 : `ReentrantLock.lockInterruptibly `
        > 获取锁，过程中当前线程可被中断。
1. 线程结束
    * 自然结束
    * ~~stop() suspend() resume()~~ 这三个方法已被废弃, 因为很容易产生数据不一致的问题
    * volatile 不好精确控制
    * interrupted 标志位,不好精确控制,可感知sleep wait操作 相对优雅
    * 使用锁配合其他线程, 才能实现精确控制
## 并发编程三大特性: 可见性, 有序性, 原子性
* **synchronized** 可以保障: 原子性 可见性; **不保障 有序性**
* **volatile** 可以保障:可见性, 有序性; **不保障 原子性** 
    * 特性
        1. 保证线程可见性
            - MESI 缓存一致性协议
        1. 禁止指令重排
            - DCL单例
            - Double Check Lock 双重检查
            - 单例模式还要不要加`volatile`? 

                要! `INSTANCE = new InstanceObj()`对象操作在编译器编译后分成三步指令

                1. 给指对象申请内存
                1. 给成员变量初始化
                1. 把这块内存的内容赋值给INSTANCE
    * A B线程都用到一个变量，java默认是A线程中保留一份copy，这样如果B线程修改了该变量，则A线程未必知道
    * 使用volatile关键字，会让所有线程都会读到变量的修改值
    * `System.out.println("hello");` 会触发内存同步刷新, 因为`println`内部有用到`synchronized`
    * `volatile`修饰引用类型只能针对引用地址保证其可见性, 引用内部属性不能保证
### 可见性
1. 线程可见性问题: A线程修改了变量, B线程能否感知变量已被修改

1. 可见性 与 缓存行
    * CPU L1 L2 L3缓存
    * cache line 缓存行 (64 bytes) 与`volatile`没有关系
        ```java
        private static class T {
            //前后7个long类型补齐了64 bytes,保证了 T.x存放在单独的缓存行
            private long p1, p2, p3, p4, p5, p6, p7;
            public long x = 0L;
            private long p9, p10, p11, p12, p13, p14, p15;
        }
    
        public static T[] arr = new T[2];
    
        static {
            //arr[0] arr[1] 应为填充了缓存行 两个数据在计算过程中 不会触发缓存一致性同步协议
            //从而提高计算性能
            arr[0] = new T();
            arr[1] = new T();
        }
        ```
    * JDK8引入了`@sun.misc.Contended`注解，来保证缓存行隔离效果 要使用此注解，必须去掉限制参数：-XX:-RestrictContended    
        ```java
        //注意：运行这个程序的时候，需要加参数：-XX:-RestrictContended
        import sun.misc.Contended;
        
        private static class T {
            @Contended  //只有1.8起作用 , 保证x位于单独一缓存行中
            public long x = 0L;
        }
        ```
    * MESI Cache **一致性协议** CPU每个cache line 标记四种状态
        1. Modified(修改) Exclusive(独占) Shared(共享) Invalid(失效)
        1. 缓存锁实现之一 有些无法被缓存的数据或者跨越多个缓存行的数据依然必须使用总线锁/缓存锁
    * 缓存行容量 64 bytes    
        1. cache line 越大, 局部性空间效率越高,但读取时间慢
        1. cache line 越小, 局部性空间效率月底,但读取时间快
        1. 工业实践取折中值, 目前多用:64 bytes
### 有序性
* 乱序 
    1. 为什么会乱序？主要是为了提高效率, CPU指令可能会乱序执行; 前提是先后两条语句没有依赖关系
    1. 乱序的原则: as-if-serial 看上去像是序列化（单线程）; 不影响单线程中最终一致性
        ```
        单个线程，两条语句，未必是按顺序执行
        单线程的重排序，必须保证最终一致性
        as-if-serial：看上去像是序列化（单线程）
        ```
        乱序测试代码
        ```java
        for(i<Integer.MAX;i++){
            a = b = y = z = 0;
            CountDownLatch latch = new CountDownLatch(2);
            Thread t1 = new Thread(() -> {
                //下面两行代码可能会打乱执行顺序
                a = 1;
                y = b;
                latch.countDown();
            });
            Thread t2 = new Thread(() -> {
                //下面两行代码可能会打乱执行顺序
                b = 1;
                z = a;
                latch.countDown();
            });
            t1.start();
            t2.start();
            try {
                latch.await();
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            if (y == 0 && z == 0) {
                System.err.println("第" + i + "次结果：" + y + "," + z);
                break;
            }
        }
        ```
    1. **半初始化状态**;对象创建过程中可能出现的重排序; 中间状态逸出; 
        * 对象半初始化状态
            ```
            0 new #2 <java/lang/Object> //申请一块内存
            3 dup
            4 invokespecial #1 <java/lang/Object.<init>> //构造方法 初始化成员变量 赋初始值
            7 astore_1 //与局部变量建立关联
            8 return
            ```
        * 可能存在中间状态逸出
            ```java
            private int num = 8;//刚刚申请内存创建对象 默认值 num=0
            public T03_ThisEscape() {
                //初始化过程中汇编指令 4 7可能出现重排的情况
                //num还是中间状态, 默认值; new Thread就启动了;可能输出0
                new Thread(() -> System.out.println(this.num)
                ).start();
            }
            public static void main(String[] args) throws Exception {
                new T03_ThisEscape(); 
                System.in.read();
            }
            ```
        * **不要在构造函数中启动线程** ,因为很容易出现上述中间状态逸出的问题
* 内存屏障
    1. CPU 使用内存屏障阻止乱序执行
    
        内存屏障是特殊指令：看到这种指令，前面的必须执行完，后面的才能执行

        intel : lfence sfence mfence(CPU特有指令)
    1. JVM内存屏障  
        ```
        Load 读操作
        Store 写操作
        
        四种类型:
        1. LoadLoad屏障: Load1;LoadLoad;Load2
            Load2及后续操作之前, 需要保证Load1读取数据完毕才能执行Load2.
        2. StoreStore屏障: Store1;StoreStore;Store2
            Store2及后续操作之前, 需要保证Store1读取数据完毕才能执行Store2.
        3. LoadStore屏障: Load1;LoadStore;Store2
            Store2及后续操作之前, 需要保证Load1读取数据完毕才能执行Store2.
        4. StoreLoad屏障: Store;StoreLoad;Load    
            Load2及后续操作之前, 需要保证Store1读取数据完毕才能执行Load2.
        ```
    1. 在java中使用volatile关键字可以**禁止指令重排** 和**保持线程可见性**
    
        volatile修饰的内存，不可以重排序，对volatile修饰变量的读写访问，都不可以换顺序
        
        ```java
        volatile实现细节 JVM层面
        
        //写操作
        ---StoreStoreBarrier---
        volatile 写
        ---StoreLoadbarrier---
        
        //读操作
        volatile 读
        ---LoadLoadBarrier---
        ---LoadStoreBarrier---
        ```    
    1. hotspot实现

        bytecodeinterpreter.cpp
        ```c++
        int field_offset = cache->f2_as_index();
                  if (cache->is_volatile()) {
                    if (support_IRIW_for_not_multiple_copy_atomic_cpu) {
                      OrderAccess::fence();
                    }
        ```
        orderaccess_linux_x86.inline.hpp
        ```c++
        inline void OrderAccess::fence() {
          if (os::is_MP()) {
            // always use locked addl since mfence is sometimes expensive
        #ifdef AMD64
            __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
        #else
            __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
        #endif
          }
        }
        ```
        **LOCK 用于在多处理器中执行指令时对共享内存的独占使用。**
        它的作用是能够将当前处理器对应缓存的内容刷新到内存，并使其他处理器对应的缓存失效。
        
        **另外还提供了有序的指令无法越过这个内存屏障的作用。**
### 原子性
* 基本概念

    1. race condition => **竞争条件** ， 指的是多个线程访问共享数据的时候产生竞争

    1. **数据的不一致**（unconsistency)，并发访问之下产生的不期望出现的结果

    如何保障数据一致呢？--> 线程同步（线程执行的顺序安排好）
    
    1. **原子操作** : 中间过程不能被其它线程打断
    
        在java代码中任何一句简单的代码如:`n++`翻译成汇编后,
        
        都可能需要执行多条汇编指令, 在这个过程中很有可能出现被其他线程打断的情况
        
        常见的保证原子性方法 -- 加锁 `synchronized`
    
* 上锁的本质: 把并发操作, 变成序列化操作

    monitor （管程） ---> 锁

    critical section -> 临界区; 如果临界区执行的时间长, 语句多, 叫做锁的 **粒度比较粗**, 反之就是锁的 **粒度比较细**
* 悲观锁/乐观锁
    * 悲观锁 默认操作可能会被其它线程打断 提前在执行代码块加锁 `synchronized`
    * 乐观锁 默认操作不会被其它线程打断 CAS/自旋锁/无锁
        * CAS操作 Compare And Set/Swap/Exchange 比较替换(设置/修改)
        * ABA问题 解决方案 - Version(增加版本标识)
        * 最终实现：
            ```
            //非原子操作
            cmpxchg = cas修改变量值 
            //lock指令在执行的时候视情况采用缓存锁或者总线锁
            lock cmpxchg 
            ```
    * 两种锁的效率 根据不同场景分析
        * 乐观锁 消耗CPU资源 不断轮询
        
            适合场景: 时间短,等待队列少
        * 悲观锁 处于等待队列中不消耗CPU资源 
        
            适合场景: 临界区执行时间较长(运算时间长),等待队列长
    
    