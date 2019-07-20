---
layout: post
title: Spring Boot2(十一)：Mybatis使用总结（自增长、多条件、批量操作、多表查询等等）
category: springboot
tags: [springboot]
copyright: java
---

上次用Mybatis还是2017年做项目的时候，已经很久过去了。中途再没有用过Mybatis。导致现在学习SpringBoot过程中遇到一些Mybatis的问题，以此做出总结（XML极简模式）。当然只是实用方面的总结，具体就不深究♂了。**这里只总结怎么用！！！**

## 一、关于Mybatis

### 1、什么是Mybatis

（1）Mybatis是一个半ORM（对象关系映射）框架，它内部封装了JDBC，开发时只需要关注SQL语句本身，不需要花费精力去处理加载驱动、创建连接、创建statement等繁杂的过程。程序员直接编写原生态sql，可以严格控制sql执行性能，灵活度高。

（2）MyBatis 可以使用 XML 或注解来配置和映射原生信息，将 POJO映射成数据库中的记录，避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

（3）通过xml 文件或注解的方式将要执行的各种 statement 配置起来，并通过java对象和 statement中sql的动态参数进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射为java对象并返回。（从执行sql到返回result的过程）。

### 2、Mybaits的优点

（1）基于SQL语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL写在XML里，解除sql与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态SQL语句，并可重用。

（2）与JDBC相比，减少了50%以上的代码量，消除了JDBC大量冗余的代码，不需要手动开关连接；

（3）很好的与各种数据库兼容（因为MyBatis使用JDBC来连接数据库，所以只要JDBC支持的数据库MyBatis都支持）。

（4）能够与Spring很好的集成；

（5）提供映射标签，支持对象与数据库的ORM字段关系映射；提供对象关系映射标签，支持对象关系组件维护。

### 3、MyBatis框架的缺点

（1）SQL语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写SQL语句的功底有一定要求。

（2）SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

### 4、MyBatis框架适用场合

（1）MyBatis专注于SQL本身，是一个足够灵活的DAO层解决方案。

（2）对性能的要求很高，或者需求变化较多的项目，如互联网项目，MyBatis将是不错的选择。

### 5、MyBatis与Hibernate有哪些不同

（1）Mybatis和hibernate不同，它不完全是一个ORM框架，因为MyBatis需要程序员自己编写Sql语句。

（2）Mybatis直接编写原生态sql，可以严格控制sql执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。但是灵活的前提是mybatis无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套sql映射文件，工作量大。 

（3）Hibernate对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用hibernate开发可以节省很多代码，提高效率。 

---



## 二、使用总结

> 以下的用法实例建议将源码clone到本地运行，全部使用的是**XMl极简模式**
>
> 因为我没有贴出完整的代码，只贴出关键处理的部分
>
> 所有测试都已经通过Postman发送请求测试。
>
> 不过我建议各位看官可以用下IDEA的插件：**Restfultookit**，非常好用的，根据controller定义的url地址快捷生成请求报文，可以直接测试。对于测试报文来说这个插件简直无敌！强烈推荐（已经安装的当我没说）

---

### 1、Java，JDBC与MySQL数据类型对照数据类型关系表

任何MySQL数据类型都可以转换为Java数据类型。

如果选择的Java数值数据类型的精度或容量低于要转换为的MySQL数据类型，则可能会出现舍入，溢出或精度损失。

下表列出了始终保证有效的转换。 第一列列出了一种或多种MySQL数据类型，第二列列出了可以转换MySQL类型的一种或多种Java类型。

| These MySQL Data Types                                       | Can always be converted to these Java types                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `CHAR, VARCHAR, BLOB, TEXT, ENUM, and SET`                   | `java.lang.String, java.io.InputStream, java.io.Reader, java.sql.Blob, java.sql.Clob` |
| `FLOAT, REAL, DOUBLE PRECISION, NUMERIC, DECIMAL, TINYINT, SMALLINT, MEDIUMINT, INTEGER, BIGINT` | `java.lang.String, java.lang.Short, java.lang.Integer, java.lang.Long, java.lang.Double, java.math.BigDecimal` |
| `DATE, TIME, DATETIME, TIMESTAMP`                            | `java.lang.String, java.sql.Date, java.sql.Timestamp`        |

