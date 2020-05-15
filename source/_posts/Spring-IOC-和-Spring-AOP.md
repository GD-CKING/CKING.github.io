---
title: Spring IOC 和 Spring AOP
date: 2020-01-09 08:44:12
tags: Spring
categories: Spring
---

## Spring IOC

在没有 Spring 之前，我们开发 Web 系统基本是使用 Servlet + Tomcat。就是 Tomcat 启动之后，它可以监听一个端口号的 http 请求，然后可以把请求转交给你的 servlet，jsp 配合起来使用，由 servlet 处理请求。大概像下面的代码：

```java
public class MyServlet {
	private MyService myService = new MyServiceImpl();
	
	public void doPost(HttpServletRequest request) {
		// 对请求一通处理
		// 调用自己的业务逻辑组件
		myService.doService(request);
	}
}

public interface MyService {}

public class MyServiceImpl implements MyService {}

public class NewServiceManagerImpl implements MyService {}
```

例如我们的一个 Tomcat + servlet 的这样的一个系统里，有几十个地方，都是直接用 `MyService myService = new MyServiceImpl()` 直接创建，引用和依赖了一个 MyServiceImpl 这样的一个类的对象。那么在这个系统里，就有几十个地方都跟 MyServiceImpl 类直接耦合在一起了。



如果现在不想用 MyServiceImpl 了，希望使用 NewServiceManagerImpl，同样也是 implements MyService 这个借口的。那么所有的实现逻辑都不同了，我们需要在这个系统的几十个地方，都去修改对应的 MyServiceImpl 这个类，切换为 NewServiceManagerImpl 这个类。这样就很麻烦。而且代码的改动成本很大，改动以后的测试成本很大，改动的过程可能会有点复杂，出现一些 bug。归根到底，代码里各种类之间完全耦合在一起，出现任何一丁点的变动，都需要改动大量的代码，重新测试，可能还有 bug。



这个时候，Spring IOC 框架就出场了。IOC（Inversion Of Control），中文翻译为“控制反转”，我们也叫依赖注入。之前是通过 XML 文件进行一个配置，现在可以基于注解来进行自动依赖注入。

```java
@Controller
public class MyController {
	
	@Resource
	private MyService myService;
	
	public void doRequest(HttpServletRequest request) {
		// 对请求一通处理
		// 调用自己的业务逻辑组件，去执行一些业务逻辑
		myService.doService(request);
	}
}

public class MyServiceImpl implements MyService {}

@Service
public class NewServiceManagerImpl implements MyService {}
```

我们只要在这个工程里通过 maven 引入一些 spring 框架的依赖，就可以实现 IOC 的功能。



Tomcat 在启动的时候，会直接启动 spring 容器。而 spring 容器，会根据 XML 的配置，或者是你的注解，去实例化一些 bean 对象，然后根据 XML 配置或者注解，去对 bean 对象之间的引用关系，进行依赖注入，某个 bean 依赖了另一个 bean。而这底层的核心技术，就是**反射**。它会通过反射的技术，直接根据你的类去构建对应的对象出来。



Spring IOC，最大的好处就是让系统的类与类之间彻底的解耦合。

![spring ioc](Spring-IOC-和-Spring-AOP/springIoc.png)

## Spring AOP

Spring AOP，又叫面向切面编程，可以应用于事务和日志等场景。拿事务举个例子，在数据库里，例如 MySQL，都提供一个事务机制，如果我们开启一个事务，在这个事务里执行多条增删改的 SQL 语句。在这个过程中，如果任何一个 SQL 语句失败了，会导致这个事务的回滚，把其他 SQL 做的数据更改都恢复回去。



在一个事务里的所有 SQL，要么一起成功，要么一起失败，事务功能可以保证我们的数据的一致性。我们可以在业务逻辑组件里加入这个事务。

```java
@Controller
public class MyController {
	
	@Resource
	private MyService myServiceA;
	
	public void doRequest() {
		myServiceA.doServiceA();
	}
}

@Service
public class MyServiceAImpl implements MyServiceA {

	public void doServiceA() {
		
		// 开启事务
		
		//insert语句
		//update语句
		//delete语句
		
		//根据是否抛出异常，回滚事务 or 提交事务
	}
}

@Service
public class MyServiceBImpl implements MyServiceB {

	public void doServiceB() {
		// 开启事务
		
		// update语句
		// update语句
		// insert语句
		
		//根据是否抛出异常，回滚事务 or 提交事务
	}
}
```

由上面的代码可以看出，所有的业务逻辑都有几段跟事务相关的代码。假设我们有几十个 Service 组件，类似一样的代码，重复的代码，必须在几十个地方都去写一样的东西，这就很难受了。这时候就轮到 Spring AOP 机制出马了。



AOP 作为一种编程范式，已经衍生出了属于它的一些先关术语。为了更好地理解如何在 Spring 中使用 AOP，我们必须对这些术语有一定的认知。



### 通知（Advice）

Spring AOP 支持五种类型的通知，它们分别定义了切面在什么时候使用，以及定义了切面需要做些什么。

@Before 前置通知，目标方法被调用之前执行

@After 后置通知，目标方法完成之后执行

@AfterReturning 返回通知，目标方法执行成功（未抛出异常）之后执行

@AfterThrowing 异常通知，目标方法执行失败（抛出异常）之后执行

@Around 环绕通知，目标方法执行前后都会调用

### 连接点（JoinPoint）

