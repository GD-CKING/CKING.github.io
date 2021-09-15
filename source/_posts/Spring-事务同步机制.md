---
title: Spring 事务同步机制
date: 2021-04-08 11:00:09
tags: Spring
categories: Spring
---

## Spring 事务极简介绍

Spring 有「声明式事务」和「编程式事务」：



- **声明式事务**只需要提供 `Transactional` 的注解，然后事务的开启和提交 / 回滚、资源的清理就都由 Spring 来管控，我们只需要关注业务代码即可

- **编程式事务**则需要使用 Spring 提供的模板，如 `TransactionTemplate`，或者直接使用底层的 `PlatformTransactionManager` 手动控制提交、回滚



声明式事务最大的优点就是对代码的**侵入性小**，只需要在方法上加 `@Transcational`的注解就可以实现事务；编程式事务的最大优点是对事务的**管控粒度较细**，可以实现代码块级别的事务



## 前提介绍

Spring 把 JDBC 的 `Connection` 或者 `Hibernate` 的 `Session` 等数据库的链接（会话）都统一称为资源。我们知道 `Connection` 是线程不安全的，同一时刻是不能被多个线程共享的



> 同一时刻，我们每个线程持有的 Connection 应该是独立的，且都是互不干扰和互不相同的



**但是 Spring 管理的 Service、Dao 等都是无状态的单例 Bean**，如何保证单例 Bean 里面使用的 `Connection` 都能够独立？Spring 引入一个类：事务同步管理器 `TransactionSynchronizationManager` 来解决这个问题。它的做法是内部使用了很多的 `ThreadLocal` 为不同的事务线程提供了独立的资源副本，**并同时维护这些事务的配置属性和运行状态信息（比如强大的事务嵌套、传播属性等）**



同步管理器 `TransactionSynchronizationManager` 是掌管这一切的大脑，它管理的 `TransactionSynchronization` 是开放给调用者一个非常重要的扩展点



`TransactionSynchronizationManager` 将 `Dao`、`Service` 中影响线程安全的所有 “状态” 都统一抽取到该类中，并用 `ThreadLocal` 进行封装，这样一来，Dao（基于模板类或资源获取工具类创建的 Dao）和 Service（采用 Spring 事务管理机制）就不用自己来保存一些事务状态了，从而就变成了线程安全的单例对象了



### DataSourceUtils

在这里我们先介绍一下 `DataSourceUtils` 这个工具类。在一些场景里，比如使用 Mybatis 的时候，可能无法使用 Spring 提供的模板来达到效果，而是需要直接操作原生 API `Connection` 



那如何拿到这个链接 `Connection` 呢（这里有个前提：必须保证和当前 Mybatis 线程使用的是同一个链接，这样才接收本事务的控制）。这个事务 `DataSourceUtils` 这个工具类就闪亮登场了，它提供了这个能力：



```java
public abstract class DataSourceUtils {
	...
	public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException { ... }
	...
	
	// 把definition和connection进行一些准备工作~
	public static Integer prepareConnectionForTransaction(Connection con, @Nullable TransactionDefinition definition) throws SQLException { ...}

	// Reset the given Connection after a transaction,
	// con.setTransactionIsolation(previousIsolationLevel);和con.setReadOnly(false);等等
	public static void resetConnectionAfterTransaction(Connection con, @Nullable Integer previousIsolationLevel) { ... }

	// 该JDBC Connection 是否是当前事务内的链接~
	public static boolean isConnectionTransactional(Connection con, @Nullable DataSource dataSource) { ... }

	// Statement 给他设置超时时间  不传timeout表示不超时
	public static void applyTransactionTimeout(Statement stmt, @Nullable DataSource dataSource) throws SQLException { ... }
	public static void applyTimeout(Statement stmt, @Nullable DataSource dataSource, int timeout) throws SQLException { ... }

	// 此处可能是归还给连接池，也有可能是close~（和连接池参数有关）
	public static void releaseConnection(@Nullable Connection con, @Nullable DataSource dataSource) { ... }
	public static void doReleaseConnection(@Nullable Connection con, @Nullable DataSource dataSource) throws SQLException { ... }

	// 这个是真close
	public static void doCloseConnection(Connection con, @Nullable DataSource dataSource) throws SQLException { ... }
	
	// 如果链接是代理，会拿到最底层的connection
	public static Connection getTargetConnection(Connection con) { ... }
}
```