ResultSet.getObject（）方法使用MySQL和Java类型之间的类型转换，遵循适当的JDBC规范。 ResultSetMetaData.GetColumnTypeName（）和ResultSetMetaData.GetColumnClassName（）返回的值如下表所示。 有关JDBC类型的更多信息，请参阅java.sql.Types类的参考。

| MySQL Type Name               | Return value of `GetColumnTypeName` | Return value of `GetColumnClassName`                         |
| ----------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| `BIT(1)`                      | `BIT`                               | `java.lang.Boolean`                                          |
| `BIT( > 1)`                   | `BIT`                               | `byte[]`                                                     |
| `TINYINT`                     | `TINYINT`                           | `java.lang.Boolean` if the configuration property `tinyInt1isBit` is set to `true` (the default) and the storage size is 1, or `java.lang.Integer` if not. |
| `BOOL`, `BOOLEAN`             | `TINYINT`                           | See `TINYINT`, above as these are aliases for `TINYINT(1)`, currently. |
| `SMALLINT[(M)] [UNSIGNED]`    | `SMALLINT [UNSIGNED]`               | `java.lang.Integer` (regardless of whether it is `UNSIGNED` or not) |
| `MEDIUMINT[(M)] [UNSIGNED]`   | `MEDIUMINT [UNSIGNED]`              | `java.lang.Integer` (regardless of whether it is `UNSIGNED` or not) |
| `INT,INTEGER[(M)] [UNSIGNED]` | `INTEGER [UNSIGNED]`                | `java.lang.Integer`, if `UNSIGNED` `java.lang.Long`          |
| `BIGINT[(M)] [UNSIGNED]`      | `BIGINT [UNSIGNED]`                 | `java.lang.Long`, if UNSIGNED `java.math.BigInteger`         |
| `FLOAT[(M,D)]`                | `FLOAT`                             | `java.lang.Float`                                            |
| `DOUBLE[(M,B)]`               | `DOUBLE`                            | `java.lang.Double`                                           |
| `DECIMAL[(M[,D])]`            | `DECIMAL`                           | `java.math.BigDecimal`                                       |
| `DATE`                        | `DATE`                              | `java.sql.Date`                                              |
| `DATETIME`                    | `DATETIME`                          | `java.sql.Timestamp`                                         |
| `TIMESTAMP[(M)]`              | `TIMESTAMP`                         | `java.sql.Timestamp`                                         |
| `TIME`                        | `TIME`                              | `java.sql.Time`                                              |
| `YEAR[(2|4)]`                 | `YEAR`                              | If `yearIsDateType` configuration property is set to `false`, then the returned object type is `java.sql.Short`. If set to `true` (the default), then the returned object is of type `java.sql.Date` with the date set to January 1st, at midnight. |
| `CHAR(M)`                     | `CHAR`                              | `java.lang.String` (unless the character set for the column is `BINARY`, then `byte[]` is returned. |
| `VARCHAR(M) [BINARY]`         | `VARCHAR`                           | `java.lang.String` (unless the character set for the column is `BINARY`, then `byte[]` is returned. |
| `BINARY(M)`                   | `BINARY`                            | `byte[]`                                                     |
| `VARBINARY(M)`                | `VARBINARY`                         | `byte[]`                                                     |
| `TINYBLOB`                    | `TINYBLOB`                          | `byte[]`                                                     |
| `TINYTEXT`                    | `VARCHAR`                           | `java.lang.String`                                           |
| `BLOB`                        | `BLOB`                              | `byte[]`                                                     |
| `TEXT`                        | `VARCHAR`                           | `java.lang.String`                                           |
| `MEDIUMBLOB`                  | `MEDIUMBLOB`                        | `byte[]`                                                     |
| `MEDIUMTEXT`                  | `VARCHAR`                           | `java.lang.String`                                           |
| `LONGBLOB`                    | `LONGBLOB`                          | `byte[]`                                                     |
| `LONGTEXT`                    | `VARCHAR`                           | `java.lang.String`                                           |
| `ENUM('value1','value2',...)` | `CHAR`                              | `java.lang.String`                                           |
| `SET('value1','value2',...)`  | `CHAR`                              | `java.lang.String`                                           |

参考：[6.5 Java, JDBC, and MySQL Types](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-type-conversions.html)

### 2、当实体类中的属性名和表中的字段名不一样，怎么办

> 其一：定义字段别名，使之与实体类属性名一致。

```xml
<!-- 查询用户信息列表1 -->
<select id="queryUserList1" resultType="com.niaobulashi.entity.SysUser">
   SELECT
		u.user_id, u.username userNameStr, u.password, u.salt, u.email,
		u.mobile, u.status, u.dept_id, u.create_time
	FROM
		sys_user u
	where 1=1
</select>
```

> 其二：通过resultMap映射字段名和实体类属性名保持一致

```xml
<resultMap id="sysUserInfoMap" type="com.niaobulashi.entity.SysUser">
	<!-- 用户Id属性来映射主键字段 userId-->
	<id property="id" column="userId"/>
	<!-- 用result属性来映射非主键字段，property为实体类属性名，column为数据表中的属性-->
	<result property="userNameStr" column="username"/>
</resultMap>

<!--用户Vo-->
<sql id="selectSysUserVo">
	SELECT
		u.user_id, u.username, u.password, u.salt, 
		u.email, u.mobile, u.status, u.dept_id, u.create_time
	FROM
		sys_user u
</sql>

<!-- 查询用户信息列表2 -->
<select id="queryUserList2" resultMap="sysUserInfoMap">
    <include refid="selectSysUserVo"/>
    where 1=1
</select>
```

推荐使用第二种。

### 2、获取Mybatis自增长主键

思路：`useGeneratedKeys="true" keyProperty="id"`

```xml
<!-- 获取自动生成的(主)键值 -->
<insert id="insertSysTest" parameterType="com.niaobulashi.model.SysTest"
		useGeneratedKeys="true" keyProperty="id">
	INSERT INTO sys_test(name, age, nick_name) VALUES (#{name},#{age},#{nickName})
</insert>
```

获取自增长主键

```java
/**
 * 获取自增长主键ID
 * @param sysTest
 * @throws Exception
 */
@RequestMapping(value = "/add", method = RequestMethod.POST)
private void addSysTest(@RequestBody SysTest sysTest) throws Exception {
	try {
		SysTest sysTestParam = new SysTest();
        // 将传入参数Copy到新申明的对象中，这样才能从sysTestParam中获取到自增长主键
		BeanUtils.copyProperties(sysTest, sysTestParam);
		this.sysTestService.insertSysTest(sysTestParam);
		log.info("获取自增长主键为：" + sysTestParam.getId());
	} catch (Exception e) {
		e.printStackTrace();
		throw new Exception();
	}
}
```

### 3、模糊查询



使用`%"#{value}"%"`方法会引起SQL注入

推荐使用：**`CONCAT('%',#{value},'%')`**

```java
<!--用户Vo-->
<sql id="selectSysUserVo">
	SELECT
		u.user_id, u.username, u.password, u.salt, 
		u.email, u.mobile, u.status, u.dept_id, u.create_time
	FROM
		sys_user u
</sql>

<!-- 查询用户信息列表2 -->
<select id="queryUserListByName" parameterType="String" resultMap="sysUserInfoMap">
    <include refid="selectSysUserVo"/>
	where 1=1
	and u.username like concat('%',#{userName},'%')
</select>
```

### 4、多条件查询

1、使用@Param

```java
List<SysUser> queryUserByNameAndEmail(@Param("userName") String userName, @Param("email") String email);
```

```xml
<!--使用用户名和邮箱查询用户信息-->
<select id="queryUserByNameAndEmail" resultMap="sysUserInfoMap">
	<include refid="selectSysUserVo"/>
	<where>
        <if test="userName != null and userName != ''">
            AND u.username like concat('%',#{userName},'%')
        </if>
        <if test="email != null and email != ''">
            AND u.email like concat('%',#{email},'%')
        </if>
	</where>
</select>
```

2、使用JavaBean

这里给了一些常见的查询条件：日期、金额。

```
List<SysUser> queryUserByUser(SysUser sysUser);
```

```xml
<select id="queryUserByUser" parameterType="com.niaobulashi.model.SysUser" resultMap="sysUserInfoMap">
	<include refid="selectSysUserVo"/>
	<where>
		1=1
		<if test="userNameStr != null and userNameStr != ''">
			AND u.username like concat('%', #{userNameStr}, '%')
		</if>
		<if test="email != null and email != ''">
			AND u.email like concat('%', #{email}, '%')
		</if>
		<if test="mobile != null and mobile != ''">
			AND u.mobile like concat('%', #{mobile}, '%')
		</if>
		<if test="createDateStart != null and createDateStart != ''">/*开始时间检索*/
			AND date_format(u.create_time, '%y%m%d') <![CDATA[ >= ]]> date_format(#{createDateStart}, '%y%m%d')
		</if>
		<if test="createDateEnd != null and createDateEnd != ''">/*结束时间检索*/
			AND date_format(u.create_time, '%y%m%d') <![CDATA[ <= ]]> date_format(#{createDateEnd}, '%y%m%d')
		</if>
		<if test="amtFrom != null and amtFrom != ''">/*起始金额*/
			AND u.amt <![CDATA[ >= ]]> #{amtFrom}
		</if>
		<if test="amtTo != null and amtTo != ''">/*截至金额*/
			AND u.amt <![CDATA[ <= ]]> #{amtTo}
		</if>
	</where>
</select>
```

### 5、批量删除foreach

xml部分

```xml
<delete id="deleteSysTestByIds" parameterType="String">
	delete from sys_test where id in
	<foreach collection="array" item="id" open="(" separator="," close=")">
		#{id}
	</foreach>
</delete>
```

其中foreach包含属性讲解：

- open：整个循环内容开头的字符串。
- close：整个循环内容结尾的字符串。
- separator：每次循环的分隔符。
- item：从迭代对象中取出的每一个值。
- index：如果参数为集合或者数组，该值为当前索引值，如果参数为Map类型时，该值为Map的key。
- collection：要迭代循环的属性名。

dao部分

```java
int deleteSysTestByIds(String[] ids);
```

service层

```java
@Transactional(rollbackFor = Exception.class)
@Override
public int deleteDictDataByIds(String ids) throws Exception{
	try {
		return sysTestDao.deleteSysTestByIds(ids.split(","));
	} catch (Exception e) {
		e.printStackTrace();
		throw new Exception();
	}
}
```

controller

```java
@RequestMapping(value = "/deleteIds", method = RequestMethod.POST)
public int deleteIds(String ids) throws Exception {
	try {
		return sysTestService.deleteDictDataByIds(ids);
	} catch (Exception e) {
		e.printStackTrace();
		throw new Exception();
	}
}
```

请求URL：http://localhost:8081/test/deleteIds

请求报文：

```
ids : 1,2
```

### 6、多表查询association和collection

多表查询，多表肯定首先我们先要弄清楚两个关键字：

**association: 一对一关联(has one)**；**collection:一对多关联(has many)**

的各个属性的含义：

| association和collection                                      |
| ------------------------------------------------------------ |
| property：映射数据库列的字段或属性。<br/>colum：数据库的列名或者列标签别名。<br/>javaTyp：完整java类名或别名。<br/>jdbcType：支持的JDBC类型列表列出的JDBC类型。这个属性只在insert,update或delete的时候针对允许空的列有用。<br/>resultMap：一个可以映射联合嵌套结果集到一个适合的对象视图上的ResultMap。这是一个替代的方式去调用另一个select语句。 |

这样说起来可能不好理解，我举个栗子

涉及到这三张表，我粗略的画了一下：

| -            | 用户表   | 部门表                           | 角色表                           |
| ------------ | -------- | -------------------------------- | -------------------------------- |
| 表名         | sys_user | sys_dept                         | sys_role                         |
| 与用户表关系 | -        | 一对一（一个用户只属于一个部门） | 一对多（一个用户可以有多个角色） |

于是用户表关联部门表，我们用**association**

用户表关联角色表，我们用**collection**

当然了，能用得这么蛋疼关键字的前提条件是，你要查询关联的字段，如果你只是关联不查它，那就不需要用这玩意。。

辣么，我结合这两个多表查询的关键字**association**、**collection**举个栗子。

#### 1、用户表实体类

```java
@Data
public class SysUser implements Serializable {
	private static final long serialVersionUID = 1L;
	/** 用户ID */
	private Long userId;
	/** 用户名 */
	private String userNameStr;
	/** 密码 */
	private String password;
	/** 盐 */
	private String salt;
	/** 邮箱 */
	private String email;
	/** 手机号 */
	private String mobile;
	/** 状态  0：禁用   1：正常 */
	private Integer status;
	/** 部门Id */
	private Long deptId;
	/** 创建时间 */
	private Date createTime;
	/****************关联部分**************
	/** 部门 */
	private SysDept dept;
	/** 角色集合 */
	private List<SysRole> roles;
}
```

#### 2、部门表实体类

```java
@Data
public class SysDept implements Serializable {
    /** 部门ID */
    private Long deptId;
    /** 部门名称 */
    private String deptName;
}
```

#### 3、角色表实体类

```java
@Data
public class SysRole implements Serializable {
    /** 角色ID */
    private Long roleId;
    /** 角色名称 */
    private String roleName;
}
```

#### 4、Mapper、Service部分（略）

```java
List<SysUser> queryUserRoleDept(SysUser user);
```

#### 5、XML部分

```xml
<!--查看用户部门和角色信息-->
<select id="queryUserRoleDept" parameterType="com.niaobulashi.model.SysUser" resultMap="UserResult">
	select u.user_id, u.username, u.dept_id, d.dept_name, r.role_id, r.role_name
	from sys_user u
	LEFT JOIN sys_dept d on d.dept_id = u.dept_id
	LEFT JOIN sys_user_role ur on ur.user_id = u.user_id
	LEFT JOIN sys_role r on r.role_id = ur.role_id
	WHERE 1=1
	<if test="userId != null and userId != ''">
		AND u.user_id = #{userId}
	</if>
</select>
```

UserResult部分

```xml
<!--用户表-->
<resultMap type="com.niaobulashi.model.SysUser" id="UserResult">
	<id property="userId" column="user_id"/>
	<result property="userNameStr" column="username"/>
	<result property="password" column="login_name"/>
	<result property="salt" column="password"/>
	<result property="email" column="email"/>
	<result property="mobile" column="mobile"/>
	<result property="status" column="status"/>
	<result property="deptId" column="dept_id"/>
	<result property="createTime" column="create_time"/>
	<association property="dept" column="dept_id" javaType="com.niaobulashi.model.SysDept" resultMap="DeptResult"/>
	<collection property="roles" javaType="java.util.List" resultMap="RoleResult"/>
</resultMap>

<!--部门表-->
<resultMap id="DeptResult" type="com.niaobulashi.model.SysDept">
	<id property="deptId" column="dept_id"/>
	<result property="deptName" column="dept_name"/>
</resultMap>

<!--角色表-->
<resultMap id="RoleResult" type="com.niaobulashi.model.SysRole">
	<id property="roleId" column="role_id"/>
	<result property="roleName" column="role_name"/>
</resultMap>
```

#### 6、Controller部分

```java
@RequestMapping(value = "/queryUserRoleDept", method = RequestMethod.POST)
private List<SysUser> queryUserRoleDept(@RequestBody SysUser sysUser) {
	List<SysUser> userList = sysUserService.queryUserRoleDept(sysUser);
	return userList;
}
```

#### 7、测试部分

请求结果：

![多表关联查询测试报文](https://images.niaobulashi.com/typecho/uploads/2019/07/3893102344.png)



### 7、分页插件

使用分页插件`PageHelper Spring Boot Starter`，引入maven依赖：[PageHelper Spring Boot Starter1.2.12](https://mvnrepository.com/artifact/com.github.pagehelper/pagehelper-spring-boot-starter)

application.yml配置

```yaml
# PageHelper分页插件
pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

controller

```java
@RequestMapping(value = "/queryUserByPage", method = RequestMethod.GET)
private PageInfo queryUserByPage(Integer currentPage, Integer pageSize) {
	PageHelper.startPage(currentPage, pageSize);
	List<SysUser> userList = sysUserService.queryUserRoleDept(new SysUser());
	PageInfo info=new PageInfo(userList);
	return info;
}
```

目前暂时写到这里，本篇会持续补充

To be continued