程序执行的某个特定位置：如类开始初始化前、类初始化后、类满足某个方法调用前、调用后、方法抛出异常后。一个类或一段程序代码拥有一些具有边界性质的特定点，这些点中的特定点就称为“连接点”。Spring 仅支持方法的连接点，即仅能在方法调用前、方法调用后、方法抛出异常时以及方法调用前后这些程序执行点织入增强。连接点由两个信息确定：第一是用方法表示的程序执行点，第二是用相对点表示的方位。

### 切点（Poincut）

每个程序类都拥有多个连接点，如一个拥有两个方法的类，这两个方法都是连接点，即连接点是程序中客观存在的事物。AOP 通过“切点”定位特定的连接点。连接点相当于数据库中的记录，而切点相当于查询条件。切点和连接点不是一对一的关系，一个切点可以匹配多个连接点。在 Spring 中，切点通过 `org.springframework.aop.Pointcut` 接口进行描述，它使用类和方法作为连接点的查询条件，Spring AOP 的规则解析引擎负责切点所设定的查询条件，找打对应的连接点。其实确切地说，不能称之为查询连接点，因为连接点是方法执行前、执行后等包括方位信息的具体程序执行点，而切点只定位到某个方法上，所以如果希望定位到具体连接点上，还需要提供方位信息。

### 切面（Aspect）

切面由切点和增强（引介）组成，它既包括了横切逻辑的定义，也包括了连接点的定义，Spring AOP 就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。 

### 引入（Introduction）

引入使我们具备了为类添加一些属性和方法的能力。这样，即使一个业务类原本没有实现某个接口，通过 AOP 的引介功能，我们可以动态地为该业务类添加接口的实现逻辑，让业务类成为这个接口的实现类。 

### 织入（weaving）

织入是将增强添加对目标类具体连接点上的过程。AOP 像一台织布机，将目标类、增强或引介通过 AOP 这台织布机天衣无缝地编织到一起。根据不同的实现技术，AOP 有三种织入的方式：1. 编译期织入，这要求使用特殊的Java编译器。2. 类装载期织入，这要求使用特殊的类装载器。3. 动态代理织入，在运行期为目标类添加增强生成子类的方式。Spring 采用动态代理织入，而 AspectJ 采用编译期织入和类装载期织入。

### 代理（Proxy）

 一个类被 AOP 织入增强后，就产出了一个结果类，它是融合了原类和增强逻辑的代理类。根据不同的代理方式，代理类既可能是和原类具有相同接口的类，也可能就是原类的子类，所以我们可以采用调用原类相同的方式调用代理类。 



一个切面，如何定义呢？例如 MyServiceImplXXXX 的这种类，在这些类的所有方法中，都去织入一些代码，在所有这些方法刚开始运行的时候，都先去开启一个事务，在所有这些方法执行完毕之后，去根据是否抛出异常来判断一下，如果抛出异常，就回滚事务，如果没有异常，就提交事务。



Spring 在运行的时候，会使用动态代理技术，这也是 AOP 的核心技术，来给你的那些类生成动态代理。

```java
public class ProxyMyServiceA implements MyServiceA {
	private MyServiceA myServiceA;
	
	public void doServiceA() {
		// 开启事务
		// 直接去调用我依赖的MyServiceA对象的方法
		myServiceA.doServiceA();
		
		// 根据是否抛出异常，回滚事务 or 提交事务
	}
}
```

那么上面的代码就可以变成这样：

```java
@Controller
public class MyController {
	
	@Resource
	private MyService myServiceA;	// 注入的是动态代理的对象实例，ProxyMyServiceA
	
	public void doRequest() {
		myServiceA.doServiceA();	// 直接调用到动态代理的对象实例的方法中去
	}
}

@Service
public class MyServiceAImpl implements MyServiceA {

	public void doServiceA() {
	
		//insert语句
		//update语句
		//delete语句
	}
}
```

下面给个简单的AOP代码的例子：

```java
@Aspect
@Component
public class ApiIdempotentAop {

    @Autowired
    private JedisUtil jedisUtil;

    @Pointcut("@annotation(com.example.midx.annotation.ApiIdempotent)")
    public void apiIdempotentPointCut() {

    }

    @Around("apiIdempotentPointCut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {

        System.err.println("进来AOP了");
        Object[] args = joinPoint.getArgs();

        //获取request和response
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        HttpServletResponse response = attributes.getResponse();
        String requestURI = request.getRequestURI();

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        ApiIdempotent annotation = signature.getMethod().getAnnotation(ApiIdempotent.class);

        Object result;
        if(annotation != null) {
            String token = request.getHeader("token");
            if (StringUtils.isBlank(token)) {// header中不存在token
                token = request.getParameter("token");
                if (StringUtils.isBlank(token)) {// parameter中也不存在token
                    throw new ServiceException("参数不合法");
                }
            }

            if (!jedisUtil.exists(token)) {
                System.err.println("请勿重复操作");
                throw new ServiceException("请勿重复操作");
            }

            result = joinPoint.proceed();

            Long del = jedisUtil.del(token);
            if (del <= 0) {
                throw new ServiceException("请勿重复操作");
            }

            return result;
        }
        return null;
    }
    
    @Before("execution(* com.cdc.bdom.portal..*.mapper.*Mapper.select*(..)) || execution(* com.cdc.bdom.portal..*.mapper.*Mapper.get*(..)) || execution(* com.cdc.bdom.portal..*.mapper.*Mapper.find*(..))")
			public void setReadDataSourceType() {
				logger.info("调用读数据库");
				DataSourceContextHolder.read();
			}
}
```