`getConnection()` 这个方法就是从 `TransactionSynchronizationManager` 里拿到一个现成的 `Connection`，若没有现成的，会用 `DataSource` 创建一个链接然后放进去



其实 Spring 不就为 JDBC 提供了这个工具类，还有 Hibernate、JPA、JDO 等提供了类似的工具类



- org.springframework.orm.hibernate.SessionFactoryUtils.getSession()
- org.springframework.orm.jpa.EntityManagerFactoryUtils.getTransactionalEntityManager()
- org.springframework.orm.jdo.PersistenceManagerFactoryUtils.getPersistenceManager()



## 问题场景模拟

为了更好地解释和说明，此处模拟出这样的一个场景：



```java
@Slf4j
@Service
public class HelloServiceImpl implements HelloService {
    
    @Autowired
    private JdbcTemplate jdbcTemplates;
    
    @Transactional
    @Override
    public Object hello(Integer id) {
        // 向数据库插入一条记录
        String sql = "insert into user(id, name, age) values(" + id + ", 'fsx', 21)";
        jdbcTemplate.update(sql);
        
        // 做其余的事情，可能抛出异常
        System.out.println(1 / 0);
        return "service hello";
    }
}
```



如上 Demo，因为 `1 / 0` 会报错，因为有事务，所以最终这个插入是不会成功的



```java
@Transactional
@Override
public Ojbect hello(Integer id) {
    // 向数据库插入一条记录
    String sql = "insert into user(id, name, age) values(" + id + ", 'fsx', 21)";
    jdbcTemplage.update(sql);
    
    // 根据id去查询获取总数（若查询到了肯定是 count = 1）
    String query = "select count(1) from user where id = " + id;
    Integer count = jdbcTemplate.queryForObject(query, Integer.class);
    log.info(count.toString());
    
    return "service hello";
}
```



稍微改造一下，按照上面那么写，count 永远是返回 1 的。前面 insert 一条记录，下面是可以立马去查询出来的



下面把它改造如下：



```java
@Transactional
@Override
public Ojbect hello(Integer id) {
    // 向数据库插入一条记录
    String sql = "insert into user(id, name, age) values(" + id + ", 'fsx', 21)";
    jdbcTemplage.update(sql);
    
    // 生产环境，一般会把这些操作交给线程池，这里只是模拟一下效果
    new Thread(() -> {
   		 String query = "select count(1) from user where id = " + id;
    	Integer count = jdbcTemplate.queryForObject(query, Integer.class);
    	log.info(count.toString());     
    }).start();
   
    // 把问题放大
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch(Exception e) {
        e.printStackTrace();
    }
    
    return "service hello";
}
```



经过上面的改造，这样我们的 count 的值就永远是 0 了



这个现象是一个非常严重的问题。他可能会出现：刚插入的数据竟然查不到的诡异现象。



## 解决方案

在互联网中，我们经常为了提高吞吐量、程序性能，会使用到异步的方式进行优化、削峰等。因此连接池、线程池得到了大量的应用。比如一个业务处理中，发短信、发微信通知、记录操作日志等这些非主干需求，我们一般都希望都交给线程池去处理而不要干扰主要业务流程，所以上面模拟的现象出现问题的可能性还是蛮高的



上面出现的问题中，我们先给解决方案。我们的诉求是：**我们的异步线程执行时，必须确保记录已经持久化到数据库了**。因此可以这么处理：



