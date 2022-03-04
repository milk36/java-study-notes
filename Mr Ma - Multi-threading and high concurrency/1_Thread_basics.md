线程基础知识
===
### 基础知识
1. [2020版多线程，包括第一版的代码 代码仓库](http://git.mashibing.com/mashibing/juc2020)
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
1. 线程状态
    * New：新建状态 -> Thread.start()
    * Runnable：可运行状态 -> ready(挂起) running / thread.yield()
        * yield 屈服 让步
    * Waiting：等待状态 -> Lock.lock 可重入锁 公平锁 等
    * TimedWaiting：超时等待状态 -> Thread.sleep
    * Blocked：阻塞状态 -> synchronized 等待进入同步代码块的锁
    * Terminated：终止状态
    * 状态图:
    
        ![image](https://i.imgur.com/bmVkDY5.png)