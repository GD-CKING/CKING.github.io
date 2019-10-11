---
title: 消息队列之RocketMQ
date: 2019-10-10 14:33:59
tags: 消息队列
categories: 消息队列
---

​		RocketMQ是阿里开源并贡献给Apache基金会的一款分布式消息平台，具有低延迟。高性能和可靠性、万亿级容量和灵活的可伸缩性的特点。单机也可以支持亿级的消息堆积能力、单机写入TPS单实例越7万条/秒，单机部署3个Broker，最高可以跑到12万条/秒。

## RocketMQ的基本构成

​		整个RocketMQ消息系统主要由4个部分组成。

​		从中间件服务角度来看整个RocketMQ消息系统（服务端）主要分为：**NameSrv**和**Broker**两个部分。

### NameSrv

​		在RocketMQ分布式消息系统中，NameSrv主要提供两个功能：

- 提供服务发现和注册。这里主要是管理Broker，NameSrv接受来自Broker的注册，并通过心跳机制来检测Broker服务的健康性。
- 提供路由功能。集群（这里是指以集群方式部署的NameSrv）中的每个NameSrv都保存了Broker集群（这是是指以集群方式部署的Broker）中整个的路由信息和队列信息。这里需要注意，在NameSrv集群中，每个NameSrv都是相互独立的，所以每个Broker需要连接所有的NameSrv，每创建一个新的topic都要同步到所有的NameSrv上。

### Broker

​		主要是负责消息的存储、传递、查询以及高可用（HA）保证等。其由如下几个子模块构成：

- remoting：是Broker的服务入口，负责客户端的接入（Producer和Consumer）和请求处理。
- client：管理客户端和维护消费者对于Topic的订阅。
- store：提供针对存储和消息查询的简单的API（数据存储在物理磁盘）。
- HA：提供数据在主从节点间同步的功能特性。
- Index：通过特定的key构建消息索引，并提供快速的索引查询服务。

​        而从客户端的角度看主要有：**Producer**、**Consumer**两个部分。

### Producer

​		消息的生产者，由用户进行分布式部署，消息有Producer通过多种负载均衡模式发送到Broker集群，发送低延时，支持快速失败。

### Consumer

​		消息的消费者，也有用户部署，支持PUSH和PULL两种消费模式，支持集群消费和广播消费，提供实时的消息订阅机制，满足大多数消费场景。

## RocketMQ的其他概念

### Producer Group

​		相同角色的生产者被组织到一起。在事务提交后，生产组中不同实例都可以连接broker执行提交或回滚事务，以防原生产者在提交后就挂掉。

### Consumer Group

​		具有完全相同角色的消费者被组合在一起并命名为消费者组，消费群体是一个很好的概念，它在消费信息方面实现负载平衡和容错目标是非常容易的。另外，**消费者组的消费者实例必须具有完全相同的主题订阅。**

### Topic

​		主题是生产者提供消息和消费者提取消息的类别。主题与生产者和消费者的关系非常松散，具体而言，一个主题可能有零个，一个或多个向其发送消息的生产者；相反，生产者可以发送不同主题的信息。从消费者的角度来看，一个主题可能有零个，一个或多个消费者群体订阅。同样，一个消费群体可以订阅一个或多个主题，只要这个群体的实例保持其订阅的一致性即可。

### Message

​		消息是要传递的信息。一条信息必须要有一个主题，可以将其解释为要发送给您的信件的地址。一条消息也可能有一个可选标签和额外的键值对。例如，你可以为消息设置业务密钥，并在Broker上查找消息以在开发期间诊断问题。

### Message Queue

​		主题被划分为一个或多个子主题，这就是消息队列。

### Tag

​		标签可以理解为更细一级的主题，为使用者提供更灵活的查找。使用标签，来自同一业务模块的具有不同目的消息可能就有相同的主题和不同的标签。标签将有助于保持代码的清洁和一致性，并且标签还可以方便RocketMQ提供的查询系统。

