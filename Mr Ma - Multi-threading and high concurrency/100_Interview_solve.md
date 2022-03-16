面试解题
===
### 面试题1

  实现一个容器，提供两个方法add、size，写两个线程：

  线程1，添加10个元素到容器中

  线程2，实时监控元素个数，当个数到5个时，线程2给出提示并结束

  > volatile一定要尽量去修饰普通的值，不要去修饰引用值，因为volatile修饰引用类型，这个引用对象指向的是另外一个new出来的对象对象，如果这个对象里边的成员变量的值改变了，是无法观察到的
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
  Object lock = new Object();
  new Thread(()->{
    synchronized(lock){
      //if(size()==5){
        try{
          //启动后就等待
          lock.wait();
        }catch(InterruptedException e){
          e.printStackTrace();
        }
      //}
      System.out.println("size:"+size())
      //通知T2继续执行
      lock.notify();
    }
  },"T1").start();

  new Thread(()->{
    synchronized(lock){
      for(int i=0 ;i<10;i++){
        add(i);
        System.out.println("add:"+i);
        if(i == 5){
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