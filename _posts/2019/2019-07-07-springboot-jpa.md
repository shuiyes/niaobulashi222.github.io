---
layout: post
title: Spring Boot2(九)：整合Jpa的基本使用
category: springboot
tags: [springboot]
copyright: java
---

## 一、前言

今天早上看到一篇微信文章，说的是国内普遍用的Mybatis，而国外确普遍用的是Jpa。我之前也看了jpa，发现入门相当容易。jpa对于简单的CRUD支持非常好，开发效率也会比Mybatis高出不少，因为`JpaRepository`会根据你定制的实体类，继承了`JpaRepository`会有一套完整的封装好了的基本条件方法。减少了很多开发量。你只需要写SQL就行了。可能我才刚入门Jpa，对一些认识还是很浅显。我觉得Jpa对于多表查询，开发起来有点吃力。。

这是我开始玩Jpa的最初的感受，但是Jpa却受到了极大的支持和赞扬，在国外Jpa远比Mybatis流行得多得多。国内却还是在流程用Mybatis，估计也是收到很多培训机构或者大V的带领下，很多国内优秀的开源项目也是用的Mybatis，因为已经用得非常熟练了。

话不多说，先看看SpringBoot如何整合使用Jpa吧！

这里具体讲一讲Jpa的搭建，几种常见的场景的使用：增删改查、多表查询，非主键查询这几种情况的一个学习总结。

## 二、代码部署

### 1、添加Maven依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-devtools</artifactId>
	<scope>runtime</scope>
	<optional>true</optional>
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
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
</dependency>
```

其实Jpa关键用到的是最下面两块

### 2、配置application

`application.yml`

```yaml
server:
  port: 8081
#指定配置文件为test
spring:
  profiles:
    active: test
```

`application-test.yml`

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/jpatest?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: root
  jpa:
    # 数据库类型
    database: mysql
    #打印SQL
    show-sql: true
    hibernate:
      ddl-auto: update  #第一次启动创建表，之后修改为update
```

`application-test.yml`需要了解的是jpa分支，如果需要通过jpa在数据库中建表，就将`spring.jpa.hibernate.ddl-auto`改为`create`，建完表之后，建议改为update，否则你再次重启，表会回炉重造，数据相应的会丢失。可得注意啦。

### 3、创建实体类

用户表sys_user的实体类

```java
@Data
@Entity
@Table(name = "sys_user")
public class SysUser implements Serializable {

    @Id
    private String userId;

    @Column(nullable = false)
    private String userName;

    @Column(nullable = false)
    private String passWord;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false, unique = true)
    private String salt;

    @Column(nullable = false)
    private Date regTime;
    
}
```

用户角色对照表sys_user_role的实体类

```
@Entity
@Data
@Table(name = "sys_user_role")
public class SysUserRole implements Serializable {

    @Id
    @GeneratedValue
    private int id;

    // 用户ID
    private String userId;

    // 角色ID
    private int roleId;
}
```

### 4、Dao层

用户表SysUserDao

```java
public interface SysUserDao extends JpaRepository<SysUser, Integer> {
    
}
```

用户角色对照表SysUserRoleDao

```java
public interface SysUserRoleDao extends JpaRepository<SysUserRole, Integer> {
}
```

