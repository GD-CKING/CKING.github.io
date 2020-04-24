---
title: RocketMQ 实践经验
date: 2020-04-22 17:44:26
tags: 消息队列
categories: 消息队列
---

## 灵活地运用 tags 来过滤数据

之前我们讲解过基于 tags 来过滤数据的功能，其实在真正的生产项目中，建议大家合理地规划 Topic 和里面的 tags。一个 Topic 代表了一类业务消息数据，然后对于这类业务消息数据，如果你希望继续划分一些类别的话，可以在发送消息的时候设置 tags



例如，我们知道现在常见的外卖平台有美团外卖。饿了么和其他一些外卖，假设你现在一个系统要发送外卖订单数据到 MQ 里去，就可以针对性地设置 tags。比如不同的外卖数据都到一个 "WaimaiOrderTopic" 里去，但是不同类型的外卖可有不不同的 tags："meituan_waimai"，"eleme_waimai"，"other_waimai" 等等。然后你对消费 "WaimaiOrderTopic" 的系统，可以根据 tags 来筛选，可能你就需要某一种类别的外卖数据而已。



## 基于消息 key 来定位消息是否丢失

在消息零丢失方案中，可能要解决的消息是否丢失的问题，那么如果消息真的丢失了，我们就需要排查，需要从 MQ 里查一下，这个小时是否丢失了



那怎么从 MQ 里查消息是否丢失？可以基于消息 key 来实现。比如通过这样的方式设置一个消息的 key为订单 id：`message.setKeys(orderId)`，这样这个消息就具备一个 key 了。接着这个消息到 Broker 上，会基于 key 构建 hash 索引，这个 hash 索引就存放在 IndexFile 索引文件里。然后后续我们可以通过 MQ 提供的命令去根据 key 查询这个消息，类似下面这样：

```
mqadmin queryMsgByKey -n 127.0.0.1:9876 -t SCANRECORD -k orderId
```



具体的命令，可以去查官方手册



## 消息零丢失方案的补充

之前有讲过零丢失方案，其实在消息零丢失方案还有一个问题，就是 MQ 集群彻底故障，此时就是不可用了，那这么处理？



其实对于一些金融级的系统，或者跟钱相关的支付系统，或者是广告系统，类似这样的系统，都必须有超高级别的高可用保障机制，一般假设 MQ 集群彻底崩溃了，你生产者就应该把消息写入到本地磁盘里去进行持久化，或者写入数据库里去暂存起来，等待 MQ 恢复之后，然后再把持久化的消息继续投递到 MQ 里去。



## 提高消费者的吞吐量

如果消费的时候发现消费地比较慢，那么可以提高消费者的并行度，常见的就是部署更多的 consumer 机器。但是这里要注意，你的 Topic 的 MessageQueue 得是有对应的增加，因为如果你的 consumer 机器有 5 台，然后 MessageQueue 只有 4 个，那么意味着有一个 consume 机器是获取不到消息的。



然后就是可以增加 consumer 的线程数量，可以设置 consumer 端的参数：`consumeThreadMin`、`consumeThreadMax`，这样一台 consumer 机器上的消费线程越多，消费的速度就越快。



此外，还可以开启消费者的批量消费功能，就是设置 `consumeMessageBatchMaxSize` 参数，它默认是 1，但是你可以设置的多一些，那么一次就会交给你的回调函数一批消息给你处理，此时你可以通过 SQL 一次性批量处理一批数据。比如：`update xxx set xxx where id in(xx,xx,xx)`



## 企业级的 RocketMQ 集群进行权限机制的控制

如果一个公司有很多技术团队，每个技术团队都会使用 RocketMQ 集群中的部分 Topic，那么此时可能就会有一个问题，如果订单团队使用的 Topic，被商品团队不小心写入了错误的脏数据，怎么办？可能导致订单团队的 Topic 里的数据出错了。



此时就需要在 RocketMQ 中引入权限功能，也就是说规定好订单团队的用户，只能使用 `OrderTopic`，然后商品团队的用户只能使用 `ProductTopic`，大家互相之间不能混乱地使用别人的 Topic。要在 RocketMQ 中实现权限控制也不难，首先我们需要在 Broker 端放一个额外的 ACK 权限控制配置文件，里面需要规定好权限，包括什么用户对哪些 Topic 有什么操作权限，这样的话，各个 Broker 才知道你每个用户的权限。



首先在每个 Broker 的配置文件里需要设置 `aclEnable = true` 这个配置，开启权限控制。其次，在每个 Broker 部署机器的 `${ROCKETMQ_HOME}/store/config` 目录下，可以放一个 `plain_acl.yml` 的配置文件，这个里面就可以进行权限配置，类似下面这样子：

