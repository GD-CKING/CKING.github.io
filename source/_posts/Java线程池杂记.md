---
title: Java线程池杂记
date: 2019-12-27 14:26:27
tags: Java
categories: Java
---

## 简单说一下Java线程池的底层工作原理

​		一般情况下，系统是不会说无限制地创建大量的线程，会构建一个线程池，保持一定数量的线程，让他们执行各种各样的任务，线程执行完任务后，不要销毁自己，继续去等待执行下一个任务。这样可以避免频繁地创建线程和销毁线程。

```java
  ExecutorService threadPool = Executors.newFixedThreadPool(3);
  threadPool.submit(new Callable<Object>() {

       @Override
       public Object call() throws Exception {
           return null;
       }
  });
```

​		大概流程是这样的：提交任务，先看一下线程池里的线程数量是否小于corePoolSize，也就是上面代码的3。如果小于，直接创建一个线程出来执行你的任务；执行完任务之后，这个线程是不会死掉的，它会尝试从一个无界的LinkedBlockingQueue里获取新的任务，如果没有新的任务，此时就会阻塞住，等待新的任务到来。

​		应用持续提交任务，上述流程反复执行，只要线程池的线程数量小于corePoolSize，都会直接创建新线程来执行这个任务，执行完了就尝试从无界队列里获取任务，知道线程里有corePoolSize个线程；接着再次提交任务，会发现线程数量已经跟corePoolSize一样大了，此时就会直接把任务放入队列中就可以了，线程会争取获取任务执行。如果所有人的线程此时都在执行任务，那么无界队列里的任务就可能会越来越多。

![线程池](Java线程池杂记/线程池.png)

## 线程池的核心配置参数

​		上面的那段代码：`Executors.newFixedThreadPool(3);`，进去里面查看源码，是这样子的：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

​		创建一个线程池就是这样子的。除了那个0L，参数依次是corePoolSize、maximumPoolSize，keepAliveTime，queue。如果你不用fixed之类的线程池，可以自己通过这个构造函数创建自己的线程池。

```
corePoolSize: 3
maximumPoolSize: 200
keepAliveTime: 60s
new ArrayBlockingQueue<Runnable>(200)
```

​		如果你把queue做成有界队列，比如上面的new ArrayBlockingQueue<Runnable>(200)，假设corePoolSize个线程都在繁忙地工作，大量的任务进入有界队列，队列满了，如果你的maximumPoolSize是比corePoolSize大的，此时会继续创建额外的线程放入线程池里，来处理这些任务，然后超过corePoolSize数量的线程如果处理完了一个任务也会尝试从队列里去获取任务来执行。

​		如果额外线程都创建完了去处理任务了，队列还是满了，此时还有新的任务，那该怎么办？只能reject掉。目前有几种reject策略，可以传入RejectExecutionHandler

- AbortPolicy

- DiscardPolicy

- DiscardOldestPolicy

- CallerRunsPolicy

- 自定义

​		如果后续队列里，慢慢没有任务了，线程空闲了，超过corePoolSize的线程会自动释放掉，在keepAliveTime之后就会释放。在具体场景中，我们可以根据上述原理定制自己的线程池，来考虑corePoolSize的数量、队列类型、最大线程数量、拒绝策略和线程释放时间等等。一般常用的是fixed线程。