### 5、Controller层

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private SysUserDao sysUserDao;

    @Autowired
    private SysUserRoleDao sysUserRoleDao;

    /**
     * 用户表sys_user，用户角色对照表sys_user_role。数据初始化
     */
    //发送get请求进行数据添加：127.0.0.1:8081/user/init
    @RequestMapping(value = "/init", method = RequestMethod.GET)
    public String initData() {
        for (int i = 1; i < 6; i++) {
            // 根据时间戳生成userId
            String userId = String.valueOf(System.currentTimeMillis());
            // new出用户表和用户角色表的对象
            SysUser sysUser = new SysUser();
            SysUserRole sysUserRole = new SysUserRole();
            // 新增用户表
            sysUser.setUserId(userId);
            sysUser.setUserName("username_num" + i);
            sysUser.setPassWord("password_num" + i);
            sysUser.setEmail("email_num" + i + "@qq.com");
            sysUser.setSalt(i + "");
            sysUser.setRegTime(new Date());
            sysUserDao.save(sysUser);

            // 暂时规定小于3的，角色为1，新建用户角色表
            if (i < 3) {
                sysUserRole.setId(i);
                sysUserRole.setUserId(userId);
                sysUserRole.setRoleId(1);
                sysUserRoleDao.save(sysUserRole);
            } else {
                // 大于3的，角色为2
                sysUserRole.setId(i);
                sysUserRole.setUserId(userId);
                sysUserRole.setRoleId(2);
                sysUserRoleDao.save(sysUserRole);
            }
        }
        return "init data success";
    }

    /**
     * 删除
     */
    // 发送get请求：127.0.0.1:8081/user/delete/1562486017644
    @RequestMapping(value = "/delete/{userId}", method = RequestMethod.GET)
    public String deleteUser(@PathVariable("userId") String userId) {
        sysUserDao.deleteByUserId(userId);
        return "delete success";
    }

    /**
     * 查询全部
     * @return
     */
    // 发送get请求：127.0.0.1:8081/user/list
    @RequestMapping(value = "/list", method = RequestMethod.GET)
    public List<SysUser> getUsers() {
        return sysUserDao.findAll();
    }

    /**
     * 根据id查询
     */
    // 发送get请求：127.0.0.1:8081/user/info/1562486017644
    @RequestMapping(value = "/info/{userId}", method = RequestMethod.GET)
    public Optional<SysUser> getUserById(@PathVariable("userId") String userId) {
        return sysUserDao.findByUserId(userId);
    }

    /**
     * 更新
     */
    // 发送post请求：127.0.0.1:8081/user/update
    // 发送报文体如下
    /*
     {
        "userId":"1562486017551",
        "passWord": "231231231212312",
        "userName":"Tom",
        "email": "1111111@qq.com"
     }
     */
    @RequestMapping(value = "/update", method = RequestMethod.POST)
    public String updateAccount(@RequestBody HashMap<String, String> map) {
        // 根据Id更新用户信息
        sysUserDao.updateOne(
                map.get("email"),
                map.get("userName"),
                map.get("passWord"),
                map.get("userId"));
        return "update success";
    }

    /**
     * 关联查询用户的角色信息
     */
    // 发送post请求：127.0.0.1:8081/user/getUserRole
    // 发送报文体如下
    /*
      {
         "userId":"1562486017629"
      }
     */
    @RequestMapping(value = "/getUserRole", method = RequestMethod.POST)
    public List<SysUserInfo> getUserRole(@RequestBody HashMap<String, String> map) {
        return sysUserDao.findUserRole(map.get("userId"));
    }

    /**
     * 根据非主键username模糊查询
     */
    // 发送post请求：127.0.0.1:8081/user/getUserByUserName
    // 发送报文体如下
    /*
    {
        "userName":"username"
    }
     */
    @RequestMapping(value = "/getUserByUserName", method = RequestMethod.POST)
    public List<SysUser> getUserByUserName(@RequestBody HashMap<String, String> map) {
        return sysUserDao.findUserName(map.get("userName"));
    }
}
```

代码有点多，只是我写的例子多了点

### 6、补充Dao

```java
public interface SysUserDao extends JpaRepository<SysUser, Integer> {

    /**
     * 根据userId删除数据
     */
    @Transactional
    @Query(value = "delete u from sys_user u where u.user_id = ?1", nativeQuery = true)
    @Modifying
    void deleteByUserId(String userId);

    /**
     * 根据UserId查询
     * @param userId
     * @return
     */
    @Query(value = "select u.* from sys_user u where u.user_id = ?1", nativeQuery = true)
    Optional<SysUser> findByUserId(String userId);

    /**
     * 根据Id更新用户相关信息
     * nativeQuery = true 添加该属性等于true则是原生SQL语句查询，不添加则是HQL语句
     */
    @Transactional
    @Query(value = "update  sys_user set email=?1, user_name=?2, pass_word=?3 where user_id=?4", nativeQuery = true)
    @Modifying
    public void updateOne(String email, String userName, String passWord, String userId);

    /**
     * 查询用户角色
     * @param userId
     * @return
     */
    @Query(value = "SELECT " +
            "t.user_id AS userId, " +
            "t.user_name AS userName, " +
            "t.email AS email, " +
            "t.pass_word AS passWord, " +
            "r.role_id AS roleId " +
            "FROM sys_user t LEFT JOIN sys_user_role r " +
            "ON r.user_id = t.user_id " +
            "WHERE t.user_id = ?1", nativeQuery = true)
    List<SysUserInfo> findUserRole(String userId);

    /**
     * 根据username查询用户信息
     * @return
     */
    @Query(value = "select u.* from sys_user u where u.user_name like CONCAT('%',?1,'%')", nativeQuery = true)
    List<SysUser> findUserName(String nickName);
}
```

这里需要注意的在`findUserRole`方法，是联表查询，其结果集在`SysUserInfo`中

```java
public interface SysUserInfo {

    String getUserId();

    String getUserName();

    String getEmail();

    String getPassWord();

    int getRoleId();
}
```

## 三、测试

启动项目之前，将`spring.jpa.hibernate.ddl-auto`改为`create`。启动完成之后改为update或者none。

会生成两张表sys_user用户表，sys_user_role用户角色对应表

然后通过controller里的一个接口init，发送get请求

生成一些数据。

之后可以进行具体的数据库接口操作啦。

## 四、总结

在学习过程中，敲代码也遇到不少坑，感觉Jpa还行，确实比Mybatis快了不少，不需要建立mapper.xml文件。

可是在项目中不可能都是一些简单的查询SQL呀，肯定会遇到许多复杂的SQL，如果用Jpa的话，感觉并不是那么好用。当然我还没有深入去学习它。肯定有许多我不太明白的技术。肯定可以解决不复杂SQL。

我在网上也搜索了，有些人会建议将Jpa和Mybatis结合使用。我也感觉这点子不错。后续会继续研究



## 五、源码

github源码地址：[Spring Boot2(九)：整合Jpa的基本使用](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-18-jpa02)

To be continued...

