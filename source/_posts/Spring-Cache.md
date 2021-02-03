---
title: Spring Cache
date: 2021-02-01 15:55:27
tags: Spring
categories: Spring
---

## Spring Cache 简介

在 Spring 3.1 中引入了多 Cache 的支持，在 spring - context 包中定义了 `org.springframework.cache.Cache` 和 `org.springframework.cache.CacheManager` 两个接口来统一不同的缓存技术。Cache 接口包含缓存的常用操作：增加、删除、读取等。CacheManager 是 Spring 各种缓存的抽象接口。Spring 支持的常用 CacheManager 如下：



| CacheManager              | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| SimpleCacheManager        | 使用简单的 Collection 来存储缓存                             |
| ConcurrentMapCacheManager | 使用 java.util.ConcurrentHashMap 来实现缓存                  |
| NoOpCacheManager          | 仅测试用，不会实际存储缓存                                   |
| EhCacheCacheManager       | 使用 EhCache 作为缓存技术。EhCache 是一个纯 Java 的进程内缓存框架，特点是快速、精干，是 Hibernate 中默认的 CacheProvider，也是 Java 领域应用最为广泛的缓存 |
| JCacheCacheManager        | 支持 JCache（JSR - 107）标准的实现作为缓存技术               |
| CaffeineCacheManager      | 使用 Caffeine 作为缓存技术。用于取代 Guava 缓存技术          |
| RedisCacheManager         | 使用 Redis 作为缓存技术                                      |
| HazelcastCacheManager     | 使用 Hazelcast 作为缓存技术                                  |
| CompositeCacheManager     | 用于组合 CacheManager，可以从多个 CacheManager 中轮询得到相应的缓存 |



Spring Cache 提供了 `@Cacheable`、`@CachePut`、`@CacheEvict`、`@Caching` 等注解，在方法上使用。通过注解 Cache 可以实现类似事务一样，缓存逻辑透明地应用到我们的业务代码上，且只需要更少的代码。核心思想：**当我们调用一个方法时会把该方法的参数和返回结果作为一个键值对放在缓存中，等下次利用同样的参数来调用该方法时就不会再执行，而是直接从缓存中获取结果进行返回**



## Cache 注解

### @EnableCaching

开启缓存功能，一般放在启动类上



### @CacheConfig

当我们需要缓存的地方越来越多，你可以使用 `@CacheConfig(cacheNames = {"cacheName"})` 注解在 class 之上来统一制定 value 的值，这时可省略 value。如果你在你的方法依旧写上了 value，那么依然以方法的 value 值为准



### @Cacheable

根据方法对其返回结果进行缓存，下次请求时，如果缓存存在，则直接读取缓存数据返回；如果缓存不存在，则执行方法，并把返回的结果存入缓存中。**一般用在查询方法上**。有以下属性值：



| 属性/方法值   | 解释                                           |
| ------------- | ---------------------------------------------- |
| value         | 缓存名，必填，它指定了你的缓存放在哪块命名空间 |
| CacheNames    | 与 value 差不多，二选一即可                    |
| key           | 可选属性，可以使用 SpEL 标签自定义缓存的 key   |
| keyGenerator  | key 的生成器。key / keyGenerator 二选一使用    |
| cacheManager  | 指定缓存管理器                                 |
| cacheResolver | 指定获取解析器                                 |
| condition     | 条件符合则缓存                                 |
| unless        | 条件符合则不缓存                               |
| sync          | 是否使用异步模式。默认为 false                 |



### @CachePut

使用该注解标志的方法，每次都会执行，并将结果存入指定的缓存中。其它方法可以直接从响应的缓存中读取缓存数据，而不需要再去查询数据库。**一般用在新增方法上**。属性值如下：





| 属性/方法值   | 解释                                           |
| ------------- | ---------------------------------------------- |
| value         | 缓存名，必填，它指定了你的缓存放在哪块命名空间 |
| CacheNames    | 与 value 差不多，二选一即可                    |
| key           | 可选属性，可以使用 SpEL 标签自定义缓存的 key   |
| keyGenerator  | key 的生成器。key / keyGenerator 二选一使用    |
| cacheManager  | 指定缓存管理器                                 |
| cacheResolver | 指定获取解析器                                 |
| condition     | 条件符合则缓存                                 |
| unless        | 条件符合则不缓存                               |



