# 使用JMH实现性能测试

## 前言

如果我们开发一个很实用的功能，会期望能让更多人用，以发挥他的价值。但是事实上，能不能被使用者接受，最终往往由一些功能之外的原因决定，比如稳定性，可用性，性能等。所以需要在这些非功能方面做到足够好，才能被大家所接受。要把东西做好，就需要知道它目前表现得怎么样。所以我们需要一套方法测试它目前的表现，只有测出来的数据才能说明一切。这些非功能的指标很多，相应的测试方法也有很多。这里我们就聊性能相关的测试。

性能指执行任务的能力（速度），一般可以用`吞吐量`和`影响时间`两个指标来表示。性能测试一般就是要测出这两个参数。

> * 吞吐量（throughput）表示单位时间能处理的任务数量，决定任务处理进度
> * 响应时间（response time）表示单个任务处理从提交到处理完成的耗时，影响用户体验

性能测试方法一般是启多个进程和线程同时循环执行任务，测量各种压力下的这两个参数的值。总结起来，一个性能测试的工作，包括：

* 编写压测逻辑：调用被测方法执行任务
* 编写起压逻辑：起进程和线程执行压测方法
* 编写控制逻辑：何时开始运行，何时开始采集数据，何时停止
* 编写数据收集逻辑：收集压测过程中`吞吐量`和`响应时间`这些信息
* 编写报告生成逻辑：生成一个结论性的报表
* 实际起压测试，获取报表