### Message Order

​		当使用DefaultMQPushConsumer时，你可能需要决定消费是顺序的还是并发的。

- Orderly（顺序）：有序的消息意味着消息的使用顺序与生产者为每个消息队列发送的顺序相同。如果你的使用场景要求是必须顺序的，你要确保只用一个队列存放消息。如果消费顺序被指定，最大的消费并发数就是这个消费者组的消息队列的订阅数。
- Concurrently（并发）：并发使用消息时，消费消息的最大并发性仅受限于为每个消费者客户端指定的线程池。

## 安装RocketMQ

### 1、下载Apache最新rocketmq二进制压缩文件

​		可以到官网下载后上传到服务器上，也可以用`wget`命令。	

```shell
wget https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.5.2/rocketmq-all-4.5.2-bin-release.zip/
```

### 2、解压安装

​		使用`unzip`命令进行解压。

```shell
unzip -d /usr/local rocketmq-all-4.5.2-bin-release.zip
```

### 3、环境变量

​		配置环境变量 `vi /etc/profile`。添加如下代码：

```shell
export NAMESRV_ADDR=127.0.0.1:9876
```

### 4、启动RocketMQ

​		进入rocketmq的bin目录，修改`runserver.sh`，如下代码：

```shell
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

​		主要根据自己机器内存酌情修改-Xms -Xmx -Xmn这几个参数的内存。

​		同理修改bin目录下的`runbroker.sh`的JVM参数。

​		进入conf目录，修改`broker.conf`，新增一行：

```shell
brokerIP1=xx.xx.xx.xx  # 你的公网IP
```

​		然后开始启动mqnamesrv，进入到rocket目录，执行`nohup sh bin/mqnamesrv &`启动namesrv，然后再执行`nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.conf &`启动broker。

​		执行`jps`，看namesrv和broker是否启动成功，如果没成功。可以通过执行`tail -f ~/logs/rocketmqlogs/namesrv.log`和`tail -f ~/logs/rocketmqlogs/broker.log`查看相应日志。

## RocketMQ集群模式

​		RocketMQ集群部署有多种方式，对于NameSrv来说可以同时部署多个节点，并且这些节点间也不需要有任何的信息同步，因为这里每个NameSrv节点都会存储全量路由信息。在NameSrv集群模式下，每个Broker都需要同时向集群中的每个NameSrv节点发送注册信息，所以这里对于NameSrv的集群部署来说并不需要做什么额外的设置。

​		而对于Broker集群来说就有多种模式了，主要有如下几个模式：

### 单个Master模式

​		一个Broker作为主服务，不设置任何Slave，这种方式风险比较大，存在单节点故障会导致整个基于消息的服务挂掉，所以生产环境不可能采用这种模式。

### 多Master模式

​		这种模式的Broker集群，全是Master，没有Slave。

#### 	优点

​		配置会比较简单一些，如果单个Master挂掉或重启维护的话对应用是没有什么影响的。如果磁盘配置为RAID10（服务器的磁盘阵列模式）的话，即使在机器宕机不可恢复的情况下，由于RAID10磁盘本身的可靠性，消息也不会丢失（异步刷盘丢失少量消息，同步刷盘一条不丢），这个Broker的集群模式性能相对来说是最高的。

#### 	缺点

​		在单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前是不可以进行消息订阅的，这对消息的实时性有一些影响。

### 多Master多Slave模式（异步复制）

​		这种模式下的Broker集群存在多个Master节点，并且每个Master节点都会对应一个Slave节点，有多对Master-Slave，HA（高可用）之间采用异步复制的方式进行信息同步，在这种方式下主从之间会有短暂的毫秒级的消息延迟。

#### 	优点

​		这种模式下即使磁盘损坏了，消息丢失的情况也非常少，因为主从之间有消息备份。并且，这种模式下的实时性也不会受影响，因为Master宕机后Slave可以自动切换为Master模式，这样Consumer仍然可以通过Slave进行消息消费，而这个过程对应用来说是完全透明的，并不需要人工干预。另外，这种模式的性能与多Master模式几乎差不多。

#### 	缺点

​		如果Master宕机，并且在磁盘损坏的情况下，会丢失少量的消息。

### 多Master多Slave模式（同步复制）

​		这种模式与上面那个差不多，只是HA采用的是同步双写的方式，即主备都写成功后，才会向应用返回成功。

#### 	优点

​		这种模式下数据与服务不存在单点的情况，在Master宕机的情况下，消息也没有延迟，服务的可用性以及数据的可用性都非常高。

#### 	缺点

​		性能相比于异步复制略低一些（大约10%）。

## SpringBoot整合RocketMQ

​		这里主要讲解一下生产者和消费者的代码，完整的项目代码已上传到[github](https://github.com/GD-CKING/demo/tree/master/rocketmq_demo)上。

### 消息生产者RocketMQProvider

​		以顺序发消息为例：

```java
package com.example.rocketmqDemo.rocketmq;

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.util.StopWatch;