### @CacheEvict

使用该注解标志的方法，会清空指定的缓存，**一般用在更新或者删除方法上**。属性值如下：





| 属性/方法值      | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| value            | 缓存名，必填，它指定了你的缓存放在哪块命名空间               |
| CacheNames       | 与 value 差不多，二选一即可                                  |
| key              | 可选属性，可以使用 SpEL 标签自定义缓存的 key                 |
| keyGenerator     | key 的生成器。key / keyGenerator 二选一使用                  |
| cacheManager     | 指定缓存管理器                                               |
| cacheResolver    | 指定获取解析器                                               |
| condition        | 条件符合则缓存                                               |
| allEntries       | 是否清空所有缓存，默认为 false。如果为 true，则方法调用后将立即清空所有缓存 |
| beforeInvocation | 是否在方式执行前就清空，默认为 false。如果为 true，则在方法执行后就清空缓存 |



### @Caching

该注解可以实现同一个方法上同时使用多种注解。从其源码可以看出：



```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Caching {
    Cacheable[] cacheable() default {};

    CachePut[] put() default {};

    CacheEvict[] evict() default {};
}
```



## Spring Cache 使用

接着我们用一个 demo 来看一下 Spring Cache 如何使用



### 构件项目，添加依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>spring-cache</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-cache</name>
    <description>Demo project for Spring Cache</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <!-- spring cache -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- mybatis-plus -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.2</version>
        </dependency>
        <!-- jdbc -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.20</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```



### application.properties 配置文件

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/db_test?useUnicode=true&characterEncoding=UTF-8&useSSL=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=123456

mybatis-plus.mapper-locations=classpath:/mapper/*Mapper.xml
```



### 实体类

```java
package com.example.springcache.entity;

import lombok.Data;
import lombok.ToString;

@Data
@ToString
public class UserEntity {

    private Long id;

    private String userName;

    private String userSex;
}
```



### 数据层 dao 和 mapper.xml

```java
@Mapper
public interface UserDao {

    /**
     * 获取所有用户
     * @return
     */
    List<UserEntity> getAll();

    /**
     * 根据id获取用户
     * @param id
     * @return
     */
    UserEntity getOne(Long id);

    /**
     * 新增用户
     * @param user
     */
    void insertUser(UserEntity user);

    /**
     * 修改用户
     * @param user
     */
    void updateUser(UserEntity user);

    /**
     * 删除用户
     * @param id
     */
    void deleteUser(Long id);
}
```



```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.4//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.springcache.dao.UserDao">
    <resultMap type="com.example.springcache.entity.UserEntity" id="user">
        <id property="id" column="id"/>
        <result property="userName" column="user_name"/>
        <result property="userSex" column="user_sex"/>
    </resultMap>
    <!-- 获取所有用户 -->
    <select id="getAll" resultMap="user">
        select * from t_user
    </select>
    <!-- 根据用户ID获取用户 -->
    <select id="getOne" resultMap="user">
        select * from t_user where id=#{id}
    </select>
    <!-- 新增用户 -->
    <insert id="insertUser" parameterType="com.example.springcache.entity.UserEntity">
        insert into t_user (id, user_name,user_sex) values(#{id},#{userName},#{userSex})
    </insert>
    <!-- 修改用户 -->
    <update id="updateUser" parameterType="com.example.springcache.entity.UserEntity">
        update t_user set user_name=#{userName},user_sex=#{userSex} where id=#{id}
    </update>
    <!-- 删除用户 -->
    <delete id="deleteUser" parameterType="Long">
        delete from t_user where id=#{id}
    </delete>
</mapper>
```



### 业务代码层接口 Service 和实现类 ServiceImpl

```java
public interface UserService {

    /**
     * 获取所有用户
     * @return
     */
    List<UserEntity> getAll();

    /**
     * 根据id获取用户
     * @param id
     * @return
     */
    UserEntity getOne(Long id);

    /**
     * 新增用户
     * @param user
     */
    void insertUser(UserEntity user);

    /**
     * 修改用户
     * @param user
     */
    void updateUser(UserEntity user);


    void deleteAll1();

    void deleteAll2();
}
```



```java
public interface UserService2 {

    /**
     * 删除用户列表
     */
    void deleteUserList();
}
```



