---
title: SpringCloud生产环境配置服务的配置超时和重试参数
date: 2019-08-26 20:29:24
tags:
  - [分布式]
  - [SpringCloud]
categories:
  - [分布式]
  - [SpringCloud]
---

## 启动Ribbon的饥饿加载

​		在生产环境中，系统第一次启动时，调用其他服务经常会出现`timeout`，经过查阅资料得知：**每个服务第一次被请求的时候，他会去初始化一个Ribbon的组件**，初始化这些组件需要耗费一定的时间，所以很容易会导致timeout问题。解决方案是**让每个服务启动的时候直接初始化Ribbon相关的组件**，避免第一次请求的时候初始化。

```yml
ribbon:
  eager-load:
    enabled: true
```

​		上面只是解决了内部服务之间的调用，但还有一个问题就是：网关到内部服务的访问。由于Spring Cloud Zuul的路由转发也是通过Ribbon实现负载均衡的，所以也会存在第一次调用时比较慢的情况。

​		此时可以通过以下配置

```yaml
zuul:
  ignored-services: *
  ribbon:
    eager-load:
      enabled: true
```

​		Spring Cloud Zuul的饥饿加载中没有设计专门的参数来配置，而是直接采用了读取路由配置来进行饥饿加载的做法。所以，如果我们使用默认路由，而没有通过配置的方式制定具体路由规则，那么`zuul.ribbon.eager-load.enabled=true`的配置就没有作用了。

​		因此，在真正使用的时候，可以通过`zuul.ignored-services=*`来忽略所有的默认路由，让所有的路由配置均维护在配置文件中，以达到网关启动时就默认初始化好各个路由转发的负载均衡对象。

## Ribbon配置超时和重试参数

​		以下配置是用来配置Ribbon的超时时间和重试次数的：

```yaml
ribbon:
  ConnectTimeout: 250 # 连接超时时间(ms)
  ReadTimeout: 2000 # 通信超时时间(ms)
  OkToRetryOnAllOperations: true # 是否对所有操作重试
  MaxAutoRetriesNextServer: 1 # 同一服务不同实例的重试次数
  MaxAutoRetries: 1 # 同一实例的重试次数
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMillisecond: 10000 # 熔断超时时长：10000ms
```

​		假设在网关Zuul配置了以上参数，`MaxAutoRetriesNextServer`和`MaxAutoRetries`的意思是如果Zuul认为某个服务超时了，此时**会先重试一下该服务对应的这台机器，如果还是不行就会重试一下该服务的其他机器。**