```java
@Transactional
@Override
public Ojbect hello(Integer id) {
    // 向数据库插入一条记录
    String sql = "insert into user(id, name, age) values(" + id + ", 'fsx', 21)";
    jdbcTemplage.update(sql);
    
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        // 在事务提交之后执行的代码块（方法），此处使用 TransactionSynchronizationAdapter
        // 其实在 Spring5 后直接使用接口也很方便
        
        @Override
        public void afterCommit() {
            new Thread(() -> {
   		 	String query = "select count(1) from user where id = " + id;
    		Integer count = jdbcTemplate.queryForObject(query, Integer.class);
    		log.info(count.toString());     
    		}).start();
        }
    });
   
    // 把问题放大
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch(Exception e) {
        e.printStackTrace();
    }
    
    return "service hello";
}
```



我们使用 `TransactionSynchronizationManager` 注册一个 `TransactionSynchronization`，然后在 `afterCommit` 里执行我们的后续代码，这样就能 100% 确保我们的后续逻辑是在当前事务被 `commit` 后才执行的



### TransactionSynchronizationManager

TransactionSynchronizationManager 使用 `ThreadLocal` 记录事务的一些属性，用于应用扩展同步器的使用，在事务的开启、挂起、提交等各个点上回调应用的逻辑



```java
public abstract class TransactionSynchronizationManager {

	// ======保存着一大堆的ThreadLocal 这里就是它的核心存储======

	//  应用代码随事务的声明周期绑定的对象  比如：DataSourceTransactionManager有这么做：
	//TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
	// TransactionSynchronizationManager.bindResource(obtainDataSource(), suspendedResources);
	// 简单理解为当前线程的数据存储中心~~~~
	private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");

	// 使用的同步器，用于应用扩展
	// TransactionSynchronization同步器是最为重要的一个扩展点~~~ 这里是个set 所以每个线程都可以注册N多个同步器
	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = new NamedThreadLocal<>("Transaction synchronizations");
	
	// 事务的名称  
	private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal<>("Current transaction name");
	// 事务是否是只读  
	private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal<>("Current transaction read-only status");
	// 事务的隔离级别
	private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal<>("Current transaction isolation level");
	// 事务是否开启   actual：真实的
	private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal<>("Actual transaction active");

	// 返回的是个只读视图
	public static Map<Object, Object> getResourceMap() {
		Map<Object, Object> map = resources.get();
		return (map != null ? Collections.unmodifiableMap(map) : Collections.emptyMap());
	}

	public static boolean hasResource(Object key) { ... }
	public static Object getResource(Object key) { ... }
	
	// actualKey：确定的key  拆包后的
	@Nullable
	private static Object doGetResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
		// Transparently remove ResourceHolder that was marked as void...
		// 如果ResourceHolder 被标记为了void空白了。此处直接从map里移除掉对应的key 
		// ~~~~~~~并且返回null~~~~~~~~~~~
		if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
			map.remove(actualKey);
			// Remove entire ThreadLocal if empty...
			if (map.isEmpty()) {
				resources.remove();
			}
			value = null;
		}
		return value;
	}

	// 逻辑很简单，就是和当前线程绑定一个Map，并且处理ResourceHolder 如果isVoid就抛错
	public static void bindResource(Object key, Object value) throws IllegalStateException {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Assert.notNull(value, "Value must not be null");
		Map<Object, Object> map = resources.get();
		// set ThreadLocal Map if none found
		if (map == null) {
			map = new HashMap<>();
			resources.set(map);
		}
		Object oldValue = map.put(actualKey, value);
		// Transparently suppress a ResourceHolder that was marked as void...
		if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
			oldValue = null;
		}
		if (oldValue != null) {
			throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
					actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
		}
	}

	public static Object unbindResource(Object key) throws IllegalStateException { ... }
	public static Object unbindResourceIfPossible(Object key) { ... }
	

	// 同步器是否是激活状态~~~  若是激活状态就可以执行同步器里的相关回调方法了
	public static boolean isSynchronizationActive() {
		return (synchronizations.get() != null);
	}

	// 如果事务已经开启了，就不能再初始化同步器了  而是直接注册
	public static void initSynchronization() throws IllegalStateException {
		if (isSynchronizationActive()) {
			throw new IllegalStateException("Cannot activate transaction synchronization - already active");
		}
		logger.trace("Initializing transaction synchronization");
		synchronizations.set(new LinkedHashSet<>());
	}

	// 注册同步器TransactionSynchronization   这个非常重要 下面有详细介绍这个接口
	// 注册的时候要求当前线程的事务已经是激活状态的  而不是随便就可以调用的哦~~~
	public static void registerSynchronization(TransactionSynchronization synchronization) throws IllegalStateException {
		Assert.notNull(synchronization, "TransactionSynchronization must not be null");
		if (!isSynchronizationActive()) {
			throw new IllegalStateException("Transaction synchronization is not active");
		}
		synchronizations.get().add(synchronization);
	}


	// 返回的是只读视图  并且，并且支持AnnotationAwareOrderComparator.sort(sortedSynchs); 这样排序~~
	public static List<TransactionSynchronization> getSynchronizations() throws IllegalStateException { ... }
	public static void clearSynchronization() throws IllegalStateException { ... }

	... // 省略name等其余几个属性的get/set方法  因为没有任何逻辑
	// 这个方法列出来，应该下面会解释
	public static void setActualTransactionActive(boolean active) {
		actualTransactionActive.set(active ? Boolean.TRUE : null);
	}
	
	// 清楚所有和当前线程相关的（注意：此处只是clear清除，和当前线程的绑定而已~~~）
	public static void clear() {
		synchronizations.remove();
		currentTransactionName.remove();
		currentTransactionReadOnly.remove();
		currentTransactionIsolationLevel.remove();
		actualTransactionActive.remove();
	}
}
```



