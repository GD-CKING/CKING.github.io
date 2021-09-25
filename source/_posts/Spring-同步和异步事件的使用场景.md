---
title: Spring 同步和异步事件的使用场景
date: 2021-09-25 14:17:39
tags: Spring
categories: Spring
---

Spring 事件的监听机制可以理解为是一种观察者模式，有数据发布者（事件源）和数据接受者（监听器）。在 Java 中，事件对象都是继承 java.util.EventObject 对象，事件监听器都是 java.util.EventListener 实例



其中，java.util.EventObject 是事件状态对象的基类，它封装了事件源对象以及和事件相关的信息。所有 java 的事件类都需要继承该类



而 java.util.EventListener 是一个接口，所有事件监听器都需要实现该接口。事件监听器注册在事件源上，当事件源的属性或状态改变的时候，调用相应监听器内的回调方法



Source 作为事件源不需要实现或继承任何接口或类，它是事件最初发生的地方。因为事件源需要注册事件监听器，所以事件源内需要存放事件监听器的容器



例如，我们需要将 `User` 信息保存到数据库中，然后发送一条 `User` 的消息，然后由对应的监听器来处理



如下代码，`UserEvent` 作为需要发送的消息继承于 `ApplicationEvent`，同时提供了 `getData` 和 `setData` 的方法来存放需要处理的数据



```java
public class UserEvent <T> extends ApplicationEvent {

    private T data;
    public UserEvent(T source) {
        super(source);
        this.data = source;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```



接着就是发送 UserEvent 消息的 `sendInsertUser` 方法，如下，该方法创建 User 的实例以后填写对应的属性，然后通过 `ApplicationEventPublisher` 类型的 publisher 变量执行 `publishEvent` 方法进行发送消息



```java
@Autowired
private ApplicationEventPublisher publisher;

private void sendInsertUser() {
    User user = new User();
    user.setId(1);
    user.setUsername("李四");
    UserEvent<User> userEvent = new UserEvent<>(user);
    // 发布事件
    publisher.publishEvent(userEvent);
}
```



有了发送消息的代码就有监听消息的代码。insertUserListener 方法在 `@EventListener` 的修饰下就成为消息监听器。它会接收 UserEvent 消息，并且对消息进行处理，最后把处理的结果 message 记录到日志中



```java
@EventListener
public void insertUserListener(UserEvent<User> userEvent) {
    User data = userEvent.getData();
    String message = String.format("get user message: %s", JSON.toJSONString(data));
    log.info(message);
}
```



接着测试代码，如下，在测试方法中先执行 `insertUsers` 方法用来往 User 表中插入数据，然后再执行 `sendInsertUser` 方法来发送消息



```java
public BackResult testEvent() {

	BackResult backResult = new BackResult();
	try{
		// 向 user 表插入数据
		insertUsers();
		// 添加 user 插入对象操作
		sendInsertUser();
		
		backResult.setSuccess(true);
		backResult.setMessage("操作成功");
	} catch(Exception e) {
		backResult.setMessage(e.getMessage());
		backResult.setSuccess(false);
	}
	return backResult;
}
```



## Spring 消息的同步处理



假设上面的例子中两个方法 `insertUsers` 和 `sendInsertUser` 需要同步执行，也就是 `insertUsers` 方法会先执行，完成数据库的提交以后再 执行 `sendInsertUser` 方法发送消息。如果 `insertUsers` 数据库提交一直没有返回成功的结果，`sendInsertUser` 就需要一直等待



也就是说 `insertUsers` 会阻塞 `sendInsertUser` 方法，这就是我们常说的同步执行。**为了做到同步执行，就需要将监听器加入到主线程的事务中，让这两个方法顺序执行**



如下，首先在 `insertUsers` 方法上面打了 `@Transactional` 的注释，标注它是一个事务操作



```java
@Transactional(rollbackFor = Exception.class)
public void insertUsers() {
	// insert data into database
}
```



然后修改 `insertUserListener` 方法，在上面加 `@TransactionalEventListener`的注释，并且通过 `phase` 属性标注 `TransactionPhase.AFTER_COMMIT`，意思是在事务提交以后，再执行方法内容。



也就是在 insert user 数据库事务提交以后，再处理监听到的 UserEvent 消息，从而保证事务的一致性



```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = true)
public void insertUserListener(UserEvent<User> userEvent) {
    User data = userEvent.getData();
    String message = String.format("get user message: %s", JSON.toJSONString(data));
    log.info(message);
}
```



从这个例子可以发现，如果执行的操作和发送消息以后对应的操作业务上有较高的关联性，可以作为一个事务的两个操作来处理，要么一起完成要么都不完成的话，那么就需要进行同步处理。



常见的业务场景有，下单操作和通知减扣库存，下单以后商品别售出，库存随之扣减，如果下单失败扣减库存的操作也不能执行



## Spring 消息的异步处理



上面说的是同步处理的业务场景，还是这个例子，假设 `insertUsers` 和 `sendInsertUser` 这两个操作的业务关联性并不大，那么如何处理？例如，insertUsers 之后只是发一个通知给用户，说数据入库了。就算是入库不成功，消息发送也不用撤回



这种场景业务就可以用异步处理来做，也就是说 `snedInserUser` 不用等待 `insertUsers` 方法数据库条件完成就可以处理发送的消息了。即这两个方法不是串行执行而是并行执行，两个操作的执行时互相不干扰的



如下，异步处理只需要在原方法上加上 `@Async` 的注解，同时需要注意的是，需要使用 `EnableAsync` 开启 Spring 异步模式



```
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = true)
public void insertUserListener(UserEvent<User> userEvent) {
    User data = userEvent.getData();
    String message = String.format("get user message: %s", JSON.toJSONString(data));
    log.info(message);
}
```