这些工作中，除了`编写压测逻辑`对于每个压测任务不同，其余的工作几乎都是相同的，可以复用。[`JMH`](http://openjdk.java.net/projects/code-tools/jmh/)就是实现了这样一些通用的逻辑，让使用者不再为这些东西花费心思，可以非常简单地编写压测程序，将心思都放在压测逻辑的编写、压测结果的分析和被测代码的优化上。

## 简单的JMH性能测试程序

一般首先使用JMH所提供的archetype创建压测工程，archetype如下

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-java-benchmark-archetype</artifactId>
    <version>1.21</version>
</dependency>
```

然后就可以在`src/main/java`目录下编写压测程序了。压测程序很简单，只需要在一个`public`成员方法上加上`@Benchmark`注解，那么这个方法就是被测试的目标方法。比如我们写一个判断字符串数(`"1234"`)是不是整数的方法。

```java
public class NumberTestBenchmark {

    public static boolean isNumber(String value) {
        try {
            Integer.parseInt(value);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    // 这就是一个被测试的目标方法
    @Benchmark
    public boolean isNumber() {
        return isNumber("1234");
    }

}
```

这就是一个完整的benchmark了。在IDE中写测试用例时，可以安装JMH插件，然后像执行单元测试那样执行benchmark。执行结果如下：（后续将详细介绍[benchmark启动方式](#benchmark启动方式)）

```
# JMH version: 1.19
# VM version: JDK 1.8.0_131, VM 25.131-b11
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre/bin/java
# VM options: -Dfile.encoding=UTF-8
# Warmup: 20 iterations, 1 s each
# Measurement: 20 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: benchmark.demo.NumberTestBenchmark.isNumber

# Run progress: 0.00% complete, ETA 00:00:40
# Fork: 1 of 1
# Warmup Iteration   1: 62970693.345 ops/s
# Warmup Iteration   2: 93324937.645 ops/s
# Warmup Iteration   3: 105334487.871 ops/s
# Warmup Iteration   4: 102237892.363 ops/s
# Warmup Iteration   5: 104664621.717 ops/s
# Warmup Iteration   6: 104931613.208 ops/s
# Warmup Iteration   7: 104509623.222 ops/s
# Warmup Iteration   8: 104715106.648 ops/s
# Warmup Iteration   9: 103632707.411 ops/s
# Warmup Iteration  10: 101949112.063 ops/s
# Warmup Iteration  11: 98902292.905 ops/s
# Warmup Iteration  12: 101377811.592 ops/s
# Warmup Iteration  13: 101479726.188 ops/s
# Warmup Iteration  14: 100134341.643 ops/s
# Warmup Iteration  15: 100348767.127 ops/s
# Warmup Iteration  16: 100711223.068 ops/s
# Warmup Iteration  17: 83685861.940 ops/s
# Warmup Iteration  18: 95675372.260 ops/s
# Warmup Iteration  19: 98437784.006 ops/s
# Warmup Iteration  20: 100731674.069 ops/s
Iteration   1: 100901550.381 ops/s
Iteration   2: 100799647.361 ops/s
Iteration   3: 102025437.593 ops/s
Iteration   4: 100232830.105 ops/s
Iteration   5: 101527005.670 ops/s
Iteration   6: 102005446.245 ops/s
Iteration   7: 99333047.435 ops/s
Iteration   8: 99756746.049 ops/s
Iteration   9: 100589892.099 ops/s
Iteration  10: 100632598.021 ops/s
Iteration  11: 102487874.209 ops/s
Iteration  12: 102538183.096 ops/s
Iteration  13: 95965921.238 ops/s
Iteration  14: 102804822.281 ops/s
Iteration  15: 103465675.891 ops/s
Iteration  16: 104221280.605 ops/s
Iteration  17: 103730011.157 ops/s
Iteration  18: 103779527.007 ops/s
Iteration  19: 102530124.572 ops/s
Iteration  20: 104076161.974 ops/s


Result "benchmark.demo.NumberTestBenchmark.isNumber":
  101670189.150 ±(99.9%) 1728020.035 ops/s [Average]
  (min, avg, max) = (95965921.238, 101670189.150, 104221280.605), stdev = 1989990.442
  CI (99.9%): [99942169.114, 103398209.185] (assumes normal distribution)


# Run complete. Total time: 00:00:41

Benchmark                      Mode  Cnt          Score         Error  Units
NumberTestBenchmark.isNumber  thrpt   20  101670189.150 ± 1728020.035  ops/s
```

我们能看到：

* 在启动之后，正式开始执行之前，JMH会打印出当前的环境信息已经执行的配置信息。
* 然后可以看到开始执行benchmark的过程分为两个阶段，`预热`和`正式测试`。而每个阶段又都分成若干个测试迭代（按照前面配置信息中所打的，每秒一次迭代）。
    * 先是预热过程，逐个打出各次迭代的信息。这个过程中，我们一般能看到方法的执行速度一路提升再到稳定的过程
    * 然后是正式执行，这时它所采集的信息将会用于最后结果的统计
* 一次测试完成之后，我们会看到该次测试的一个结果汇总，包含`均值`、`最大值`、`最小值`、`标准偏差`、`99.9%置信区间`这些信息。
* 最后再打印一个格式良好的报告，说明本次测试的结果。

这就是一个简单压测的完整过程，我们可以看到，我们只需要做非常少量的工作即可完成压测，这对于压测任务简单化有非常大的帮助。

## benchmark方法参数

前面我们看到的JMH的benchmark方法是无参的，其实它是可以支持参数的。测试的很多场景都非常需要这个功能的，比如我们测试过程中的每次迭代中的所有benchmark方法的执行需要使用同一个对象，这时这个对象就很适合在外部统一初始化，再用参数的方式传进去。

> 不过它对参数有要求，必需是注有`@State`注解类型的实例或者JMH内部参数。

### 带有`@State`注解的自定义类型

如果benchmark方法需要使用我们自定义的类型作为参数，那我们需要在这个类型上加上`@State`注解

```java
@State(Scope.XXXX)
```

当我们自定义的类型带有这个注解，JMH就知道需要什么时候创建一个的实例，执行时对benchmark方法的各次调用怎么共享这个实例。`Scope`有3个值

* `Scope.Benchmark`它是为整个bencharm只创建一个实例，然后所有线程执行benchmark方法时，都使用同一个实例做为参数。比如我们需要压测多线程共同操作`AtomicInteger`的性能，这时我们就会使用`Scope.Benchmark`，代码如下：
    
    ```java
    @State(Scope.Benchmark)
    public static class AtomicIntegerProvider {
        AtomicInteger integer = new AtomicInteger();
    }
    
    @Benchmark
    public void multiThreadIncrAtomicInteger(AtomicIntegerProvider provider) {
        provider.integer.incrementAndGet();
    }
    ```
    
* `Scope.Thread`它会为每个线程创建一个实例。比如当我们需要测一测`AtomicInteger`在没有竞争的时候的性能，这时我们就会用`Scope.Thread`为每个线程创建一个`AtomicInteger`，以达到没有竞争的效果。代码如下：
    
    ```java
    @State(Scope.Thread)
    public static class AtomicIntegerProvider {
        AtomicInteger integer = new AtomicInteger();
    }
    
    @Benchmark
    public void singleThreadIncrAtomicInteger(AtomicIntegerProvider provider) {
        provider.integer.incrementAndGet();
    }
    ```
    
    有些代码多线程同时访问有并发问题，会导致结果的不正确，测他们就需要使用`Scope.Thread`，比如`DateFormat`，`Random`, `StringBuilder`这些类。
    
* `Scope.Group`

### 自定义类的`@Setup`与`@TearDown`

带有`@State`注解的类型有时候需要做一些初始化和销毁的工作，比如MySQL建立连接。这时我可以在成员方法上加`@Setup`和`@TearDown`注解来实现，它们的功能与`JUnit`中的`Before`和`After`很像，会在测试任务执行前后执行。

比如我们要对MySQL数据压的性能进行一次压测，那么就会写如下一段代码：


```java
public class MySQLBenchmark {

    @State(Scope.Thread)
    public static class ConnectionProvider {
        Connection conn;

        @Setup(Level.Trial)
        public void init() throws ClassNotFoundException, SQLException {
            Class.forName("com.mysql.jdbc.Driver");
            conn = DriverManager.getConnection("jdbc:mysql://localhost");
        }

        @TearDown(Level.Trial)
        public void destroy() throws SQLException {
            conn.close();
        }
    }

    @Benchmark
    public void test(ConnectionProvider provider) throws SQLException {
        try (final PreparedStatement statement = provider.conn.prepareStatement("SELECT * FROM test WHERE id = ?")) {
            statement.setInt(1, 1);
            try (final ResultSet resultSet = statement.executeQuery()) {
                if (resultSet.first()) {
                    resultSet.getString("value");
                }
            }
        }
    }
}
```

此时，JMH会在所有的测试执行之前先初始化一个`ConnectionProvider`实例，并调用它的`init()`方法进行初始化，然后用这个实例调用benchmark方法进行压测，所有压测都完成之后，再调用`destroy()`方法销毁client。

```plantuml
participant "init thread" as init
create "ConnectionProvider" as provider
init -> provider
init -> provider: init
activate provider
deactivate provider
==start to test==
participant "Benchmark Thread" as t1
t1 -> provider: conn.query
...all threads run benchmark with the same provider instance...
t1 -> provider: conn.query
==test finished, tear down==
"destroy thread" -> provider: destroy
activate provider
deactivate provider
```

按之前描述，JMH的性能测试中执行的对象有不同粒度的概念：

* `测试`（`Trial`）一个benchmark从启动到出结果数据为一次测试
* `迭代`（`Iteration`）一个测试中包含多次迭代，一般1秒一次迭代，每次迭代都会分别记数，最后用于统计
* `调用`（`Invocation`）执行一次benchmark方法为一次调用，一次迭代中会多次调用benchmark方法

这样的每个粒度前后都可以增加`@Setup`和`@TearDown`的方法，只需要为这些方法设置相应的`Level`参数即可，每个注解的方法也都可以出现多次，`@Setup`与`@TearDown`没有配对的关系。

* `@Setup(Level.Trial)`只会在整个测试开始之前执行一次。
* `@TearDown(Level.Trial)`只会在整个测试完成时都执行一次。
* `@Setup(Level.Iteration)`会在每次迭代开始时都执行一次。
* `@TearDown(Level.Iteration)`会在每次迭代完成时都执行一次。
* `@Setup(Level.Invocation)`会在每次benchmark方法调用开始时都执行一次。
* `@TearDown(Level.Invocation)`会在每次benchmark方法调用完成时都执行一次。

> `Level.Invocation`要慎用，因为它对测试结果的准确性可能会影响很大。

下面这张图就展示了一个Iteration级的`@Setup`与`@TearDown`的工作方式

```plantuml
participant "init thread" as init
create provider
init -> provider

== iteration 1 ==
init -> provider: init
activate provider
deactivate provider

"Benchmark thread" as t1 -> provider: benchmark
...
t1 -> provider: benchmark

"destroy thread" as destroyer -> provider: destroy
activate provider
deactivate provider

== iteration 2 ==
...all iterations...
== iteration 10 ==

init -> provider: init
activate provider
deactivate provider

t1 -> provider: benchmark
...
t1 -> provider: benchmark

destroyer -> provider: destroy
activate provider
deactivate provider

== all iterations finished, test end ==
```

### JMH参数

一般的benchmark测试的方式是将benchmark打成jar包，然后在一个无干扰的环境下用命令行启动测试。但是有些信息在测试代码打成jar包前是不知道的，或者是不便写在代码中的。这些信息可以通过在执行的时候用参数传进去。比如我们要压测MySQL，那么MySQL的服务器地址，用户名密码这些信息，就可以在运行时传进去。相应的代码中只需要在带有`@State`注解的类中字段上加上`@Param`注解即可。

```java
@State(Scope.Thread)
public static class ConnectionProvider {
    @Param("localhost")
    String mysqlServer;
    @Param("root")
    String mysqlUser;
    @Param("password")
    String mysqlPassword;
    
    Connection conn;
    
    @Setup(Level.Trial)
    public void init() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql://" + mysqlServer, mysqlUser, mysqlPassword);
    }
    
    @TearDown
    public void destroy() throws SQLException  {
        conn.close();
    }
}
```

它执行的时候结果如下


```
Benchmark              (mysqlPassword)   (mysqlServer)  (mysqlUser)    Mode       Cnt      Score     Error  Units
MySQLBenchmark.test           password       localhost         root   thrpt       200  56414.767 ± 893.774  ops/s
```

我们可以看到报告中多了几列括号`()`括起来的列，它们就是这里上面代码中设置的参数，列名就是带`@Param`注解的字段名。这些字段其实可以在执行`benchmarks.jar`的时候再指定进去的，详细情况见[benchmark启动方式](#benchmark启动方式)。这种就特别适合写代码与环境准备并发进行。

## 性能比较

掌握了以上的内容，我们就可以用`JMH`来做很多性能测试了，包括测一些简单的工具类，到测一个复杂的服务或者系统。只是一般一个测试只能得到一个数据，而面对一个绝对数据，我们并不能知道是好还是不好。比如在某个特定的测试环境下，调用某工具类的一个方法的OPS是`1,000,000`，我们还是不知道这是值得欢欣鼓舞的，还是需要忧虑性能问题。一个知道性能好坏的方法就是——比较。

比如我们不知道正则表达式的编译对我们会造成多大的影响，那么我们就可以写一组对比的压测（预编译vs每次编译）：

```java
public static final String IP_PATTERN = "[0-9]{1,3}(\\.[0-9]{1,3}){3}";

@State(Scope.Benchmark)
public static class PatternProvider {
    Pattern pattern;

    @Setup(Level.Trial)
    public void init() {
        pattern = Pattern.compile(IP_PATTERN);
    }
}

@Benchmark
public void preCompiled(PatternProvider provider) {
    provider.pattern.matcher("1.2.3.4").matches();
}

@Benchmark
public void matchDirectly() {
    Pattern.matches(IP_PATTERN, "1.2.3.4");
}
```

我们将会看到它执行的结果，如下


```
Benchmark                      Mode  Cnt         Score         Error  Units
RegExBenchmark.matchDirectly  thrpt    5   1782915.260 ±   22102.033  ops/s
RegExBenchmark.preCompiled    thrpt    5   6093182.290 ±  172088.745  ops/s
```

我们可以看到用正则表达式匹配时，预编译的性能是临时编译的`3.42`倍，这样编译的影响有多大就一目了然了。有比较才有优劣，比较结果一般对于我们选择架构的设计，代码的写法等方面，具有非常重要的指导意义。我们需要做的就是针对我们的使用方式，设计合理的对照实验。

## benchmark启动方式

JMH中有一个`Runner`类，Benchmark的执行是由`Runner.run()`方法启动的。所以理论上只要能启动`Runner.run()`方法，我们就能启动benchmark。一般有3种方式

1. 直接编写main方法启动执行，如下，这种方式适合在编写测试代码时使用

    ```java
    public static void main(String[] args) throws Exception {
        Options options = new OptionsBuilder()
                .include(NumberTestBenchmark.class.getName())
                .build();
                
        new Runner(options).run();
    }
    ```

2. 先用maven打包得到一个`benchmarks.jar`，然后用`java -jar benchmarks.jar`的命令执行这些测试用例。（实际是执行JMH自带的主类`org.openjdk.jmh.Main`）

    这个主类提供了从命令行参数设置benchmark信息的手段，功能很强大，所以benchmark一般直接使用这个主类（官方archetype生成的工程就会把这个类设置成主类）。比如可以通过`-f <n>`设置benchmark启动的子进程次数，`-i <n>`设置测试的迭代次数等等，具体的可以通过`-h`参数查看所有选项。不过有几点常用的控制项
    
    参数格式是
    
    ```
    java -jar benchmarks.jar match [options]
    ```
    
    其中的`match`是正则表达式，它会把所有的`benchmark`名字都用这里指定的正责表达式去匹配，如果能查找到匹配的子串，那么相应的benchmark就会被执行。这个功能特别合适在包含了很多benchmark的jar包中执行某一个特定的测试。
    
    具体一个benchmarks的jar包中包含了哪些jar包，可以用`-l`参数查看，比如
    
    ```
    Benchmarks: 
    benchmark.demo.RegExBenchmark.matchDirectly
    benchmark.demo.RegExBenchmark.preCompiled
    benchmark.demo.MySQLBenchmark.test
    ```
    
    还可以用`-lp`参数查看，不仅可以查看benchmark列表，还可以查看相应的参数
    
    ```
    Benchmarks: benchmark.demo.RegExBenchmark.matchDirectly
    benchmark.demo.RegExBenchmark.preCompiled
    benchmark.demo.MySQLBenchmark.test
      param "mysqlServer" = {localhost}
      param "mysqlPassword" = {root}
      param "mysqlUser" = {root}
    ```
    
    上述的都是一些查看`benchmarks.jar`中内容的参数。当需要真正执行`benchmarks.jar`的时候，有一堆参数可以控制执行中的种种行为，比如起多少个进程，多少个线程，预热多少次迭代，实际测试跑多少次迭代，结果输出为什么格式，输出来什么地方等等，这些都可以用`-h`参数查看。

3. 安装JMH插件，此时在IDE中就可以像执行单元测试那样执行这个Benchmark。它实际的实现机制就是调用`org.openjdk.jmh.Main`，可以用的参数与第`2`种方式相同


