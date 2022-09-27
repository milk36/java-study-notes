JMH -java Microbenchmark Harness
===
## 什么是JMH
JMH Java基准测试工具套件

[官网](http://openjdk.java.net/projects/code-tools/jmh/)

[官方例子](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)

[性能调优必备利器之 JMH](https://www.cnblogs.com/wupeixuan/p/13091381.html)

[JMH 使用说明](https://mp.weixin.qq.com/s/hJE4lneGppZ8M096_ALkxA)
## gradle 配置
* JMH Gradle Plugin 插件

  [JMH Gradle Plugin -- 该插件将 JMH 微基准测试框架与 Gradle 集成在一起。](https://github.com/melix/jmh-gradle-plugin)

  build.gradle 配置
  > 该插件在0.6.0之前的版本使用了 `me.champeau.gradle.jmh` 插件id。
  ```gradle
  apply plugin: 'me.champeau.jmh' 
  ```

  由于特定的配置，该插件可以轻松集成到现有项目中。特别是，基准源文件预计可以在 `src/jmh` 目录中找到
  ```gradle
  src/jmh
     |- java       : java sources for benchmarks
     |- resources  : resources for benchmarks
  ```
* jmh 配置块

  build.gradle 配置jmh 更多配置参数[参考](https://github.com/melix/jmh-gradle-plugin#configuration-options)
  ```gradle  
  jmh {
    //基准测试依赖于测试类时使用它
    includeTests = false
    //包含要执行的基准测试
    includes =['StringConnectTest']
    //排除执行的基准测试
    excludes=[]
  }
  ```
* gradle 运行
  ```gradle
  gradle jmh
  ```
## JMH中的基本概念
1. Warmup 预热，由于JVM中对于特定代码会存在优化（本地化），预热对于测试结果很重要
1. Mesurement 总共执行多少次测试
1. Timeout
1. Threads 线程数，由fork指定
1. Benchmark mode 基准测试的模式
1. Benchmark 测试哪一段代码  
## JMH 注解
* @BenchmarkMode 模式
  ```java
  //模式 AverageTime：用的平均时间，每次操作的平均时间，单位为 time/op
  @BenchmarkMode(Mode.AverageTime)
  //这个注解的 value 是一个数组，可以把几种 Mode 集合在一起执行
  @BenchmarkMode({Mode.SampleTime, Mode.AverageTime})
  ```

  其他类型
  ```java
  Throughput("thrpt", "Throughput, ops/time"), //整体吞吐量，每秒执行了多少次调用，单位为 ops/time
  AverageTime("avgt", "Average time, time/op"),//用的平均时间，每次操作的平均时间，单位为 time/op
  SampleTime("sample", "Sampling time"), //随机取样，最后输出取样结果的分布
  SingleShotTime("ss", "Single shot invocation time"), //只运行一次，往往同时把 Warmup 次数设为 0，用于测试冷启动时的性能
  All("all", "All benchmark modes"); //上面的所有模式都执行一次
  ```
* @Warmup 预热
  ```java
  //预热 iterations：预热的次数 time: 每次预热时间 timeUnit: 时间单位, 默认秒 batchSize：批处理大小，每次操作调用几次方法
  @Warmup(iterations = 3, time = 1)
  ```
  1. iterations：预热的次数
  1. time：每次预热的时间
  1. timeUnit：时间的单位，默认秒
  1. batchSize：批处理大小，每次操作调用几次方法
* @Measurement 实际测试
  ```java
  //实际测试 和@Warmup 参数相同 iterations进行测试的轮次，time每轮进行的时长，timeUnit时长单位。
  @Measurement(iterations = 5, time = 5)
  ```
* @Threads 测试线程数量
  ```java
  //测试线程数量 一般为cpu乘以2。
  @Threads(1)
  ```
* @Fork 进程数量
  ```java
  //进程数量 用多少个线程去执行我们的程序
  @Fork(1)
  ```
* @State 对象的作用范围

  通过 State 可以指定一个对象的作用范围 JMH 根据 scope 来进行实例化和共享操作。@State 可以被继承使用，如果父类定义了该注解，子类则无需定义
  ```java
  //指定一个对象的作用范围: Benchmark：所有测试线程共享一个实例;
  @State(value = Scope.Benchmark)
  ```
  1. Scope.Benchmark：所有测试线程共享一个实例，测试有状态实例在多线程共享下
  1. Scope.Group：同一个线程在同一个 group 里共享实例
  1. Scope.Thread：默认的 State，每个测试线程分配一个实例
* @OutputTimeUnit 统计结果的时间单位
  ```java
  //统计结果的时间单位
  @OutputTimeUnit(TimeUnit.MILLISECONDS)
  ```
* @Param 指定参数项  使用该注解必须定义 @State 注解。
  ```java
  @Param(value = {"10", "50", "100"})
  private int length;
  ```
* @Benchmark 用于标记指定的测试代码
  ```java
  @Benchmark
  public void wellHelloThere() {
      // this method was intentionally left blank.
  }
  public static void main(String[] args) throws RunnerException {
    Options opt = new OptionsBuilder()
            .include(JMHSample_01_HelloWorld.class.getSimpleName())
            .forks(1)
            .build();

    new Runner(opt).run();
  }
  ```
## 测试结果解读
* 测试基本信息
  ```
  > Task :java-jmh:jmh
  # JMH version: 1.29
  # VM version: JDK 1.8.0_311, Java HotSpot(TM) 64-Bit Server VM, 25.311-b11
  # VM invoker: D:\Program Files\Java\jdk1.8.0_311\jre\bin\java.exe
  # VM options: -Dfile.encoding=UTF-8 -Djava.io.tmpdir=D:\IdeaProjects\learnProjects\JavaExample\java-jmh\build\tmp\jmh -Duser.country=CN -Duser.language=zh -Duser.variant
  # Blackhole mode: full + dont-inline hint
  # Warmup: 3 iterations, 1 s each
  # Measurement: 5 iterations, 5 s each
  # Timeout: 10 min per iteration
  # Threads: 4 threads, will synchronize iterations
  # Benchmark mode: Average time, time/op
  # Benchmark: milk36.StringConnectTest.testStringAdd
  # Parameters: (length = 10) //测试参数
  ```
* 单次预热测试结果
  ```
  # Run progress: 0.00% complete, ETA 00:02:48
  # Fork: 1 of 1
  # Warmup Iteration   1: 161.395 ±(99.9%) 24.217 ns/op
  # Warmup Iteration   2: 133.802 ±(99.9%) 3.829 ns/op
  # Warmup Iteration   3: 132.379 ±(99.9%) 13.278 ns/op
  ```
* 单次实际测试结果  
  ```
  Iteration   1: 132.472 ±(99.9%) 7.102 ns/op
  Iteration   2: 133.905 ±(99.9%) 5.530 ns/op
  Iteration   3: 132.780 ±(99.9%) 8.578 ns/op
  Iteration   4: 141.014 ±(99.9%) 4.080 ns/op
  Iteration   5: 135.086 ±(99.9%) 4.920 ns/op

  Result "milk36.StringConnectTest.testStringAdd":
    135.051 ±(99.9%) 13.433 ns/op [Average]
    (min, avg, max) = (132.472, 135.051, 141.014), stdev = 3.489
    CI (99.9%): [121.618, 148.485] (assumes normal distribution)
  ```  
* 最后测试结果
  ```
  Benchmark                               (length)  Mode  Cnt     Score     Error  Units
  StringConnectTest.testStringAdd               10  avgt    5   135.051 ±  13.433  ns/op
  StringConnectTest.testStringAdd               50  avgt    5  1505.448 ±  34.706  ns/op
  StringConnectTest.testStringAdd              100  avgt    5  5329.087 ± 149.035  ns/op
  StringConnectTest.testStringBuilderAdd        10  avgt    5    68.571 ±   2.543  ns/op
  StringConnectTest.testStringBuilderAdd        50  avgt    5   385.299 ±   4.235  ns/op
  StringConnectTest.testStringBuilderAdd       100  avgt    5   781.717 ±  30.162  ns/op
  ```
  > Error那列其实没有内容，Score的结果是xxx ± xxx
## JMH 陷阱
JIT 优化中的死码消除

JMH 提供了两种方式避免这种问题，一种是将这个变量作为方法返回值 return a，一种是通过 Blackhole 的 consume 来避免 JIT 的优化消除
```java
@Benchmark
public void testStringBuilderAdd(Blackhole blackhole) {
  StringBuilder sb = new StringBuilder();
  for (int i = 0; i < length; i++) {
    sb.append(i);
  }
  blackhole.consume(sb.toString());//通过 Blackhole 的 consume 来避免 JIT 的优化消除
}
```
