---
layout: post
title: Spring Boot2(十二)：手摸手教你搭建Shiro安全框架
category: springboot
tags: [springboot]
copyright: java
---

SpringBoot+Shiro+Mybatis完成的。

之前看了一位小伙伴的Shiro教程，跟着做了，遇到蛮多坑的(´இ皿இ｀)

修改整理了一下，成功跑起来了。可以通过postman进行测试

不多比比∠( ᐛ 」∠)＿，直接上源码：https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-20-shiro

## 一、Shiro是啥

Apache Shiro是一个功能强大、灵活的、开源的安全框架。可以干净利落地处理身份验证、授权、企业会话管理和加密。

## 二、Shiro可以干啥

- 验证用户身份
- 用户访问权限控制，比如：1、判断用户是否分配了一定的安全角色。2、判断用户是否被授予完成某个操作的权限
- 在非 Web 或 EJB 容器的环境下可以任意使用 Session API
- 可以响应认证、访问控制，或者 Session 生命周期中发生的事件
- 可将一个或以上用户安全数据源数据组合成一个复合的用户 “view”(视图)
- 支持单点登录(SSO)功能
- 支持提供“Remember Me”服务，获取用户关联信息而无需登录

Shiro框架图如下：

![img](http://www.itmind.net/assets/images/2017/springboot/ShiroFeatures.png)

- **Authentication（认证）：**用户身份识别，通常被称为用户“登录”
- **Authorization（授权）：**访问控制。比如某个用户是否具有某个操作的使用权限。
- **Session Management（会话管理）：**特定于用户的会话管理,甚至在非web 或 EJB 应用程序。
- **Cryptography（加密）：**在对数据源使用加密算法加密的同时，保证易于使用。

在概念层，Shiro架构包含三个主要的理念：Subject，SecurityManager和 Realm。下面的图展示了这些组件如何相互作用，我们将在下面依次对其进行描述。

![img](http://www.itmind.net/assets/images/2017/springboot/ShiroBasicArchitecture.png)

- Subject：当前用户，Subject 可以是一个人，但也可以是第三方服务、守护进程帐户、时钟守护任务或者其它–当前和软件交互的任何事件。
- SecurityManager：管理所有Subject，SecurityManager 是 Shiro 架构的核心，配合内部安全组件共同组成安全伞。
- Realms：用于进行权限信息的验证，我们自己实现。Realm 本质上是一个特定的安全 DAO：它封装与数据源连接的细节，得到Shiro 所需的相关的数据。在配置 Shiro 的时候，你必须指定至少一个Realm 来实现认证（authentication）和/或授权（authorization）。

## 三、代码实现

### 1、添加Maven依赖

```
<dependency>
	<groupId>org.apache.shiro</groupId>
	<artifactId>shiro-spring</artifactId>
<version>1.4.1</version>
```

### 2、配置文件

`application.yml`

```yaml
# 服务器端口
server:
  port: 8081

# 配置Spring相关信息
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: root

# 配置Mybatis
mybatis:
  type-aliases-package: com.niaobulashi.model
  mapper-locations: classpath:mapper/*.xml
  configuration:
    # 开启驼峰命名转换
    map-underscore-to-camel-case: true

# 打印SQL日志
logging:
  level:
    com.niaobulashi.mapper: DEBUG
```

启动方法添加mapper扫描，我一般都是在启动方法上面声明，否则需要在每一个mapper上单独声明扫描

```
@SpringBootApplication
@MapperScan("com.niaobulashi.mapper")
public class ShiroApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShiroApplication.class, args);
    }
}
```

### 3、简单的表设计

无非就是5张表：用户表、角色表、权限表、用户角色表、角色权限表。

看下面这张图，可以说相当明了了。

![用户角色权限关系表](https://images.niaobulashi.com/typecho/uploads/2019/07/2956571916.png)

具体我就不贴出来了，太占篇幅。。直接贴链接：https://github.com/niaobulashi/spring-boot-learning/blob/master/spring-boot-20-shiro/db/test.sql

### 4、实体类

#### User.java

```
@Data
public class User implements Serializable {

    private static final long serialVersionUID = -6056125703075132981L;

    private Integer id;

    private String account;

    private String password;

    private String username;
}
```

#### Role.java

```
@Data
public class Role implements Serializable {

    private static final long serialVersionUID = -1767327914553823741L;

    private Integer id;

    private String role;

    private String desc;
}
```

### 5、mapper层

这里概括一下：简单的用户登录权限的Shiro控制涉及到的数据库操作主要有仨

- 用户**登录名**查询**用户**信息
- 根据**用户**查询**角色**信息
- 根据**角色**查询**权限**信息

#### UserMapper.java/UserMapper.xml

```
public interface UserMapper {
    /**
     * 根据账户查询用户信息
     * @param account
     * @return
     */
    User findByAccount(@Param("account") String account);
}
```

```
<!--用户表结果集-->
<sql id="base_column_list">
	id, account, password, username
</sql>

<!--根据账户查询用户信息-->
<select id="findByAccount" parameterType="Map" resultType="com.niaobulashi.model.User">
	select
	<include refid="base_column_list"/>
	from user
	where account = #{account}
</select>
```

#### RoleMapper.java/RoleMapper.xml

```
public interface RoleMapper {
    /**
     * 根据userId查询角色信息
     * @param userId
     * @return
     */
    List<Role> findRoleByUserId(@Param("userId") Integer userId);
}
```

```
<!--角色表字段结果集-->
<sql id="base_cloum_list">
	id, role, desc
</sql>

<!--根据userId查询角色信息-->
<select id="findRoleByUserId" parameterType="Integer" resultType="com.niaobulashi.model.Role">
	select r.id, r.role
	from role r
	left join user_role ur on ur.role_id = r.id
	left join user u on u.id = ur.user_id
	where 1=1
	and u.user_id = #{userId}
</select>
```

#### PermissionMapper.java/PermissionMapper.xml

```
public interface PermissionMapper {
    /**
     * 根据角色id查询权限
     * @param roleIds
     * @return
     */
    List<String> findByRoleId(@Param("roleIds") List<Integer> roleIds);
}
```

```
<!--权限查询结果集-->
<sql id="base_column_list">
	id, permission, desc
</sql>

<!--根据角色id查询权限-->
<select id="findByRoleId" parameterType="List" resultType="String">
	select permission
	from permission, role_permission rp
	where rp.permission_id = permission.id and rp.role_id in
	<foreach collection="roleIds" item="id" open="(" close=")" separator=",">
		#{id}
	</foreach>
</select>
```

### 6、Service层

没有其他逻辑，只有继承。

**注意：**

> 不过需要注意的一点是，我在Service层中，使用的注解@Service：启动时会自动注册到Spring容器中。
>
> 否则启动时，拦截器配置初始化时，会找不到Service。。。这点有点坑。

#### UserService.java/UserServiceImpl.java

```
public interface UserService {
    /**
     * 根据账户查询用户信息
     * @param account
     * @return
     */
    User findByAccount(String account);
}
```

```
@Service("userService")
public class UserServiceImpl implements UserService {

    @Resource
    private UserMapper userMapper;

    /**
     * 根据账户查询用户信息
     * @param account
     * @return
     */
    @Override
    public User findByAccount(String account) {
        return userMapper.findByAccount(account);
    }
}
```

#### RoleService.java/RoleServiceImpl.java

```
public interface RoleService {

    /**
     * 根据userId查询角色信息
     * @param id
     * @return
     */
    List<Role> findRoleByUserId(Integer id);
}
```

```
@Service("roleService")
public class RoleServiceImpl implements RoleService {

    @Resource
    private RoleMapper roleMapper;

    /**
     * 根据userId查询角色信息
     * @param id
     * @return
     */
    @Override
    public List<Role> findRoleByUserId(Integer id) {
        return roleMapper.findRoleByUserId(id);
    }
}
```

#### PermissionService.java/PermissionServiceImpl.java

```
public interface PermissionService {

    /**
     * 根据角色id查询权限
     * @param roleIds
     * @return
     */
    List<String> findByRoleId(@Param("roleIds") List<Integer> roleIds);
}
```

```
@Service("permissionService")
public class PermissionServiceImpl implements PermissionService {

    @Resource
    private PermissionMapper permissionMapper;

    /**
     * 根据角色id查询权限
     * @param roleIds
     * @return
     */
    @Override
    public List<String> findByRoleId(List<Integer> roleIds) {
        return permissionMapper.findByRoleId(roleIds);
    }
}
```

### 7、系统统一返回状态枚举和包装方法

状态字段枚举

#### StatusEnmus.java

```
public enum StatusEnums {

    SUCCESS(200, "操作成功"),
    SYSTEM_ERROR(500, "系统错误"),
    ACCOUNT_UNKNOWN(500, "账户不存在"),
    ACCOUNT_IS_DISABLED(13, "账号被禁用"),
    INCORRECT_CREDENTIALS(500,"用户名或密码错误"),
    PARAM_ERROR(400, "参数错误"),
    PARAM_REPEAT(400, "参数已存在"),
    PERMISSION_ERROR(403, "没有操作权限"),
    NOT_LOGIN_IN(15, "账号未登录"),
    OTHER(-100, "其他错误");

    @Getter
    @Setter
    private int code;
    @Getter
    @Setter
    private String message;

    StatusEnums(int code, String message) {
        this.code = code;
        this.message = message;
    }
}
```

响应包装方法

#### ResponseCode.java

```
@Data
@AllArgsConstructor
public class ResponseCode<T> implements Serializable {

    private Integer code;
    private String message;
    private Object data;

    private ResponseCode(StatusEnums responseCode) {
        this.code = responseCode.getCode();
        this.message = responseCode.getMessage();
    }

    private ResponseCode(StatusEnums responseCode, T data) {
        this.code = responseCode.getCode();
        this.message = responseCode.getMessage();
        this.data = data;
    }

    private ResponseCode(Integer code, String message) {
        this.code = code;
        this.message = message;
    }

    /**
     * 返回成功信息
     * @param data      信息内容
     * @param <T>
     * @return
     */
    public static<T> ResponseCode success(T data) {
        return new ResponseCode<>(StatusEnums.SUCCESS, data);
    }

    /**
     * 返回成功信息
     * @return
     */
    public static ResponseCode success() {
        return new ResponseCode(StatusEnums.SUCCESS);
    }

    /**
     * 返回错误信息
     * @param statusEnums      响应码
     * @return
     */
    public static ResponseCode error(StatusEnums statusEnums) {
        return new ResponseCode(statusEnums);
    }
}
```

### 8、Shiro配置

#### ShiroConfig.java

```
@Configuration
public class ShiroConfig {

	/**
	 * 路径过滤规则
	 * @return
	 */
	@Bean
	public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
		ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
		shiroFilterFactoryBean.setSecurityManager(securityManager);
		// 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
		shiroFilterFactoryBean.setLoginUrl("/login");
		shiroFilterFactoryBean.setSuccessUrl("/");
		// 拦截器
		Map<String, String> map = new LinkedHashMap<>();
		// 配置不会被拦截的链接 顺序判断
		map.put("/login", "anon");
		// 过滤链定义，从上向下顺序执行，一般将/**放在最为下边
		// 进行身份认证后才能访问
		// authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问
		map.put("/**", "authc");
		shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
		return shiroFilterFactoryBean;
	}

	/**
	 * 自定义身份认证Realm（包含用户名密码校验，权限校验等）
	 * @return
	 */
	@Bean
	public AuthRealm authRealm() {
		return new AuthRealm();
	}

	@Bean
	public SecurityManager securityManager() {
		DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
		securityManager.setRealm(authRealm());
		return securityManager;
	}

	/**
	 * 开启Shiro注解模式，可以在Controller中的方法上添加注解
	 * @param securityManager
	 * @return
	 */
	@Bean
	public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager){
		AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
		authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
		return authorizationAttributeSourceAdvisor;
	}
}
```

#### 扩展：权限拦截Filter的URL的一些说明

这里扩展一下**权限拦截Filter的URL的一些说明**

1、URL匹配规则

（1）“?”：匹配一个字符，如”/admin?”，将匹配“ /admin1”、“/admin2”，但不匹配“/admin”

（2）“*”：匹配零个或多个字符串，如“/admin*”，将匹配“ /admin”、“/admin123”，但不匹配“/admin/1”

（3）“**”：匹配路径中的零个或多个路径，如“/admin/**”，将匹配“/admin/a”、“/admin/a/b”

2、shiro过滤器

| Filter       | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| anon         | 无参，开放权限，可以理解为匿名用户或游客                     |
| authc        | 无参，需要认证                                               |
| logout       | 无参，注销，执行后会直接跳转到`shiroFilterFactoryBean.setLoginUrl();` 设置的 url |
| authcBasic   | 无参，表示 httpBasic 认证                                    |
| user         | 无参，表示必须存在用户，当登入操作时不做检查                 |
| ssl          | 无参，表示安全的URL请求，协议为 https                        |
| perms[user]  | 参数可写多个，表示需要某个或某些权限才能通过，多个参数时写 perms["user, admin"]，当有多个参数时必须每个参数都通过才算通过 |
| roles[admin] | 参数可写多个，表示是某个或某些角色才能通过，多个参数时写 roles["admin，user"]，当有多个参数时必须每个参数都通过才算通过 |
| rest[user]   | 根据请求的方法，相当于 perms[user:method]，其中 method 为 post，get，delete 等 |
| port[8081]   | 当请求的URL端口不是8081时，跳转到[schemal://serverName:8081?queryString](https://link.jianshu.com?t=schemal%3A%2F%2FserverName%3A8081%3FqueryString) 其中 schmal 是协议 http 或 https 等等，serverName 是你访问的 Host，8081 是 Port 端口，queryString 是你访问的 URL 里的 ? 后面的参数 |

常用的主要就是 anon，authc，user，roles，perms 等

**注意**：anon, authc, authcBasic, user 是第一组认证过滤器，perms, port, rest, roles, ssl 是第二组授权过滤器，要通过授权过滤器，就先要完成登陆认证操作（即先要完成认证才能前去寻找授权) 才能走第二组授权器（例如访问需要 roles 权限的 url，如果还没有登陆的话，会直接跳转到 `shiroFilterFactoryBean.setLoginUrl();` 设置的 url ）。

### 9、自定义Realm

主要继承`AuthorizingRealm`，重写里面的方法`doGetAuthorizationInfo`，`doGetAuthenticationInfo`

授权：`doGetAuthorizationInfo`

认证：`doGetAuthenticationInfo`

#### AuthRealm.java

```
public class AuthRealm extends AuthorizingRealm {

    @Resource
    private UserService userService;

    @Resource
    private RoleService roleService;

    @Resource
    private PermissionService permissionService;

    /**
     * 授权
     * @param principalCollection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        User user = (User) principalCollection.getPrimaryPrincipal();
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        // 根据用户Id查询角色信息
        List<Role> roleList = roleService.findRoleByUserId(user.getId());
        Set<String> roleSet = new HashSet<>();
        List<Integer> roleIds = new ArrayList<>();
        for (Role role : roleList) {
            roleSet.add(role.getRole());
            roleIds.add(role.getId());
        }
        // 放入角色信息
        authorizationInfo.setRoles(roleSet);
        // 放入权限信息
        List<String> permissionList = permissionService.findByRoleId(roleIds);
        authorizationInfo.setStringPermissions(new HashSet<>(permissionList));

        return authorizationInfo;
    }

    /**
     * 认证
     * @param authToken
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authToken) throws AuthenticationException {
        UsernamePasswordToken token = (UsernamePasswordToken) authToken;
        // 根据用户名查询用户信息
        User user = userService.findByAccount(token.getUsername());
        if (user == null) {
            return null;
        }
        return new SimpleAuthenticationInfo(user, user.getPassword(), getName());
    }
}
```

### 10、Contrller层

```
@RestController
public class LoginController {

    /**
     * 登录操作
     * @param user
     * @return
     */
    @RequestMapping(value = "/login", method = RequestMethod.POST)
    public ResponseCode login(@RequestBody User user) {
        Subject userSubject = SecurityUtils.getSubject();
        UsernamePasswordToken token = new UsernamePasswordToken(user.getAccount(), user.getPassword());
        try {
            // 登录验证
            userSubject.login(token);
            return ResponseCode.success();
        } catch (UnknownAccountException e) {
            return ResponseCode.error(StatusEnums.ACCOUNT_UNKNOWN);
        } catch (DisabledAccountException e) {
            return ResponseCode.error(StatusEnums.ACCOUNT_IS_DISABLED);
        } catch (IncorrectCredentialsException e) {
            return ResponseCode.error(StatusEnums.INCORRECT_CREDENTIALS);
        } catch (Throwable e) {
            e.printStackTrace();
            return ResponseCode.error(StatusEnums.SYSTEM_ERROR);
        }
    }


    @GetMapping("/login")
    public ResponseCode login() {
        return ResponseCode.error(StatusEnums.NOT_LOGIN_IN);
    }

    @GetMapping("/auth")
    public String auth() {
        return "已成功登录";
    }

    @GetMapping("/role")
    @RequiresRoles("vip")
    public String role() {
        return "测试Vip角色";
    }

    @GetMapping("/permission")
    @RequiresPermissions(value = {"add", "update"}, logical = Logical.AND)
    public String permission() {
        return "测试Add和Update权限";
    }

    /**
     * 登出
     * @return
     */
    @GetMapping("/logout")
    public ResponseCode logout() {
        getSubject().logout();
        return ResponseCode.success();
    }
}
```

## 四、测试

1、登录：http://localhost:8081/login

```
{
	"account":"123",
	"password":"232"
}
```

2、其他的是get请求，直接发URL就行啦。

已通过接口测试，大家可放心食用。



参考：https://juejin.im/post/5d27db16e51d454fbe24a717

推荐阅读：

张开涛老的《跟我学Shiro》https://jinnianshilongnian.iteye.com/blog/2018936



---

To be continued

> 作者：**鸟不拉诗**   出处：[ https://juejin.im/user/5b3de9155188251aa0161fe4](https://juejin.im/user/5b3de9155188251aa0161fe4)
>
> 本文版权归作者和掘金共有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。如果觉得还有帮助的话，可以点一下左上角的【点赞】。

