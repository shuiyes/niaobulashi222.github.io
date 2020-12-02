---
layout: post
title: Spring Boot2(三)：使用Spring Boot2集成Redis缓存
category: springboot
tags: [springboot]
copyright: java
---

前面一节总结了[SpringBoot实现Mybatis的缓存机制](<https://niaobulashi.github.io/springboot/2019/06/28/mybatis-2levelcache.html>)，但是实际项目中很少用到Mybatis的二级缓存机制，反而用到比较多的是第三方缓存[Redis](<https://redis.io/>)。

**Redis**是一个使用ANSI C编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库。

## 一、安装启动Redis

安装Redis的就不讲太多了，直接去[官方下载redis](<https://github.com/microsoftarchive/redis/releases/tag/win-3.2.100>)，下载[Redis-x64-3.2.100.zip](https://github.com/microsoftarchive/redis/releases/download/win-3.2.100/Redis-x64-3.2.100.zip)，cmd，在redis目录下输入：redis-server.exe redis.windows.conf启动即可

另外可以通过Redis桌面客户端可视化连接工具操作：[redisdesktop](<https://redisdesktop.com/>)

## 二、代码部署

快速建立Spring Boot项目

### 添加redis依赖

```
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### application.yml配置

```
spring:
  redis:
    host: 127.0.0.1
    database: 0
    password:
    port: 6379
    jedis:
      pool:
        max-active: 1000  # 连接池最大连接数（使用负值表示没有限制）
        max-wait: -1ms  # 连接池最大阻塞等待时间（使用负值表示没有限制）
        max-idle: 10  # 连接池中的最大空闲连接
        min-idle: 5 # 连接池中的最小空闲连接
```

### RedisConfig配置类

```
@Autowired
private RedisConnectionFactory factory;

/**
 *
 * @return
 */
@Bean
public RedisTemplate<String, Object> redisTemplate() {
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    //更改在redis里面查看key编码问题
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setHashKeySerializer(new StringRedisSerializer());
    redisTemplate.setHashValueSerializer(new StringRedisSerializer());
    redisTemplate.setValueSerializer(new JdkSerializationRedisSerializer());
    redisTemplate.setConnectionFactory(factory);
    return redisTemplate;
}
```

### RedisUtils工具类

```
@Autowired
private RedisTemplate redisTemplate;
// 简单的K-V操作
@Resource(name="redisTemplate")
private ValueOperations<String, String> valueOperations;

// 针对Map类型的数据操作
@Resource(name="redisTemplate")
private HashOperations<String, String, Object> hashOperations;

// 针对List类型的数据操作
@Resource(name="redisTemplate")
private ListOperations<String, Object> listOperations;

// set类型数据操作
@Resource(name="redisTemplate")
private SetOperations<String, Object> setOperations;

// zset类型数据操作
@Resource(name="redisTemplate")
private ZSetOperations<String, Object> zSetOperations;
```

### 实体类SysCodeEntity

```
@Data
public class SysCodeEntity implements Serializable {

    private static final long serialVersionUID = 1L;

    private int id;

    // 分类编码
    private String kindCode;

    // 分类名称
    private String kindName;

    // CODE编码
    private String code;
	......
}
```

### ServiceImpl实现类

```
/**
 * 查询所有数字字典
 * @return
 */
@Override
public List<SysCodeEntity> queryCodeAll() {
    logger.info("先从缓存中查找，如果没有则去数据进行查询");
    List<SysCodeEntity> codeList = (List<SysCodeEntity>)redisTemplate.opsForList().leftPop("codeList");
    if (codeList == null) {
        logger.info("说明缓存中没有数据，则到数据库中查询");
        codeList = sysCodeDao.queryCodeAll();

        logger.info("将数据库获取的数据存入缓存");
        redisTemplate.opsForList().leftPush("codeList", codeList);
    } else {
        logger.info("则说明缓存中存在，直接从缓存中获取数据");
    }
    logger.info("codeList=" + codeList);
    return codeList;
}
```

上面例子具体解释已经在注释中体现，通过opsForList的leftPop和leftPush存入和获取Redis缓存的数据。

### Controller层实现

```
/**
 * 查询所有数字字典
 * @return
 */
@RequestMapping("/getAll")
private List<SysCodeEntity> getUser() {
    Long startTime = System.currentTimeMillis(); //开始时间
    List<SysCodeEntity> codeList = sysCodeService.queryCodeAll();
    Long endTime = System.currentTimeMillis(); //结束时间
    System.out.println("查询数据库--共耗时：" + (endTime - startTime) + "毫秒"); //1007毫秒
    return codeList;
}
```

### Postman进行测试

```
http://localhost:8080/getAll
```

### 日志信息

![](<https://niaobulashi.github.io/assets/images/2019/springboot/springboot_03_01.png>)

## 三、总结和扩展

1、Redis支持：字符串String、哈希Hash、列表List、集合Set、有序集合Sorted Set、发布订阅Pub/Sub、事务Transactions，7种数据类型

2、Redis实用场景：缓存系统、计数器、消息列队系统、排行版及相关问题、社交网络、按照用户投票和时间排序、过期项目处理、实时系统

3、Redis的高级功能：慢查询（内部执行时间超过某个指定的时限查询）、PipeLine管道（降低客户端与redis通信次数，适用于批处理）、BitMap位图（针对大数据量设计）、HyperLogLog（极小空间完成独立数据统计）、发布订阅、消息队列、GEO地理位置存储

4、Redis持久化：

​    快照RDB（使用[快照](https://zh.wikipedia.org/w/index.php?title=%E5%BF%AB%E7%85%A7&action=edit&redlink=1)，一种半持久耐用模式。不时的将数据集以异步方式从内存以RDB格式写入硬盘）

​    日志AOF（1.1版本开始使用更安全的AOF格式替代，一种只能追加的日志类型。将数据集修改操作记录起来。Redis能够在后台对只可追加的记录作修改来避免无限增长的日志）

5、Redis分布式解决方案：主从复制、集群...



## [**示例代码-github**](<https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-12-mybatis-redis>)



后期持续探索Redis技术 To be continued...



推荐阅读：[**Redis 入门到分布式实践**](<https://gitbook.cn/gitchat/column/5a55d8e232c7126d8482f5d2>)

------

