ThreadLocal 强软弱虚 引用
===
### ThreadLocal
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
//1.8以后可以使用withInitial 初始化
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
* ThreadLocal 源码

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
          e != null;
          e = tab[i = nextIndex(i, len)]) {
          //替换旧值
          ThreadLocal<?> k = e.get();
          if (k == key) {
              e.value = value;
              return;
          }
          if (k == null) {
              replaceStaleEntry(key, value, i);
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
  