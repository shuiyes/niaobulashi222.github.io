---
layout: post
title: Spring Boot2(一)：使用Spring Boot2集成Mybatis基础搭建
category: springboot
tags: [springboot]
copyright: java
---

Mybatis 初期使用比较麻烦，需要各种配置文件、实体类、Dao 层映射关联、还有一大推其它配置。`mybatis-spring-boot-starter` 就是 Spring Boot+ Mybatis 可以完全注解不用配置文件，也可以简单配置轻松上手。

## mybatis-spring-boot-starter

官方说明：`MyBatis Spring-Boot-Starter will help you use MyBatis with Spring Boot`  
其实就是 Mybatis 看 Spring Boot 这么火热也开发出一套解决方案来凑凑热闹，但这一凑确实解决了很多问题，使用起来确实顺畅了许多。`mybatis-spring-boot-starter`主要有两种解决方案，一种是使用注解解决一切问题，一种是简化后的老传统。

当然任何模式都需要首先引入`mybatis-spring-boot-starter`的 Pom 文件，现在最新版本是 2.0.1

``` xml
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>2.0.1</version>
</dependency>
```

我一般使用的是XML极简模式，可能是由于之前用的hibernate用习惯了

## 极简 xml 版本

极简 xml 版本保持映射文件的老传统，接口层只需要定义空方法，系统会自动根据方法名在映射文件中找对应的 Sql .

### 1 添加相关 Maven 文件

``` xml
<dependencies>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.0.1</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

完整的 Pom 包这里就不贴了，大家直接看源码


### 2、`application.yml`相关配置

推荐使用`application.yml`进行配置，其实使用`application.yml`或者`application.properties`都是一样的效果，`application.yml`最终是转换为`application.properties`进行生效的，只不过`application.yml`视觉效果看起来更加明了。新建项目默认为`application.properties`，直接改为`application.yml`，另外新增一个`application-test.yml`用户不同环境使用不同的配置文件用。

`application.yml`配置：

``` properties
#指定配置文件为test
spring:
  profiles:
    active: test

#配置Mybatis
mybatis:
  type-aliases-package: com.niaobulashi.entity
  mapper-locations: classpath:mapper/*.xml
  configuration:
    # 开启驼峰命名转换，如：Table(create_time) -> Entity(createTime)。不需要我们关心怎么进行字段匹配，mybatis会自动识别`大写字母与下划线`
    map-underscore-to-camel-case: true

#打印SQL日志
logging:
  level:
    com.niaobulashi.dao: DEBUG
```

`application-test.yml`配置：

```properties
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
```



Spring Boot 会自动加载 `spring.datasource.*` 相关配置，数据源就会自动注入到 sqlSessionFactory 中，sqlSessionFactory 会自动注入到 Mapper 中，对了，你一切都不用管了，直接拿起来使用就行了。

在启动类中添加对 mapper 包扫描`@MapperScan`

``` java
@SpringBootApplication
@MapperScan("com.niaobulashi.dao")
public class MybatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(MybatisApplication.class, args);
    }

}
```

或者直接在 Mapper 类上面添加注解`@Mapper`，建议使用上面那种，不然每个 mapper 加个注解也挺麻烦的

### 3、添加 User 的映射文件

```java
@Data
public class SysUserEntity implements Serializable {
	private static final long serialVersionUID = 1L;
	//用户ID
	private Long userId;

	//用户名
	private String username;

	//密码
	private String password;

	//盐
	private String salt;

	//邮箱
	private String email;

	//手机号
	private String mobile;

	//状态  0：禁用   1：正常
	private Integer status;
	
	//创建时间
	private Date createTime;
}
```



### 4、添加 User 的映射文件

``` xml
<mapper namespace="com.niaobulashi.dao.SysUserDao">

    <!--查询用户的所有菜单ID-->
    <select id="queryUserInfo" resultType="com.niaobulashi.entity.SysUserEntity">
        SELECT
            ur.*
        FROM
            sys_user ur
        WHERE
            1 = 1
          AND ur.user_id = #{userId}
    </select>

</mapper>
```

其实就是把上个版本中 Mapper 的 Sql 搬到了这里的 xml 中了


### 5、编写 Mapper 层的代码

``` java
public interface SysUserDao {
	/**
	 * 根据userId查询用户信息
	 * @param userId  用户ID
	 */
	List<SysUserEntity> queryUserInfo(Long userId);
}
```

### 5、编写Service层的代码

`SysUserService`接口类：

```java
public interface SysUserService {
	/**
	 * 查询用户的所有菜单ID
	 */
	List<SysUserEntity> queryUserInfo(Long userId);
}
```

`SysUserServiceImpl`实现类：

```java
@Service("sysUserService")
public class SysUserServiceImpl  implements SysUserService {
	@Resource
	private SysUserDao sysUserDao;
	/**
	 * 查询用户的所有菜单ID
	 * @param userId
	 * @return
	 */
	@Override
	public List<SysUserEntity> queryUserInfo(Long userId) {
		return sysUserDao.queryUserInfo(userId);
	}
}
```


### 7、测试

经过上面5个步骤就可以完成基本的接口开发，省去了Controller层的开发

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MabatisTest {
    private final static Logger logger = LoggerFactory.getLogger(MabatisTest.class);

    @Autowired
    private SysUserService sysUserService;

    @Test
    public void queryUserInfo() throws Exception {
        SysUserEntity userEntity = new SysUserEntity();
        userEntity.setUserId(1L);
        List<SysUserEntity> list = sysUserService.queryUserInfo(userEntity.getUserId());
        logger.info("list:" + list);
    }

}
```

## 总结

SpringBoot和Mybatis这对CP，完美

**[示例代码-github](https://github.com/niaobulashi/spring-boot-mybatis/tree/master/mybatis_01_hello)**