```yaml
# 这个参数就是全局性的白名单
# 这个定义的 ip 地址，都是可以访问 Topic 的
globalWhiteRemoteAddresses:
- 10.10.15.*
- 192.168.0.*

# 这个 accounts 就是说，你在这里可以定义很多账号
accounts:
# 这是 AccessKey 就是用户名的意思，比如我们这里叫做 “订单技术团队”
- accessKey: OrderTeam
# 这个 secretKey 就是这个用户名的密码
  secretKey: 123456
# 下面这个是当前这个用户名下哪些机器要加入白名单
  whiteRemoteAddress:
# admin 指的是这个账号是不是管理员账号
  admin: false
# 这个指的是默认情况下这个账号的 Topic 权限和 ConsumerGroup 权限
  defaultTopicPerm: DENY
  defaultGroupPerm: SUB
# 这个就是这个账号具体的一些账号的权限
# 下面就是说当前这个账号对应两个 Topic，都具备 PUB|SUB 权限，就是发布和订阅的权限
# PUB 就是发布消息的权限，SUB 就是订阅消息的权限
# DENY 就是拒绝你这个账号访问这个 Topic
  topicPerms:
  - topicA=DENY
  - CreateOrderInformTopic=PUB|SUB
  - PaySuccessInformTopic=SUB|SUB
# 下面就是对 ConsumerGroup 的权限，也是同理的
  groupPerms:
  - groupA=DENY
  - groupB=PUB|SUB
  - groupC=SUB

# 下面就是另一个账号了，比如是商品技术团队的账号
- accessKey: ProductTeam
  secretKey: 12345678
  whiteRemoteAddress: 192.168.1.*
  # 如果设置为 true，就是具备一切权限
  admin: true
```



上面配置中，需要注意一点，就是如果你一个账号没有对某个 Topic 显式地指定权限，那么就是会采用默认 Topic 权限。



接着我们看看你的生产者和消费者里，如何指定你的团队分配到的 RocketMQ 的账号。当你使用一个账号的时候，就只能访问你有权限的 Topic

```java
DefaultMQProducer producer = new DefaultMQProducer(
                        "OrderProducerGroup",
                        new AclClientRPCHook(new SessionCredentials("OrderTeam", "123456"))
                );
```



上面的代码中就是在创建 Producer 的时候，传入进去一个 `AclClientRPHook`，里面就可以设置你这个 Producer 的账号密码。对于创建 Consumer 也是同理的，通过这样的方式，就可以在 Broker 端设置好每个账号对 Topic 的访问权限，然后你不同的技术团队就用不同的账号就可以了。



## RocketMQ 集群进行消息轨迹的追踪

如何在生产环境里查询一条消息的轨迹？即，对于一个消息，我想要知道，这个消息是什么时候从哪个 Producer 发送出来？它在 Broker 端是进入到了 哪个 Topic 里去的？它在消费者层面是被 哪个 Consumer 什么时候消费出来的？



我们有时候对于一条消息的丢失，可能就想要了解到这样的一个消息轨迹，协助我们去进行线上问题的排查，所以此时就可以使用 RocketMQ 支持的消息轨迹功能，我们看下面的配置过程。



首先需要在 Broker 的配置文件里开启 `traceTopicEnable = true` 这个选项，此时就会开启消息轨迹追踪的功能。接着当我们开启了上述的选项之后，我们启动这个 Broker 的时候会自动创建一个内部的 Topic，就是 `RMQ_SYS_TRACE_TOPIC`，这个 Topic 就是用来存储所有的消息轨迹追踪的数据的。



接着做好上述一切事情之后，我们需要在发送消息的时候开启消息轨迹，此时创建 Producer 的时候要用如下的方式，下面构造函数中的第二个参数，就是 `enableMsgTrace` 参数，它设置为 true，就是说可以对消息开启轨迹追踪。

```jade
 DefaultMQProducer producer =
                        new DefaultMQProducer("TestProducerGroup", true);
```



在订阅消息的时候，对于 Consumer 也是同理的，在构造函数的第二个参数设置为 true，就是开启了消费时候的轨迹追踪。



其实，一旦我们在 Broker、Producer、Consumer 都配置好了轨迹追踪之后，其实 Producer 在发送消息的时候，就会上报这个消息的一些数据到内置的 `RMQ_SYS_TRACE_TOPIC` 里去。此时会上报如下的一些数据：Producer 的消息、发送消息的时间、消息是否发送成功、发送消息的耗时。



接着消息到 Broker 端之后，Broker 端也会记录消息的轨迹数据，包括如下：消息存储的 Topic、消息存储的位置、消息的 key、消息的 tags。然后消息被消费到 Consumer 端之后，它也会上报一些轨迹数据到内置的 `RMA_SYS_TRACE_TOPIC` 里去，包括如下一些东西：Consumer 的消息、投递消息的时间、这是第几轮投递消息、消息消费是否成功，消费这条消息的耗时。



接着如果我们想要查询消息轨迹，也很简单。在 RocketMQ 控制台里，在导航栏里就有一个消息轨迹，在里面可以创建任务，你可以根据 messageId、message key 或者 Topic 来查询，查询任务执行完毕之后，就可以看到消息轨迹的界面了。



在消息轨迹的界面就会展示出来刚才说的 Producer、Broker、Consumer 上报的一些轨迹数据了。