ThreadLocal , 强软弱虚 引用
===
### ThreadLocal

[ThreadLocal的作用以及应用场景](https://mp.weixin.qq.com/s/klhLeoVgPKzRoTVSBmjFFA)

ThreadLocal 在多线程中为每一个线程创建单独的变量副本的类; 

当使用ThreadLocal来维护变量时, ThreadLocal会为每个线程创建<big>单独的变量副本</big>, 避免因多线程操作共享变量而导致的数据不一致的情况。

使用场景: 每个线程对应一个独立的数据库链接 ; 将用户ID , 事务ID 与线程关联起来

`initialValue` 初始值 `get() set() remove()` 获取/设置/删除

使用样例 
```java
private static final ThreadLocal<StringBuilder> stringbuffer = new ThreadLocal<StringBuilder>() {
    @Override
    protected StringBuilder initialValue() {
        return new StringBuilder("buffer 1:");
    }
};
//1.8以后可以使用 withInitial 初始化 
//lambda形式的封装 最后还是调用的 initialValue
private static final ThreadLocal<StringBuilder> stringbuffer2 = ThreadLocal.withInitial(() -> new StringBuilder("buffer 2:"));

public static void main(String[] args) {
    new Thread(() -> {
        System.out.println();
        stringbuffer.get().append("t1");
        stringbuffer2.get().append("t1");
        System.out.println(stringbuffer.get());
        System.out.println(stringbuffer2.get());
    }).start();
    new Thread(() -> {
        System.out.println();
        stringbuffer.get().append("t2");
        stringbuffer2.get().append("t2");
        System.out.println(stringbuffer.get());
        System.out.println(stringbuffer2.get());
    }).start();
}
```
输出:
```
buffer 1:t1
buffer 2:t1

buffer 1:t2
buffer 2:t2
```
* ThreadLocal 类图

  ![ThreadLocal](https://i.imgur.com/4Ftvec2.png "ThreadLocal")
* ThreadLocalMap 结构图

  <!-- ![ThreadLocalMap](https://i.imgur.com/cb6A9r5.png "ThreadLocalMap")   -->
  ![](https://i.imgur.com/oCgArZV.png "ThreadLocalMap 结构")
* ThreadLocal 源码
  
  [深度解析ThreadLocal源码](https://segmentfault.com/a/1190000022663697#item-14)

  在 ThreadLocal 源码实现中 ，涉及到了 ：数据结构、拉链存储、斐波那契数列、神奇的 0x61c88647、弱引用Reference、过期 key 探测清理等等
  * ThreadLocalMap Hash算法
  
    每当创建一个ThreadLocal对象，这个ThreadLocal.nextHashCode 这个值就会增长 0x61c88647 

    这个值很特殊，它是斐波那契数 也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash 分布非常均匀。
    ```java
    private static final int HASH_INCREMENT = 0x61c88647;
    public static void main(String[] args) {
        int hashCode = 0;
        for (int i = 0; i < 16; i++) {
        hashCode = i * HASH_INCREMENT + HASH_INCREMENT;
        int bucket = hashCode & 15;
        System.out.println(i+" bucket:"+bucket);
        }
    }
    ```

  * `ThreadLocalMap.Entry` 结构 Entry[] (初始大小 = 16) 用于存储每个 ThreadLocal 对应 value

    `Entry <ThreadLocal<?> k, Object v>` 
    ```java
    private static final int INITIAL_CAPACITY = 16;
    Entry[] table = new Entry[INITIAL_CAPACITY];

    int i = key.threadLocalHashCode & (len-1);
    tab[i] = new Entry(key, value)
    ```
  * `ThreadLocal.set(value)`

    简化逻辑 `Thread.currentThread().ThreadLocalMap.set(this,value)` 给当前线程设置独立变量
    ```java
    public void set(T value) {
        Thread t = Thread.currentThread();//拿到当前线程对象
        ThreadLocalMap map = getMap(t);// t.threadLocals 当前线程 map
        if (map != null) {
            map.set(this, value);//添加 或 覆盖 ThreadLocalMap.set()
        } else {
            createMap(t, value);//初始创建 map new ThreadLocalMap() ;
            // t.threadLocals = new ThreadLocalMap(this, firstValue)
        }
    }
    ```
  * `new ThreadLocalMap()`
    ```java
    ThreadLocal.ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];//INITIAL_CAPACITY =16
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);//用ThreadLocal hash & (与上) Entry数组长度 , 计算出ThreadLocal Key对应的index
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);//ThreadLocalMap.threshold = len * 2 / 3;
    }
    ```
  * `ThreadLocalMap.set()` 
    ```java
    private void set(ThreadLocal<?> key, Object value) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);
      for (Entry e = tab[i];
          e != null;//进入循环中替换旧值
          e = tab[i = nextIndex(i, len)]) {
          //替换旧值
          ThreadLocal<?> k = e.get();
          if (k == key) {
              e.value = value;
              return;
          }
          if (k == null) {//处理弱引用 key为空的Entry
              replaceStaleEntry(key, value, i);//替代旧 Entry ; 旧 Entry key 因为没有强引用关联 以为空
              return;
          }
      }
      //添加新值
      tab[i] = new Entry(key, value);
      int sz = ++size;
      //扩容检查
      if (!cleanSomeSlots(i, sz) && sz >= threshold)
          rehash();//ThreadLocalMap.threshold 扩容临界值
    }
    ```
  * `ThreadLocal.get()` 获取数据 必要时触发初始化 `initialValue `
    ```java
    public T get() {
            Thread t = Thread.currentThread();
            ThreadLocalMap map = getMap(t);
            if (map != null) {
                ThreadLocalMap.Entry e = map.getEntry(this);
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    T result = (T)e.value;
                    return result;
                }
            }
            return setInitialValue();
        }
    ```
### 强软弱虚 引用
* 强 引用 

  `NormalReference`普通类型引用就是 => "Strong" Reference 强引用

  普通的引用也就是默认的引用，默认的引用就是说，只要有一个应用指向这个对象，那么垃圾回收器一定不会回收它

  当内存空间不足时，JAVA虚拟机宁可抛出OutOfMemoryError终止应用程序也不会回收具有强引用的对象。

  `finalize()`这个方法永远都不需要重写，而且也不应该被重写
  ```java
  public class M {
    @Override
    //正常情况下不用重写 finalize 方法 该方法 Deprecated 以弃用
    protected void finalize() throws Throwable {
        System.out.println("finalize");
    }
  }

  M m = new M();
  m = null;//置空 m 没有了引用关联 才会被回收
  System.gc(); //DisableExplicitGC 显示调用GC
  System.out.println(m);  
  ```
  执行GC后 `finalize()` 方法执行打印输出
* Reference 子类 `软 弱 虚` 引用速览

  |引用类型|被GC条件|用途|生存时间|
  |:-:|:-:|:-:|:-:|
  |强引用|不会|普通对象状态|JVM停止运行时|
  |软引用|内存不足|对象缓存|内存不足时|
  |弱引用|GC时|对象缓存|GC时|
  |虚引用|GC时|动态内存/堆外内存操作|GC时|

  ```java
  M softObj = new M("Soft-M");
  ReferenceQueue softQueue = new ReferenceQueue();
  SoftReference softReference = new SoftReference(softObj, softQueue);

  M weakObj = new M("Weak-M");
  ReferenceQueue weakQueue = new ReferenceQueue();
  WeakReference weakReference = new WeakReference(weakObj, weakQueue);

  M phantomObj = new M("Phantom-M");
  ReferenceQueue phantomQueue = new ReferenceQueue();
  PhantomReference pReference = new PhantomReference(phantomObj, phantomQueue);

  System.out.println("softReference:"+softReference.get());
  System.out.println("weakReference:"+weakReference.get());
  System.out.println("phantomReference:"+pReference.get());//phantom 只有虚引用一开始就无法 get 出引用对象

  softObj=null;weakObj=null;phantomObj=null;
  System.gc();
  System.out.println("GC after...");
  System.out.println("softReference:"+softReference.get());
  System.out.println("weakReference:"+weakReference.get());//gc后 weak引用也被回收
  System.out.println("phantomReference:"+pReference.get());
  ```  
* 软 引用 

  SoftReference  <big>当内存不够时 软引用才被回收掉</big>

  适合使用场景: 缓存
  ```java
  //VM启动参数: -Xms20M -Xmx20M
  public class T02_SoftReference {
    public static void main(String[] args) {
        SoftReference<byte[]> sr = new SoftReference<>(new byte[1024 * 1024 * 10]);//占用 10M 内存空间
        System.out.println(sr.get());
        System.gc();//内存足够 不会被GC掉
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(sr.get());//非空 还在内存里

        //再分配一个数组，heap将装不下，这时候系统会垃圾回收，先回收一次，如果不够，会把软引用干掉
        byte[] b = new byte[1024 * 1024 * 12];//新申请 12M 内存空间
        System.out.println(sr.get());//内存不足 就被清理掉 输出 null
    }
  }
  ```
* 弱 引用

  `WeakReference` <big>只要遇到 GC 就会被回收</big>

  无论当前内存是否紧缺，GC都将回收被弱引用关联的对象。

  使用场景: 如果一个强引用指向了一个弱引用 只要强引用消失 这个弱引用就会被GC回收
  ```java
  WeakReference<M> wr = new WeakReference<>(new M());

  System.out.println(wr.get());
  System.gc();
  SleepHelper.sleepSeconds(1);
  System.out.println(wr.get());//被回收 打印 null 
  ```  
  * ThreadLocal 用途

    应用场景: Spring 中的 Transaction connection 事务连接, 保持同一个 connection
  * ThreadLocal 可能会出现内存泄漏的情况

    ![](https://blog.xiaohansong.com/media/15473672636979/15473735704801.jpg "ThreadLocal 内存关系")    

    内存泄漏 memory leak，**申请了内存用完了不释放, 对象已经死了** 不可回收内存空间变多

    内存溢出 out of memory , **申请内存时, 没有足够的内存可以使用** 堆内存空间已满

    `ThreadLocalMap.Entry extends WeakReference<ThreadLocal<?>>` 继承自弱引用
    ```java
    ThreadLocal<M> tl = new ThreadLocal<>();// tl 是一个ThreadLocal类型的强引用
    tl.set(new M());//内部实现为 new Entry(key, value) 为一个弱引用
    tl.remove();//及时调用 remove 防止内存泄漏
    ```
    `ThreadLocalMap` 是和线程绑定 , tl 强引用消失 可以确保 `ThreadLocalMap.Entry.reference` 被回收 也就是 key 被回收
    
    ```java
    ThreadLocal local = new ThreadLocal();
    Thread thread = Thread.currentThread();
    local.set(thread.getName());
    local = null;
    System.gc();//GC后 key因为失去外部强引用 而被回收掉
    System.out.println(thread);//但这时断点查看 当前线程内部 threadLocals 会有 reference = null ; value = "main"
    ```
    <big>弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除</big>

    但是这些被动的预防措施并不能保证不会内存泄漏

    `ThreadLocalMap.Entry` 的弱引用只能确保 key回收 value还没有回收掉 
    
    还是会出现内存泄漏的情况

    ThreadLocal 内存泄漏的原因: 
    1. 没有手动删除Entry 手动调用`ThreadLocal.remove()`
    1. CurrentThread依然在运行

    由于ThreadLocalMap的生命周期跟Thread一样长, 如果没有手动删除对应key 就会导致内存泄漏

    避免内存泄漏的两种方式:
    1. 使用完ThreadLocal 调用`remove()`方法删除对应的Entry
    1. 使用完ThreadLocal 当前 Thread 也随之运行结束

  * `ThreadLocalMap.Entry` 使用弱引用的原因

    通过弱引用的特性 可以实现对 Map 过期数据的标识

    Entry 数据 key 为 null 则说明数据过期 , 表明此数据key值已经被垃圾回收掉了

    后续在 set get remove 操作中就可以将过期数据进行清理

    探测式清理`expungeStaleEntry()`, 启发式清理`cleanSomeSlots()`
* 虚 引用 

    `PhantomReference` 触发GC 后虚引用将被放入 事先定义好的`ReferenceQueue`队列中 这意味着该实例仍在内存中

    对象设置一个虚引用的唯一目的是：能在此对象被垃圾收集器回收的时候收到一个系统通知

    无法获得虚引用引用. 引用对象永远无法通过 API 直接访问 get不到值
    ```java
    public T get() {
        return null;//覆盖了Reference.get()并且总是返回null
    }
    ```

    GC后只能得到一个通知 `ReferenceQueue` 引用队列就是获取这个通知的方式

    适用场景: 
    1. 动态内存 直接内存 `ByteBuffer.allocateDirect` 堆外内存(回收) 关联知识:"NIO零拷贝"
    1. 确定对象何时从内存中删除，这有助于调度内存敏感任务。例如，我们可以等待一个大对象被移除，然后再加载另一个对象。
    1. 避免使用finalize方法，改进finalize过程。
    ```java
    //参考:https://www.baeldung.com/java-phantom-reference
    static class LargeObjectFinalizer extends PhantomReference<Object> {
        public LargeObjectFinalizer(Object referent, ReferenceQueue<? super Object> q) {
            super(referent, q);
        }
        //自定义实现的 资源释放逻辑
        public void finalizeResources() {
            // free resources
            System.out.println("clearing ...");
        }
    }
    public static void main(String[] args) {
        ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();//引用队列
        ArrayList<LargeObjectFinalizer> references = new ArrayList<>();//大对象的虚引用集合
        ArrayList<Object> largeObjects = new ArrayList<>();//大对象集合

        for (int i = 0; i < 10; i++) {
            Object largeObject = new Object();
            largeObjects.add(largeObject);
            //封装虚引用
            references.add(new LargeObjectFinalizer(largeObject, referenceQueue));
        }
        //解除引用
        largeObjects = null;
        System.gc();//执行垃圾回收
        Reference<?> referenceFromQueue;
        for(PhantomReference<Object> reference : references){
            System.out.println(reference.isEnqueued());//返回此引用是否已被GC回收
        }
        while((referenceFromQueue = referenceQueue.poll())!=null){
            ((LargeObjectFinalizer)referenceFromQueue).finalizeResources();//执行 自定义资源释放逻辑
            referenceFromQueue.clear();//执行清除操作
        }
    }
    ```
* Reference    

  参考:[深入理解JDK中的Reference原理和源码实现](https://www.cnblogs.com/throwable/p/12271653.html)

  `Reference`的继承关系或者实现是由JDK定制，引用实例是由JVM创建，所以自行继承Reference实现自定义的引用类型是无意义的，但是可以继承已经存在的引用类型，如SoftReference等.

  `Reference` 是 `软 弱 虚` 三种引用的共同父类

  这个类不能直接子类化

  `Reference` 成员变量
  ```java
  public abstract class Reference<T> {
    private T referent;//保存的引用指向的对象
    volatile ReferenceQueue<? super T> queue;//引用队列 对象如果即将被垃圾收集器回收 此队列作为通知的回调队列
    volatile Reference next;//下一个 引用 通过此构造单向链表
    private transient Reference<T> discovered;
  }
  ```
  * 可达性

    当一个对象到GC根集没有任何引用链相连时，则证明此对象是不可用的。
    
    不可用的对象"有机会"被判定为可以回收的对象。

    在Java语言中，可以作为GC根集的对象包括下面几种:
    1. 虚拟机栈(栈帧中的本地/局部变量表)中引用的对象。
    1. 方法区中常量引用的对象(在JDK1.8之后不存在方法区，也就是有可能是metaspace中常量引用的对象)。
    1. 本地方法栈中JNI(即一般常说的Native方法)引用的对象。

* ReferenceQueue   
  存放引用的队列 保存`Reference`对象  

  用于在`Reference`关联的对象引用被GC回收时, 该`Reference`对象将会被加入`ReferenceQueue`引用队列中的末尾

  * ReferenceQueue使用方式
    ```java
    class MSoftReference extends SoftReference<M>{
        String gc_logic_name;
        public MSoftReference(M referent, ReferenceQueue q) {
            super(referent, q);
            gc_logic_name = referent.name;
        }
    }
    class SoftReferenceExample{
        static List<SoftReference<M>> softReferences = new ArrayList<>();
        //引用队列旨在让我们了解垃圾收集器执行的操作。当JVM决定删除此引用的所指对象时，它将引用对象附加到引用队列中
        static ReferenceQueue<M> queue=new ReferenceQueue();
        static volatile boolean gcWatchThreadRun = true;
        public SoftReferenceExample() {
            System.out.println("Starting now...");
            new Thread(()->{
                MSoftReference ref;
                while (gcWatchThreadRun) {
                    if((ref= (MSoftReference) queue.poll())!=null) {
                        System.out.println("GC ref:" + ref.gc_logic_name );
                    }
                }
            }).start();
            for (int i = 0; i <50000 ; i++) {
                M softObj = new M("Soft-M-" + i);
                softReferences.add(new MSoftReference(softObj, queue));
            }
            System.out.println("Completed !");
            gcWatchThreadRun = false;
        }
    }
    ```
* ReferenceHandler

  `ReferenceHandler`线程是`Reference`的静态代码创建的，所以只要`Reference`这个父类被初始化，该线程就会创建和运行，由于它是守护线程，除非JVM进程终结，否则它会一直在后台运行
* WeakHashMap

  通过`WeakReference`和`ReferenceQueue`实现的。 `WeakHashMap`的key是“弱键”，即是`WeakReference`类型的;`ReferenceQueue`是一个队列，它会保存被GC回收的“弱键”

  `expungeStaleEntries()` 清理过期记录, getTable() size()和resize() 方法都会调用该方法。基本上 WeakHashMap 的每个方法都会调用 getTable()

  如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。 接着，WeakHashMap会根据“引用队列”，来删除“WeakHashMap中已被GC回收的‘弱键’对应的键值对”

  通俗的说在 WeakHashMap 中，当某个 key 不再正常使用时，将自动移除该 Entry。

  [参考](https://segmentfault.com/a/1190000039668399#item-1-6)
  ```java
  //JVM: -Xmx20M -verbose:gc
  WeakHashMap<Object,Object> map = new WeakHashMap();
  Object value = new Object();
  for (int i = 0; i <20 ; i++) {
    //内存不足将触发GC WeakHashMap中的key会被回收掉
    map.put(new MyObject("Obj:" + i,new byte[1024*1024]), value);
  }
  System.out.println("size:"+map.size());//因为限制堆内存只有20M 最后实际打印size数量将远远小于20
  ```  

  * HashMap 配合 `WeakReference ReferenceQueue`来实现类似功能
    ```java
    //JVM: -Xmx20M -verbose:gc
    Object value = new Object();
    HashMap hashMap = new HashMap();
    //虽然设置堆内存只有20M 但因为使用弱引用 key能够及时的被GC回收 而不会发生OOM 内存溢出
    for (int i = 0; i <2000 ; i++) {
        hashMap.put(new WeakReference(new MyObject("Obj:" + i, new byte[1024*1024]), queue)
                , value);
    }
    System.out.println("map.size:"+hashMap.size());

    Thread t1 = new Thread(() -> {
        WeakReference<MyObject> ref;
        try {
            //GC时 被清除的的引用存入ReferenceQueue
            //执行到这一行代码时 理论上已经执行了很多次GC ReferenceQueue中已经有很多WeakReference
            while ((ref = (WeakReference<MyObject>) queue.remove(100)) != null) {
                hashMap.remove(ref);
            }
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
    });

    t1.start();
    try{
        t1.join();
    }catch(InterruptedException ex){
        ex.printStackTrace();
    }
    System.out.println("after map.size:"+hashMap.size());
    ```
  