```java
@Service
@CacheConfig(cacheNames = {"userCache"})
public class UserServiceImpl implements UserService {
    @Resource
    private UserDao userDao;

    @Autowired
    private UserService2 userService2;


    @Override
    @Cacheable("userList")  // 标志读取缓存操作，如果缓存不存在，则调用目标方法，并将结果放入缓存
    public List<UserEntity> getAll() {
        System.out.println("缓存不存在，执行方法");
        return userDao.getAll();
    }

    @Override
    @Cacheable(cacheNames = {"user"}, key = "#id") // 如果缓存存在，直接读取缓存值；如果不存在，则调用目标方法，并将结果缓存缓存
    public UserEntity getOne(Long id) {
        System.out.println("缓存不存在，执行方法");
        return userDao.getOne(id);
    }

    @Override
    @CachePut(cacheNames = {"user"}, key = "#user.id") // 写入缓存，key 为 user.id，一般该注解标注在新增方法上
    public void insertUser(UserEntity user) {
        System.out.println("写入缓存");
        userDao.insertUser(user);
    }

    @Override
    @CacheEvict(cacheNames = {"user"}, key = "#user.id") // 根据 key 清除缓存，一般该注解标注在修改和删除方法上
    public void updateUser(UserEntity user) {
        System.out.println("清除缓存");
        userDao.updateUser(user);
        userService2.deleteUserList();
    }

    @Override
    @CacheEvict(value = "userCache", allEntries = true) // 方法调用后清空所有缓存
    public void deleteAll1() {

    }

    @Override
    @CacheEvict(value = "userCache", beforeInvocation = true) // 方法调用前清空所有缓存
    public void deleteAll2() {

    }
}
```



```java
@Service
@CacheConfig(cacheNames = {"userCache"})
public class UserService2Impl implements UserService2 {

    @Override
    @CacheEvict("userList")
    public void deleteUserList() {

    }
}
```



### 测试 Controller

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @RequestMapping("/getAll")
    public List<UserEntity> getAll() {
        return userService.getAll();
    }

    @RequestMapping("/getOne")
    public UserEntity getOne(Long id) {
        UserEntity user = userService.getOne(id);
        System.out.println(user);
        return user;
    }

    @RequestMapping("insertUser")
    public String insertUser(UserEntity user) {
        userService.insertUser(user);
        return "insert success";
    }

    @RequestMapping("updateUser")
    public String updateUser(UserEntity user) {
        userService.updateUser(user);
        return "update success";
    }
}
```



### 启动 Cache 功能

```java
@SpringBootApplication
@EnableCaching
public class SpringCacheApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCacheApplication.class, args);
    }

}
```



## Spring Cache 整合 Caffeine

在不做任何配置的情况下，默认使用的是 `SimpleCacheConfiguration`，即使用 `ConcurrentMapCacheManager`。如果要使用其它的缓存框架，要怎么做？我们只需重新定义好 `CacheManager` 即可。我们在上面的例子中，将默认的缓存框架切换为 `Caffeine`



### EhCache、Guava、Caffeine 对比

这三个都是作为进程缓存（本地缓存）的优秀开源产品，如果我们要使用本地缓存来加速访问，选择哪种？下面做一个简单的对比：



1. `EhCache`：是一个纯 Java 的进程内存缓存框架，具有快速、精干等特点，是 Hibernate、Mybatis 默认的缓存提供（虽然 EhCache3 支持到了分布式，但它还是基于 Java 进程的缓存）
2. `Guava`：它是 Google Guava 工具包中一个非常方便易用的本地化缓存实现，基于 LRU 算法实现，支持多种缓存过期策略
3. `Caffeine`：它是使用 Java8 对 Guava 缓存的重写版本，在 Spring5 中取代了 Guava，支持多种缓存过期策略



Caffeine 在性能上碾压其余两者。它可以完全地替代 Guava，因为 API 上都差不多一致，并且它还提供了 Adapter 让 Guava 过度到 Caffeine 上来。Caffeine 被称为**进程缓存之王**



### 引入 jar 包

想要整合 Caffeine，首先先在 `pom` 文件引入 Caffeine 的 jar 包



```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```



### 配置缓存配置类

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterAccess(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000));

        return cacheManager;
    }
}
```



这样，Spring Cache 默认的缓存框架就切换成 Caffeine 了



## 参考资料

[Spring Cache 使用](https://juejin.cn/post/6844903966615011335)