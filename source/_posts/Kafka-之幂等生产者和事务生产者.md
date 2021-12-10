---
title: Kafka 之幂等生产者和事务生产者
date: 2021-12-06 18:18:45
tags: Kafka
categories: Kafka
---

### 幂等性 Producer



在 Kafka 中，Producer 默认不是幂等的，但我们可以创建幂等性 Producer。它是 0.11.0.0 版本引入的新功能。指定 Producer 幂等性的方法很简单，仅需要设置一个参数即可，即 props.put("enable.idempotence", true) 或 props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true)



enable.idempotence 被设置成 true 后，Producer 自动升级成幂等性 Producer，其他所有的代码逻辑都不需要改变，Kafka 自动帮你做消息的重复去重。底层具体的原理很简单，就是经典的用空间去换时间的优化思路，即在 Broker 端多保存一些字段。当 Producer 发送了具有相同字段值的消息后，Broker 能够自动知晓这些消息已经重复了，于是可以在后台默默丢弃掉。当然，实际的实现原理并没有那么简单，你可以大致这么理解



但是，幂等性的 Producer 也是有局限性的。首先，它只能保证单分区上的幂等性，即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。其次，它只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了



那如果要实现多分区以及多会话上的消息无重复，应该怎么做？答案就是事务或者依赖事务型 Producer。这也是幂等性 Producer 和事务型 Producer 的最大区别



### 事务型 Producer



事务型 Producer 能够保证将消息原子性地写入到多个分区中。这批消息要么全部写入成功，要么全部失败。另外，事务型 Producer 也不惧进程的重启。Producer 重启回来后，kafka 依然保证它们发送消息的精确一次处理



设置事务型 Producer 的方法也很简单，满足两个条件即可：



- 和幂等性 Producer 一样，开启 enable.idempotence = true
- 设置 Producer 端参数 transactional.id。最好设置一个有意义的名字



此外，你还需要在 Producer 代码中做一些调整，如下：



```java
producer.initTransactions();
try {
            producer.beginTransaction();
            producer.send(record1);
            producer.send(record2);
            producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```



和普通 Producer 代码相比，事务型 Producer 的显著特点是调用了一些事务 API，如 `initTransaction`、`beginTransaction`、`commitTransaction` 和 `abortTransaction`，它们分别对应事务的初始化、事务开始、事务提交以及事务终止



这段代码能够保证 Record1 和 Record2 被当做一个事务统一提交到 Kafka，要么它们全部提交成功，要么全部写入失败。实际上即使写入失败，Kafka 也会把他们写入到底层的日志中，也就是说 Consumer 还是会看到这些消息。因此在 Consumer 端，读取事务型 Producer 发送的消息也是需要一些变更的。修改起来也简单，设置 `isolation.level` 参数的值即可。当前这个参数有两个取值：



1. **read_uncommitted**：这是默认值，表明 Consumer 能够读取到 Kafka 写入的任何消息，不论事务型 Producer 提交事务还是终止事务，其写入的消息都可以读取。显然，如果你使用了事务型 Producer，那么对应的 Consumer 就不要使用这个值
2. **read_committed**：表明 Consumer 只会读取事务型 Producer 成功提交事务写入的消息。当然，它也能看到非事务型 Producer 写入的所有消息



相比较幂等性 Producer，事务型 Producer 的性能要更差，在实际使用过程中，我们需要仔细评估引入事务的开销，不可无脑地启用事务