import java.util.List;

@Service
public class RocketMQProvider {

    @Value("${apache.rocketmq.producer.producerGroup}")
    private String producerGroup;

    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    public void defaultMQProducer() {

        //生产组的名称
        DefaultMQProducer producer = new DefaultMQProducer(producerGroup);

        //指定NameServer地址，多个地址以;隔开
        producer.setNamesrvAddr(namesrvAddr);

        try {
            //Producer对象在使用之前必须要调用start初始化，初始化一次即可
            //注意：切记不可以在每次发送消息时，都调用start方法
            producer.start();

            //创建一个消息实例，包含topic、tag和消息体
            //如下：topic为"TopicTest"，tag为"push"
            Message message = new Message("TopicTest", "push", "发送消息----".getBytes());

            StopWatch stop = new StopWatch();

            stop.start();

            for(int i = 0; i < 10; i++) {
                SendResult result = producer.send(message, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, 1);
                System.out.println("发送相应：MsgId: " + result.getMsgId() + ".发送状态: " + result.getSendStatus());
            }
            stop.stop();
            System.out.println("-----------------------发送十条消息耗时：" + stop.getTotalTimeMillis());
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
            producer.shutdown();
        }

    }
}
```

​		一般消息时通过轮询所有队列来发送的（负载均衡策略），顺序消息可以根据业务发送到同一个队列。比如将订单号相同的消息发送到同一个队列。下面的代码中指定了1,1处这个值相同的获取到的队列是同一个队列。

```java
SendResult result = producer.send(message, new MessageQueueSelector() {
  @Override
  public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
         Integer id = (Integer) arg;
         int index = id % mqs.size();
         return mqs.get(index);
  }
}, 1);
```

### 消息消费者RocketMQConsumer

```java
package com.example.rocketmqDemo.rocketmq;

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Service;

@Service
public class RocketMQConsumer implements CommandLineRunner {

    //消费者的组名
    @Value("${apache.rocketmq.consumer.PushConsumer}")
    private String consumerGroup;

    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    public void defaultMQPushConsumer() {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroup);

        consumer.setNamesrvAddr(namesrvAddr);

