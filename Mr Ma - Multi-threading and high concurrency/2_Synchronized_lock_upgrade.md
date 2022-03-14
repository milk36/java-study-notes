Synchronized锁升级深入详解
===
### 用户态 与 内核态
* 内核态 能访问所有指令
* 用户态 只能访问用户能访问的指令

### 锁的升级过程
#### markword
* 工具：JOL = Java Object Layout
    ```xml
    <dependencies>
            <!-- https://mvnrepository.com/artifact/org.openjdk.jol/jol-core -->
            <dependency>
                <groupId>org.openjdk.jol</groupId>
                <artifactId>jol-core</artifactId>
                <version>0.9</version>
            </dependency>
    </dependencies>
    ```
* jdk8u: markOop.hpp
    ```java
    // Bit-format of an object header (most significant first, big endian layout below):
    //
    //  32 bits:
    //  --------
    //             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
    //             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
    //             size:32 ------------------------------------------>| (CMS free block)
    //             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
    //
    //  64 bits:
    //  --------
    //  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
    //  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
    //  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
    //  size:64 ----------------------------------------------------->| (CMS free block)
    //
    //  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
    //  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
    //  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
    //  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
    ```
* java对象的内存布局 (HotSpot 实现)    
    1. 锁信息记录在 `markword`中

        synchronized就是把锁存在Java对象头里
        
        Java对象头包括两个部分数据: **Mark Word**(标记字段); **Klass Pointer** (类型指针)
        
        **Mark Word**：默认存储对象的HashCode，分代年龄和锁标志位信息。
        
        **Klass Point**：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
        ```
        //8的整数倍
        [] 8 byte markword
        [] 4 byte klass pointer(类型指针) T.class
        [] n byte instance data(成员变量 大小看具体有多少成员变量)
        --------------------------------------------------------------
        OFF  SZ               TYPE DESCRIPTION               VALUE
          0   8                    (object header: mark)     0x0000000000000001 (non-biasable; age: 0) //markword
          8   4                    (object header: class)    0xf802a74f  //类型指针 默认压缩4字节
        12   4                int TestInfo.id               203225496 //成员变量
        16   4   java.lang.String TestInfo.name             (object)  //成员变量
        20   4                    (object alignment gap)    //对齐4字节
        Instance size: 24 bytes //总共24字节
        Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
        ```
        * Hotspot实现
            ![image](https://i.imgur.com/GBXfmdQ.png)
            ![image](https://image-static.segmentfault.com/260/918/2609180466-4799c6800a5379d1)
    1. 锁的升级过程
        * 普通对象 加锁(Synchronized) -> 偏向锁 -> 轻量级锁 -> 重量级锁
            1. (用户态 用户空间操作)  偏向锁 ->  第一访问的线程 把线程id写入markword
            1. (用户态 用户空间操作) **偏向锁** 升级到 **轻量锁** -> 竞争 将当前线程LR(Lock record 锁记录) 写入markword -> 竞争失败继续CAS 自旋
            1. (内核态 需要向内核申请) **重量锁**
        * Synchronized 加锁对象后 monitorenter 内部实现
            ```
            //判断是否为偏向锁
            if(UseBiasedLocking){
                ObjectSynchronizer::fast_enter()//快速 不需要锁竞争
            }else{
                ObjectSynchronizer::slow_enter()//慢速 自旋 -> 膨胀升级 -> 重量级锁
            }
            ```   
        * 锁重入
            synchronized是可重入锁

            重入次数必须记录, 因为解锁几次必须要对应

            重入信息记录在 偏向锁 LR(Lock record) 记录到线程栈 -> LR + 1

            `markword中的hashCode也存入到线程栈里面`  **轻量级锁也是类似结构**

            重量级锁 -> ? ObjectMonitor 字段上
        * 自旋锁什么时候升级到重量锁 

            有线程超过10次自旋 -XX:PreBlockSpin 或者自旋线程超过CPU核数的一半

            JDK 1.6以后 加入自适应自旋 Adapative Self Spinnning , JVM 自己控制   
        * 为什么有自旋锁还需要重量级锁?    
            
            自旋是消耗CPU资源的, 如果锁的时间长, 或者自旋线程多, CPU会被大量消耗

            重量级锁有等待队列, 所有拿不到所得进入等待队列, 不需要消耗CPU资源

            `ObjectMonitor.WaitSet` 锁的等待队列
        * 偏向锁是否一定比自旋锁效率高?

            不一定, 在明确知道会有多线程竞争的情况下, 偏向锁肯定会设计到锁撤销, 这时候直接使用自旋锁效率更高

            JVM启动过程, 会有很多线程竞争(这是必然的), 所以默认情况启动时不打开偏向锁(启动延迟4秒), 过一段时间再打开

            `-XX:BiasedLockingStartupDelay=0` JVM设置偏向锁的启动延迟时间
        ![image](https://i.imgur.com/xDwBciC.png)