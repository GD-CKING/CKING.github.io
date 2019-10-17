---
title: JVM中的一些参数介绍
date: 2019-09-16 11:39:52
tags: JVM
categories: JVM
---

## JVM配置常用参数

### 堆参数

| 参数              | 描述                                                        |
| ----------------- | ----------------------------------------------------------- |
| -Xms              | 设置JVM启动时堆内存的初始化大小                             |
| -Xmx              | 设置堆内存最大值                                            |
| -Xmn              | 设置年轻代的空间大小，剩下的为老年代的空间大小              |
| -XX:PermGen       | 设置永久代内存的初始化大小（JDK1.8开始废弃了永久代）        |
| -XX:MaxPermGen    | 设置永久代的最大值                                          |
| -XX:SurvivorRatio | 设置Eden区和Survivor区的空间比例：Eden/S0 = Eden/S1 默认为8 |
| -XX:NewRatio      | 设置老年代与年轻代的比例大小，默认值为2                     |

### 回收器参数

| 参数                    | 描述                                                         |
| :---------------------- | :----------------------------------------------------------- |
| -XX:+UseSerialGC        | 串行，Young区和Old区都使用串行，使用复制算法回收，逻辑简单高效，无线程切换开销 |
| -XX:+UseParallelGC      | 并行，Young区：使用Parallel scavenge回收算法，会产生多个线程并行回收。通过-XX:ParallelGCThreads=n参数指定有线程数，默认是CPU核数；Old区：单线程 |
| -XX:+UseParallelOldGC   | 并行，和UseParallelGCC一样，Young区和Old区的垃圾回收时都使用多线程手机 |
| -XX:+UseConcMarkSweepGC | 并发，短暂停顿的并发收集。Young区：可以使用普通的或者parallel垃圾回收算法，由参数 -XX:+UseParNewGC来控制；Old区：只能使用Concurrent Mark Sweep |
| -XX:+UseG1GC            | 并行的、并发的和增量式压缩短暂停顿的垃圾收集器。不区分Young区和Old区空间。它把堆空间划分为多个大小相等的区域。当进行垃圾收集时，它会优先收集存活对象较少的区域，因此叫“Garbage First” |

​		如上表所示，目前主要有**串行、并行和并发**三种。对于大内存的应用而言，串行的性能太低，因此使用到的主要是并行和并发两种。并行和并发GC的策略通过UseParallelGCC和UseCon从MarkSweepGC来指定，还有一些细节的配置参数用来配置策略的执行。例如：XX:ParallelGCThreads、XX:CMSInitiatingOccupancyFraction等。通常：Young区对象回收只可选择并行（耗时间），Old区选择并发（耗CPU）。

### 项目中常用配置

| 参数设置                              | 描述                                     |
| ------------------------------------- | ---------------------------------------- |
| -Xms4800m                             | 初始化堆空间大小                         |
| -Xmx4800m                             | 最大堆空间大小                           |
| -Xmn1800m                             | 年轻代的空间大小                         |
| -Xss512k                              | 设置线程栈空间大小                       |
| -XX:PermSize=256m                     | 永久区空间大小（jdk1.8开始废弃了永久代） |
| -XX:MaxPermSize=256m                  | 最大永久区空间大小                       |
| -XX:+UseStringCache                   | 默认开启，启用缓存常用的字符串           |
| -XX:+UseConcMarkSewwpGC               | 老年代使用CMS收集器                      |
| -XX:UseParNewGC                       | 新生代使用并行收集器                     |
| -XX:ParallelGCThreads=4               | 并行线程数量4                            |
| -XX:+CMSClassUnloadingEnabled         | 允许对类的元数据进行清理                 |
| XX:+DisableExplicitGC                 | 禁止显示的GC                             |
| -XX:+UseCMSInitiatingOccupancyOnly    | 表示只有达到阈值之后才进行CMS回收        |
| -XX:CMSInitiatingOccupancyFraction=68 | 设置CMS在老年代回收的阈值为68%           |
| -verbose:gc                           | 输出虚拟机GC详情                         |
| -XX:+PrintGCDetails                   | 打印GC详情日志                           |
| -XX:+PrintGCDataStamps                | 打印GC的耗时                             |
| -XX:+PrintTenuringDistribution        | 打印Tenuring年龄信息                     |
| -XX:+HeapDumpOnOutOfMemoryError       | 当抛出OOM时进行HeapDump                  |
| -XX:HeapDumpPath=/home/admin/logs     | 指定HeapDump的文件路径或目录             |

### 常用组合

| Young                    | Old                 | JVM Options                                   |
| ------------------------ | ------------------- | --------------------------------------------- |
| Serial                   | Serial              | -XX:+UseSerialGC                              |
| Parallel scavenge        | Parallel Old/Serial | -XX:+UseParallelGC<br />-XX:+UseParallelOldGC |
| Serial/Parallel scavenge | CMS                 | -XX:+UseParNewGC<br />-XX:+UseConcMarkSweepGC |
| G1                       |                     | -XX:+UseG1GC                                  |

## 常用GC调优策略

### GC调优原则

​		在调优之前，我们需要记住下面的原则：

- 多数的Java应用不需要在服务器上进行GC优化。
- 多数导致GC问题的Java应用，都不是因为我们参数设置错误，而是代码问题。
- 在应用上线之前，先考虑将机器的JVM参数设置到最优（最适合）。
- 减少创建对象的数量。
- 减少使用全局变量和大对象。
- GC优化是到最后不得已才 采用的手段。
- 在实际使用中，分析GC情况优化代码比优化GC参数要多得多。

### GC调优目的

- 将转移到老年代的对象数量降低到最小。
- 减少GC的执行时间

### GC调优策略

- **策略1**：将新对象预留在新生代，由于Full GC的成本远高于Minor GC，因此尽可能将对象分配在新生代是明智的做法，实际项目中根据GC日志分析新生代空间大小分配是否合理，适当通过“-Xmn”命令调节新生代大小，最低限度降低新对象直接进入老年代的情况。
- **策略2**：大对象进入老年代，虽然在大部分情况下，将对象分配在新生代是合理的。但是对于大对象这种做法却值得商榷，大对象如果首次在新生代分配可能会出现空间不足导致很多年龄不够的小对象被分配到老年代，破坏新生代的对象结构，可能会出现频繁的full gc。因此，对于大对象，可以设置直接进入老年代（当然短命的大对象对于垃圾回收来说就是噩梦）。-`XX:PretenureSizeThreshold`可以设置直接进入老年代的对象大小。
- **策略3**：合理设置进入老年代对象的年龄，-XX:MaxTenuingThreshold设置对象进入老年代的年龄大小，减少老年代的内存占用，降低full GC发送的频率。
- **策略4**：设置稳定的堆大小，堆打下设置有两个参数：-Xms初始化堆大小，-Xms最大堆大小。
- **策略5**：如果满足下面的指标，则一般不需要进行GC优化：
  - MinorGC执行时间不到50ms
  - MinorGC执行不频繁，约10秒一次
  - Full GC执行时间不到1s
  - Full GC执行频率不算频繁，不低于10分钟1次。

## 参考资料

[Java应用如何调优](https://mp.weixin.qq.com/s/2p0Pw6rB2onQDDUQL8kcGA)

