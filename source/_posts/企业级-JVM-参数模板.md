---
title: 企业级 JVM 参数模板
date: 2020-02-16 16:19:03
tags: JVM
categories: JVM
---

在一些微小型创业公司中，虽然有少数几个比较好的架构师，但是架构师往往没那么大精力把控到特别细节的地方。如果一些一线普通工程师对 JVM 那么快没有那么的精通，在开发完一个系统之后，部署生产环境的时候没有对 JVM 进行什么参数设置的时候，可能很多时候就是用一些默认的 JVM 参数。



默认的 JVM 参数绝对是系统负载逐渐增高的时候一个最大的问题。如果你不设置  `-Xms`、`-Xmx` 之类的堆内存大小的话，你启动一个系统，可能默认就给你几百 MB 的堆内存大小，新生代和老年代可能都是几百 MB 的样子。



新生代内存过小，会导致 Survivor 区域内存过小，同时 Eden 区域也很小。Eden 区域过小，自然会频繁地触发 Young GC，Survivor 区域过小，自然会导致经常在 Young GC 之后存活对象其实也没多少，但就是 Survivor 区域放不下。此时必然会导致对象经常进入老年代中，因此也必然会导致老年代过一段时间就被放慢，然后就会触发 Full GC。



Full GC 一般在正常情况下，都是以天为单位发生的，比如每天发生一次，或者是几天发生一次 Full GC。要是每小时都发生几次 Full GC，那么就会导致系统每小时都卡顿几次，这个时候肯定是不行的。在大部分工程师都对 JVM 优化不是很精通的情况下，通过推行一个 JVM 参数模板，可以让各个系统短时间内迅速就优化了 JVM 的性能。



## 企业级的 JVM 参数模板

假设在一台 4 核 8 G 的机器上部署项目，那么我们的 JVM 参数可以这么设置：

```verilog
-Xms4096M -Xmx4096M -Xmn3072M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -XX:+UseParNewGc -XX:+UseConMarkSweepGC -XX:CMSInitiatingOccupancyFraction=92 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0
```



为什么要这样定制 JVM 参数模板？首先，8G 的机器上给 JVM 堆内存分配 4G 就差不多了。毕竟可能还有其他进程会使用内存，一般别让 JVM 堆内存把机器内存给占满。然后年轻代给到 3G，之所以给到 3G 的内存时间，就是因为让年轻代尽量大一些，进而让每个 Survivor 区域都打到 300MB 左右。



根据当时对公司某个业务系统的分析，假设使用默认的 JVM 参数，可能年轻代就几百 MB 的内存，Survivor 区域就几十 MB 的内存。那么每次垃圾回收过后存活对象可能会有几十 MB，这是因为在垃圾回收的一瞬间可能有部分请求没处理完毕，此时会有几十 MB 对象式存活的，很容易触发动态年龄判定规则，让部分对象进入老年代。



所以在分析过后，给年轻代更大内存空间，让 Survivor 空间更大，这样在 Young GC 的时候，这一瞬间可能有部分请求没处理完毕，有几十 MB 的存活对象，这个时候再几百 MB 的 Survivor 空间中，可以轻松放下，不会进入老年代。



不同的系统运行时的情况略有不同，但是基本上都是在每次 Young GC 过后存活几 MB ~ 几十 MB，所以此时在这个参数下，都可以抗住。



这里有几个参数要简单介绍一下。`-XX:CMSInitiatingOccupancyFraction=92` 是指 CMS 垃圾回收器，当老年代达到 92% 时，触发 CMS 垃圾回收。而 `-XX:+UseCMSCompactAtCollection -XX:+CMSFullGCsBeforeCompaction=0` 则表示每次 Full GC 后都整理一下内存碎片。否则如果每次 Full GC 过后，都造成老年代里很多内存碎片，那么必然导致下一次 Full GC 更快到来，因为内存碎片会导致老年代可用内存变少。



## 如何优化每次 Full GC 的性能

这里再介绍一下当时做优化调整的另外两个参数，这两个参数可以帮助优化 Full GC 的性能，把每次 Full GC 的时间进一步降低一些。一个参数是 `-XX:+CMSParallelInitalMarkEnable`，这个参数会在 CMS 垃圾回收器的 “初始标记” 的阶段开启多线程并发执行。



在初始标记阶段，是会进行 Stop the World 的，会导致系统停顿，所以这个阶段开启多线程并发之后，可以尽可能优化这个阶段的性能，较少 Stop the world 的时间。



另一个参数是 `-XX:+CMSScavengeBeforeRemark`，这个参数会在 CMS 重新标记之前阶段之前，先尽量执行一次 Young GC。因为 CMS 的重新标记也是会 Stop the  World 的，所以如果在重新标记之前，先执行一次 Young GC，就会回收掉一些年轻代里没有引用的对象。



所以如果提前先回收掉一些对象，那么在 CMS 重新标记阶段就可以少扫描一些对象，此时就可以提升 CMS 重新标记阶段的性能，较少它的消耗。



所以在 JVM 参数模板中，同样也加入了这两个参数：

```verilog
-Xms4096M -Xmx4096M -Xmn3072M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -XX:+UseParNewGc -XX:+UseConMarkSweepGC -XX:CMSInitiatingOccupancyFraction=92 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:CMSParallelInitalMarkEnable -XX:+CMSScavengeBeforeRemark
```



