---
layout: post
title: Spring Boot2(二)：使用Spring Boot2集成Mybatis缓存机制
category: springboot
tags: [springboot]
copyright: java
---

## 前言

学习SpringBoot集成Mybatis的第二章，了解到Mybatis自带的缓存机制，在部署的时候踩过了一些坑。在此记录和分享一下Mybatis的缓存作用。

 本文章的源码再文章末尾

## 什么是查询缓存

MyBatis有一级缓存和二级缓存。记录可以看下这篇博文：

[聊聊MyBatis缓存机制]: https://niaobulashi.github.io/mybatis/2019/06/27/mybatis-cache.html	"聊聊MyBatis缓存机制"

### 一级缓存

首先看一下什么是一级缓存，一级缓存是指SqlSession。一级缓存的作用域是一个SqlSession。Mybatis默认开启一级缓存。

在同一个SqlSession中，执行相同的查询SQL，第一次会去查询数据库，并写到缓存中；第二次直接从缓存中获取。当执行SQL查询前后发生增删改操作时，则SqlSession的缓存清空。

具体可以看这段代码：

```java
@Test
public void testLocalCacheScope() throws Exception {
        SqlSession sqlSession1 = factory.openSession(true); 
        SqlSession sqlSession2 = factory.openSession(true); 

        StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
        StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);

        System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
        System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
        System.out.println("studentMapper2更新了" + studentMapper2.updateStudentName("小岑",1) + "个学生的数据");
        System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
        System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
}
```

开启两个sqlSession

![](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2018a/f480ac76.jpg)

从打印日志可以看出，前面两个说明sqlSession1的会话缓存生效了，第三个对sqlSession2会话执行了更新操作，这时候数据库发生数据变化，sqlSession2被清空。可是在执行第四个查询是，是查询的sqlSession1会话，由于sqlSession1没有被清空，所以还是查询的缓存的数据，是数据更新之前的，查询的是脏数据，一级缓存sqlSession是不共享的。证明了一级缓存只是在数据库会话内部共享的。

### 二级缓存

Mybatis的二级缓存是指mapper映射文件。二级缓存的作用域是同一个namespace下的mapper映射文件内容，**多个SqlSession共享**，Mybatis需要手动设置二级缓存。

在同一个namespace下的mapper文件中，执行相同的查询SQL，第一次会查询数据库，并写道缓存中；第二次z直接从缓存中获取。当执行SQL查询前后发生增删改操作时，则二级缓存清空。

上面说到二级缓存可以共享多个SqlSession。可以解决不同SqlSession回话中查询到脏数据的问题了。



## SpringBoot整合Mybatis开启二级缓存

首先，Mybatis默认是开启一级缓存的，即同一个SqlSession每次查询都会去缓存中查询，没有数据的话，再去数据库获取数据。**但是，整合到SpringBoot中后，一级缓存就会被关闭**。为什么会出现这种原因呢，可以看下这篇文章：

[spring整合mybatis后,mybatis一级缓存失效的原因]: spring整合mybatis后,mybatis一级缓存失效的原因

好了，现在来创建项目，可以根据前一篇文章来创建项目，在这基础上修改

[Spring Boot2(一)：使用Spring Boot2集成Mybatis基础搭建]: https://niaobulashi.github.io/springboot/2019/06/26/mybatis-hello.html



### pom.xml新增mybatis缓存包caches

```xml
<dependency>
	<groupId>org.mybatis.caches</groupId>
	<artifactId>mybatis-ehcache</artifactId>
	<version>1.1.0</version>
</dependency>
```

### SysUserDao.xml添加开启Mybatis二级缓存

```xml
<cache />
```

加上这个标签，二级缓存就会开启，他的默认属性如下

- 映射语句文件中的所有 select 语句将会被缓存。
- 映射语句文件中的所有 insert,update 和 delete 语句会刷新缓存。
- 缓存会使用 Least Recently Used(LRU,最近最少使用的)算法来收回。
- 根据时间表(比如 no Flush Interval,没有刷新间隔), 缓存不会以任何时间顺序来刷新。
- 缓存会存储列表集合或对象(无论查询方法返回什么)的 1024 个引用。
- 缓存会被视为是 read/write(可读/可写)的缓存,意味着对象检索不是共享的,而且可以安全地被调用者修改,而不干扰其他调用者或线程所做的潜在修改。

  也可以自定义二级缓存的属性，例如：