这里把 `setActualTransactionActive` 单独拿出来看一下。在 `AbstractPlatformTransactionManager.getTransaction()` 的时候会调用此方法：



```java
// 相当于表示事务为开启了
TransactionSynchronizationManager.setActualTransactionActive(status.hasTransacation());
```



而且该类的 `handleExistingTransaction`、`prepareTransactionStatus` 等方法都会调用上面的方法，也就是说它会参与到事务的声明周期里面去



> 备注：以上方法他们统一的判断条件有：TransactionStatus.isNewTransaction() 是新事务的时候才会调用这个方法进行标记



另外，AbstractPlatformTransactionManager 的 `suspend` 暂停的时候会直接这么调用：



```java
TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
TransactionSynchronizationManager.setActualTransactionActive(false);
```



`resume` 恢复的时候：



```java
TransactionSynchronizationManager.setCurrentTransactionReadOnly(resourcesHolder.readOnly);
TransactionSynchronizationManager.setActualTransactionActive(resourcesHolder.wasActive);
```



大体可以得出这样的一个处理步骤：



- 开启新的事务时初始化。第一次开启事务分为：real 首次或已存在事务但是 REQUIRES_NEW

- 在事务的嵌套过程中，TransactionSynchronizationManager 属性不断更新最终清除。即外层事务挂起、事务提交，这两个点需要更新 TransactionSynchronizationManager 属性

- 这里面有个内部类 ：AbstractPlatformTransactionManager.SuspendedResourcesHolder。它是负责事务挂起的时候，保存事务属性的对象，用于恢复外层事务。当恢复外层事务时，根据 `SuspendedResourcesHolder` 对象，调用底层事务框架恢复事务属性，并恢复 TransactionSynchronizationManager



### DefaultTransactionStatus

它实现了 `TransactionStatus` 接口，**这个是整个事务框架最重要的状态对象，它贯穿于事务拦截器，Spring 抽象框架和底层具体事务实现框架之间**



它的重要任务就是在新建、挂起、提交事务的过程中保存对应事务的属性，在 `AbstractPlatformTransactionManager` 中，每个事务流程都会 new 创建这个对象



### TransactionSynchronizationUtils

这个工具类比较简单，主要是处理 `TransactionSynchronizationManager` 和执行 `TransactionSynchronization` 对应的方法们



### TransactionSynchronization

这个类非常重要，它是我们对事务同步的扩展点，用于事务同步回调的接口，AbstractPlatformTransactionManager 支持它



注意：自定义的同步器可以通过实现 `Ordered` 接口来自己定制化顺序，若没实现接口就按照添加的顺序执行