        try {
            //订阅PushTopic下Tag为push的消息
            consumer.subscribe("TopicTest", "push");

            //设置Consumer第一次启动是从头部开始消费还是队列尾部开始消费
            //如果非第一次启动，那么按照上次消费的位置继续消费
            consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
            consumer.registerMessageListener((MessageListenerConcurrently)(list, context)-> {
                try {
                    for(MessageExt messageExt : list) {
                        //输出消息内容
                        System.out.println("messageExt: " + messageExt);

                        String messageBody = new String(messageExt.getBody());

                        //输出消息内容
                        System.out.println("消费响应：msgId : " + messageExt.getMsgId() + ", msgBody : " + messageBody);
                    }
                }catch (Exception e) {
                    e.printStackTrace();
                    //稍后重试
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
                //消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            });
            consumer.start();
        }catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run(String... args) throws Exception {
        this.defaultMQPushConsumer();
    }
}
```

​		消费者中我们实现了`CommandLineRunner`接口。它的作用是让消费者在springboot启动时执行。具体可以参考[CommandLineRunnable详解](https://blog.csdn.net/qq_34531925/article/details/82527066)。

​		项目成功启动后，测试的结果应该是：

```verilog
2019-10-11 17:13:50.785  INFO 12872 --- [nio-8080-exec-2] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2019-10-11 17:13:50.785  INFO 12872 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2019-10-11 17:13:50.793  INFO 12872 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 8 ms
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=51, sysFlag=0, bornTimestamp=1570785233301, bornHost=/61.144.97.110:2191, storeTimestamp=1570785231900, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEAFFF, commitLogOffset=28225535, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=52, CONSUME_START_TIME=1570785233370, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=50, sysFlag=0, bornTimestamp=1570785233233, bornHost=/61.144.97.110:2191, storeTimestamp=1570785231856, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEAF4D, commitLogOffset=28225357, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=52, CONSUME_START_TIME=1570785233370, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=52, sysFlag=0, bornTimestamp=1570785233346, bornHost=/61.144.97.110:2191, storeTimestamp=1570785231943, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB0B1, commitLogOffset=28225713, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=53, CONSUME_START_TIME=1570785233411, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=53, sysFlag=0, bornTimestamp=1570785233387, bornHost=/61.144.97.110:2191, storeTimestamp=1570785231986, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB163, commitLogOffset=28225891, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=54, CONSUME_START_TIME=1570785233453, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=54, sysFlag=0, bornTimestamp=1570785233430, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232028, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB215, commitLogOffset=28226069, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=55, CONSUME_START_TIME=1570785233495, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=55, sysFlag=0, bornTimestamp=1570785233472, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232071, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB2C7, commitLogOffset=28226247, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=56, CONSUME_START_TIME=1570785233538, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=56, sysFlag=0, bornTimestamp=1570785233516, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232114, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB379, commitLogOffset=28226425, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=57, CONSUME_START_TIME=1570785233580, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=57, sysFlag=0, bornTimestamp=1570785233557, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232156, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB42B, commitLogOffset=28226603, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=58, CONSUME_START_TIME=1570785233622, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=58, sysFlag=0, bornTimestamp=1570785233600, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232199, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB4DD, commitLogOffset=28226781, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=59, CONSUME_START_TIME=1570785233665, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
发送相应：MsgId: 847F127D324818B4AAC2373225500000.发送状态: SEND_OK
-----------------------发送十条消息耗时：1479
messageExt: MessageExt [queueId=1, storeSize=178, queueOffset=59, sysFlag=0, bornTimestamp=1570785233642, bornHost=/61.144.97.110:2191, storeTimestamp=1570785232241, storeHost=/47.105.176.129:10911, msgId=2F69B08100002A9F0000000001AEB58F, commitLogOffset=28226959, bodyCRC=800335203, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=60, CONSUME_START_TIME=1570785233707, UNIQ_KEY=847F127D324818B4AAC2373225500000, WAIT=true, TAGS=push}, body=16]]
消费响应：msgId : 847F127D324818B4AAC2373225500000, msgBody : 发送消息----
```

## 参考资料

[Centos7下安装Rocket](https://www.jianshu.com/p/04a98ba770a4)

[RocketMQ连接异常](https://blog.csdn.net/q258523454/article/details/82716027?utm_source=blogxgwz4)

[基于RocketMQ搭建生产级消息集群](https://mp.weixin.qq.com/s?__biz=MzU3NDY4NzQwNQ==&mid=2247484461&idx=1&sn=72794d12ffe3c8ec3520fdba1c87c3b3&chksm=fd2fd5efca585cf955d240bbabd99b24262d255a3880454297f10a84e4c38445bc7b9d3ba339&scene=21)