```xml
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

   这个更高级的配置创建了一个 FIFO 缓存,并每隔 60 秒刷新,存数结果对象或列表的 512 个引用,而且返回的对象被认为是只读的,因此在不同线程中的调用者之间修改它们会 导致冲突。

​        可用的收回策略有:

- LRU – 最近最少使用的:移除最长时间不被使用的对象。
- FIFO – 先进先出:按对象进入缓存的顺序来移除它们。
- SOFT – 软引用:移除基于垃圾回收器状态和软引用规则的对象。
- WEAK – 弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象。

默认的是 LRU。

  flushInterval(刷新间隔)可以被设置为任意的正整数,而且它们代表一个合理的毫秒 形式的时间段。默认情况是不设置,也就是没有刷新间隔,缓存仅仅调用语句时刷新。

​        size(引用数目)可以被设置为任意正整数,要记住你缓存的对象数目和你运行环境的 可用内存资源数目。默认值是 1024。

​        readOnly(只读)属性可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓 存对象的相同实例。因此这些对象不能被修改。这提供了很重要的性能优势。可读写的缓存 会返回缓存对象的拷贝(通过序列化) 。这会慢一些,但是安全,因此默认是 false。

## 测试验证

编写Controller接口

```java
/**
 * 查询所有用户信息
 * @return
 */
@RequestMapping("/getAll")
private List<SysUserEntity> getUser() {
    List<SysUserEntity> userList = sysUserService.queryUserAll();
    return userList;
}

/**
 * 根据userId查询用户信息
 * @return
 */
@RequestMapping("/getUser")
private List<SysUserEntity> getUser(@RequestParam(value = "userId", required = false) Long userId) {
    List<SysUserEntity> userList = sysUserService.queryUserInfo(userId);
    return userList;
}

/**
 * 更新用户信息
 * @param user
 * @return
 */
@RequestMapping("/updateUser")
private int updateUser(@RequestBody SysUserEntity user) {
    return sysUserService.updateUserInfo(user);
}
```

通过postman发送接口请求进行测试：

- 1、发送查询用户全部信息：http://localhost:8080/getAll

- 2、根据userId查询用户信息：http://localhost:8080/getUser?userId=1

- 3、更新用户信息http://localhost:8080/updateUser

  更新用户信息接口发送报文:

```json
{
	"userId":5,
	"email":"12321321",
	"mobile":"11111111111213"
}
```

通过日志可以看到，第一次发送1接口请求，对数据库进行了查询

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis_02_01.png>)

可以看到，第二次和第三次查询没有查询数据库的SQL打印，而是去数据库获取数据

此时发送3接口，进行更新操作，在发送1接口，查询改用户的数据

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis_02_02.png>)

可以看到，当执行数据库更新操作后，再进行查询，此时缓存已经清空，需要从数据库中重新查询获取。

这就演示了SpringBoot整合Mybatis的缓存机制测试。

## 总结

1、缓存的对象必须实现序列化。因为二级缓存的数据不一定都是存储到内存中，它的存储介质多种多样，所以需要给缓存的对象执行序列化，才可以确保获取无误。

2、Mybatis的二级缓存相比于一级缓存来说，实现了SqlSession之间的缓存数据的共享，做到namespace级别，粒度更细

3、在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis、Memcached等分布式缓存可能成本更低，安全性也更高。

不过建议Mybatis的缓存特性再生产环境下进行关闭，单纯作为一个[ORM框架](https://www.cnblogs.com/wisdo/p/4279091.html)使用可能更加合适。



下篇文章计划写SpringBoot整合Mybatis，使用Redis实现缓存基本配置。



**[示例代码-github](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-11-mybatis-cache)**


---
关于作者：

个人博客：[鸟不拉屎](https://niaobulashi.com)

github主页：[niaobulashi](https://github.com/niaobulashi)

github博客：[鸟不拉屎](https://niaobulashi.github.io)

掘金：[鸟不拉屎](https://juejin.im/user/5b3de9155188251aa0161fe4)

博客园：[鸟不拉屎](https://www.cnblogs.com/niaobulashi)

知乎：[鸟不拉屎](https://www.zhihu.com/people/hu-lang-lang-91/)

微博：[胡浪同學](https://www.weibo.com/godloveharry)

