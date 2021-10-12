---
title: Spring 事务失效情况
date: 2021-09-26 19:16:19
tags: Spring
categories: Spring
---

## 没有被 Spring 管理



没有被 Spring 管理的 Bean，如果其中出现了方法需要进行事务处理的情况，此时的事务不会执行。如下，在 OrderServiceImpl 类中如果将 @Service 注释掉，那么此类对应的 bean 也就不会被 Spring IOC 容器管理。即使 updateOrder 上面注释了 `@Transactional`，这个方法也不会执行事务



```java
//@Service
public class OrderServiceImpl {
    
    @Transactional
    public void updateOrder(Order order) {
        // update order
    }
}
```



## 方法不是 public



@Transactional 只能用于 public 方法上，否则事务会失效。如果要用在非 public 方法上，可以开启 AspectJ 代理模式。如下，saveAll 方法上面标注了 @Transactional 的注释，由于 saveAll 方法没有标注 public，因此 saveAll 的内容是不会执行事务的



```java
@Service
public class DemoServiceImpl implements DemoService{
    
    @Transactional(rollbackFor = Exception.class)
    @Override
    int saveAll() {
        // do something
        return 1;
    }
}
```



## 自身调用问题



当一个服务存在多个方法，都标注了事务 @Transactional，当其中一个方法调用另外一个方法的时候，被调用方法中的事务是不会执行的。



如下，在 ServiceA 中存在 doSomething 有 @Transactional 的标注，在方法体内会调用 insert 方法，insert 方法上面也有 @Transactional 的标注。此时 doSomething 中的事务可以执行，但是 insert 方法的事务就无法执行



```java
@Service
public class ServiceA {

    @Transactional
    public void doSomething() {

        insert();

        // 调用其他系统
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void insert() {
        // 向数据库中添加数据
    }
}
```



为了解决这个问题，可以将 insert 方法放到另外一个类中进行调用，如下，将 doSomething 和 insert 方法分别放到 ServiceA 和 ServiceB 两个类中，然后再通过 doSomething 调用 insert。此时两个方法中的事务都会执行



```java
@Service
public class ServiceA {
    
    @Autowired
    private ServiceB serviceB;

    @Transactional
    public void doSomething() {

        serviceB.insert();

        // 调用其他系统
    }
}

@Service
public class ServiceB {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void insert() {
        // 向数据库中添加数据
    }
}
```



## 数据源没有配置事务管理器



如下，数据源没有配置事务管理器，那么下面的方法 TransactionManager 是无法实现事务的



```java
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```



## 事务传播级别选择



如果在嵌套调用时存在事务传播的场景，在定义事务级别的时候是用来 `NOT_SUPPORTED`，那么对应的方法就不会执行事务。如下，在 OrderServiceImpl 中执行的 update 方法执行了事务，同时该方法调用 updateOrder 方法。由于 updateOrder 方法的 @Transactional 注释中标注 了 `Propagation.NOT_SUPPORTED`，因此 updateOrder 方法的事务是不执行的



```java
@Service
public class OrderServiceImpl {

    @Transactional
    public void update(Order order) {
        updateOrder();
    }
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void updateOrder(Order order) {
        // update order
    }
}
```



## 事务中出现异常



如果在执行方法中执行的事务，由于异常退出的情况，那么这个事务是无法完成的。如下，在 updateOrder 方法使用了 try catch，一旦方法体中出现异常，此时事务会中断无法执行



```java
@Service
public class OrderServiceImpl implements OrderService {

    @Transactional
    public void updateOrder(Order order) {]

        try {
            // update order

        } catch (Exception e) {
            // do something
        }
    }
}
```



