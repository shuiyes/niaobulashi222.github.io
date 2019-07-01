---
layout: post
title: Spring Boot2(四)：使用Spring Boot多数据源实现读写分离
category: springboot
tags: [springboot]
copyright: java
---

## 前言

实际业务场景中，不可能只有一个库，所以就有了分库分表，多数据源的出现。实现了读写分离，主库负责增改删，从库负责查询。这篇文章将实现Spring Boot如何实现多数据源，动态数据源切换，读写分离等操作。

## 代码部署

快速新建项目spring-boot项目

### 1、添加maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.0.1</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
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
```



### 2、application配置多数据源读取配置

和之前教程一样，首先配置application.yml

```yml
#指定配置文件为test
spring:
  profiles:
    active: test

#配置Mybatis
mybatis:
  configuration:
    # 开启驼峰命名转换，如：Table(create_time) -> Entity(createTime)。不需要我们关心怎么进行字段匹配，mybatis会自动识别`大写字母与下划线`
    map-underscore-to-camel-case: true

#打印SQL日志
logging:
  level:
    com.niaobulashi.mapper.*: DEBUG
```

其中打印SQL日志这块，因为是多数据源，在mapper包下面区分不同的数据库来源xml文件，所以用*表示。

配置application-test.yml如下

```xml
spring:
  datasource:
    #主库
    master:
      jdbc-url: jdbc:mysql://127.0.0.1:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
      username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver
    #从库
    slave:
      jdbc-url: jdbc:mysql://127.0.0.1:3306/test2?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
      username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver
```

从spring.datasource节点开始，区分主库master，从库slave。主库连接的数据库为test，从库连接的数据库为test2。

**注意**：这里需要注意的是，从Spring Boot2开始，在配置多数据源时有些配置发生了变化，网上许多教程使用的是`spring.datasource.url`。会出现`jdbcUrl is required with driverClassName.`的问题。

**解决方法**：配置多数据源时，将`spring.datasource.url`配置改为`spring.datasource.jdbc-url`

### 3、添加主库配置信息

依据知名博主：纯洁的微笑，写的博文我们来分析一波

首先看主库配置的代码：

```java
@Configuration
@MapperScan(basePackages = "com.niaobulashi.mapper.master", sqlSessionTemplateRef = "masterSqlSessionTemplate")
public class DataSourceMasterConfig {

    /**
     * 是application-test.yml中的spring.datasource.master配置生效
     * @return
     */
    @Bean(name = "masterDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.master")
    @Primary
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    /**
     * 将配置信息注入到SqlSessionFactoryBean中
     * @param dataSource    数据库连接信息
     * @return
     * @throws Exception
     */
    @Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/master/*.xml"));
        return bean.getObject();
    }

    /**
     * 事务管理器，在实例化时注入主库master
     * @param dataSource
     * @return
     */
    @Bean(name = "masterTransactionManager")
    @Primary
    public DataSourceTransactionManager masterTransactionManager(@Qualifier("masterDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    /**
     * SqlSessionTemplate具有线程安全性
     * @param sqlSessionFactory
     * @return
     * @throws Exception
     */
    @Bean(name = "masterSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate masterSqlSessionTemplate(@Qualifier("masterSqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

**问题**：看这块`masterSqlSessionFactory`，`SqlSessionFactoryBean`只获取了`spring.datasource.master`数据库连接信息，并没有获取多数据库的配置信息`mybatis.configuration`导致我们需要配置驼峰命名规则，配置信息并没有注入到`SqlSessionFactoryBean`。这样就导致在查询是，遇到下划线无法解析相应字段user_id，dept_id，create_time

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis-mutli-04-01.png>)

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis-mutli-04-02.png>)

**解决方法**：在配置中添加Configuration

同时，将配置信息注入到SqlSessionFactoryBean

```java
/**
 * 将配置信息注入到SqlSessionFactoryBean中
 * @param dataSource    数据库连接信息
 * @return
 * @throws Exception
 */
@Bean(name = "slaveSqlSessionFactory")
public SqlSessionFactory slaveSqlSessionFactory(@Qualifier("slaveDataSource") DataSource dataSource) throws Exception {
    SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
    // 使配置信息加载到类中，再注入到SqlSessionFactoryBean
    org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
    configuration.setMapUnderscoreToCamelCase(true);
    bean.setConfiguration(configuration);
    bean.setDataSource(dataSource);
    bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/slave/*.xml"));
    return bean.getObject();
}
```

### 4、添加从库配置信息

和添加主库配置信息一样，只不过不同的是，不需要添加`@Primary`首选注解

代码如下

```java
@Configuration
@MapperScan(basePackages = "com.niaobulashi.mapper.slave", sqlSessionTemplateRef = "slaveSqlSessionTemplate")
public class DataSourceSlaveConfig {

    /**
     * 是application-test.yml中的spring.datasource.master配置生效
     * @return
     */
    @Bean(name = "slaveDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    /**
     * 将配置信息注入到SqlSessionFactoryBean中
     * @param dataSource    数据库连接信息
     * @return
     * @throws Exception
     */
    @Bean(name = "slaveSqlSessionFactory")
    public SqlSessionFactory slaveSqlSessionFactory(@Qualifier("slaveDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        // 使配置信息加载到类中，再注入到SqlSessionFactoryBean
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        configuration.setMapUnderscoreToCamelCase(true);
        bean.setConfiguration(configuration);
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/slave/*.xml"));
        return bean.getObject();
    }

    /**
     * 事务管理器，在实例化时注入主库master
     * @param dataSource
     * @return
     */
    @Bean(name = "slaveTransactionManager")
    public DataSourceTransactionManager slaveTransactionManager(@Qualifier("slaveDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    /**
     * SqlSessionTemplate具有线程安全性
     * @param sqlSessionFactory
     * @return
     * @throws Exception
     */
    @Bean(name = "slaveSqlSessionTemplate")
    public SqlSessionTemplate slaveSqlSessionTemplate(@Qualifier("slaveSqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

### 5、扩展配置方法会报错

在网上还看到这样一种配置，单独通过@ConfigurationProperties注解配置Mybatis的配置信息如下

```java
/**
 * 试application.yml中的mybatis.configuration配置生效，如果不主动配置，由于@Order配置顺序不同，讲导致配置不能及时生效
 * 使配置信息加载到类中，再注入到SqlSessionFactoryBean
 * @return
 */
@Bean
@ConfigurationProperties(prefix = "mybatis.configuration")
public org.apache.ibatis.session.Configuration configuration() {
    return new org.apache.ibatis.session.Configuration();
}
```

其中`prefix`，在主库和从库中的id是一样的，必须保持不同，否则idea就会提示报错`Duplicate prefix`

导致只有主库可以执行Mybatis的配置，从库无效。

```java
@Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource dataSource, org.apache.ibatis.session.Configuration configuration) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        // 使配置信息加载到类中，再注入到SqlSessionFactoryBean
        bean.setConfiguration(configuration);
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/master/*.xml"));
        return bean.getObject();
    }
```

这块验证只有主库有效，从库的驼峰方法解析无效。后续再来研究下。。。

### 6、数据层代码

代码结构如下

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis-mutli-04-03.png>)

其中SysUserMasterDao代码

```java
public interface SysUserMasterDao {
	
	/**
	 * 根据userId查询用户信息
	 * @param userId  用户ID
	 */
	List<SysUserEntity> queryUserInfo(Long userId);

	/**
	 * 查询所有用户信息
	 */
	List<SysUserEntity> queryUserAll();

	/**
	 * 根据userId更新用户的邮箱和手机号
	 * @return
	 */
	int updateUserInfo(SysUserEntity user);

}
```

### 7、resource下数据执行语句

SysCodeMasterDao.xml

```xml
<mapper namespace="com.niaobulashi.mapper.master.SysUserMasterDao">

    <!--查询所有用户信息-->
    <select id="queryUserAll" resultType="com.niaobulashi.entity.SysUserEntity">
        SELECT
            ur.*
        FROM
            sys_user ur
        WHERE
            1 = 1
    </select>

    <!--根据用户userId查询用户信息-->
    <select id="queryUserInfo" resultType="com.niaobulashi.entity.SysUserEntity">
        SELECT
            ur.*
        FROM
            sys_user ur
        WHERE
            1 = 1
          AND ur.user_id = #{userId}
    </select>

    <!-- 根据UserId，更新邮箱和手机号 -->
    <update id="updateUserInfo" parameterType="com.niaobulashi.entity.SysUserEntity">
        UPDATE sys_user u
        <set>
            <if test="email != null">
                u.email = #{email},
            </if>
            <if test="mobile != null">
                u.mobile = #{mobile},
            </if>
        </set>
        WHERE
        u.user_id = #{userId}
    </update>

</mapper>
```

8、Controller层测试

```java
@RestController
public class SysUserController {

    @Autowired
    private SysUserMasterDao sysUserMasterDao;

    @Autowired
    private SysUserSlaveDao sysUserSlaveDao;

    /**
     * 查询所有用户信息Master
     * @return
     */
    @RequestMapping("/getUserMasterAll")
    private List<SysUserEntity> getUserMaster() {
        System.out.println("查询主库");
        List<SysUserEntity> userList = sysUserMasterDao.queryUserAll();
        return userList;
    }

    /**
     * 查询所有用户信息Slave
     * @return
     */
    @RequestMapping("/getUserSlaveAll")
    private List<SysUserEntity> getUserSlave() {
        System.out.println("查询从库");
        List<SysUserEntity> userList = sysUserSlaveDao.queryUserAll();
        return userList;
    }

    /**
     * 根据userId查询用户信息Master
     * @return
     */
    @RequestMapping("/getUserMasterById")
    private List<SysUserEntity> getUserMasterById(@RequestParam(value = "userId", required = false) Long userId) {
        List<SysUserEntity> userList = sysUserMasterDao.queryUserInfo(userId);
        return userList;
    }

    /**
     * 根据userId查询用户信息Slave
     * @return
     */
    @RequestMapping("/getUserSlaveById")
    private List<SysUserEntity> getUserSlaveById(@RequestParam(value = "userId", required = false) Long userId) {
        List<SysUserEntity> userList = sysUserSlaveDao.queryUserInfo(userId);
        return userList;
    }

}
```

### 发送查询所有用户接口

主库：http://localhost:8080/getUserMasterAll

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis-mutli-04-04.png>)

从库：http://localhost:8080/getUserSlaveAll

![](<https://niaobulashi.github.io/assets/images/2019/springboot/mybatis-mutli-04-05.png>)

## 总结

1、通过多数据源方式实现数据库层面的读写分离

2、多数据源链接数据库是，使用spring.datasource.jdbc-url

3、多数据源的mybatis.configuration配置注意需要手动注入SqlSessionFactory



## [**示例代码-github**](<https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-13-multi-mybatis>)



