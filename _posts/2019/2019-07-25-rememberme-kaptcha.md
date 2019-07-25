---
layout: post
title: Spring Boot2(十五)：Shiro记住我rememberMe、验证码Kaptcha
category: springboot
tags: [springboot]
copyright: java
---

接着上次学习的《Spring Boot2(十二)：手摸手教你搭建Shiro安全框架》，实现了Shiro的认证和授权。今天继续在这个基础上学习Shiro实现功能记住我rememberMe，以及登录时验证码Kaptcha。

Remember Me记住我：用户的登录状态会不会因为浏览器的关闭而失效，直到Cookie失效。关闭浏览器后，再次访问登录后的页面可以不用登录。因为用Cookie实现，故只在同一浏览器中有效。

Kaptcha验证码：是谷歌开源的验证码插件，实现登录的验证码验证拦截。

## 一、记住我rememberMe

用户的登录状态会不会因为浏览器的关闭而失效，直到Cookie失效。关闭浏览器后，再次访问登录后的页面可以不用登录。因为用Cookie实现，故只在同一浏览器中有效。

### 修改ShiroConfig

```java
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
	shiroFilterFactoryBean.setSuccessUrl("/index");
	// 拦截器
	LinkedHashMap<String, String> map = new LinkedHashMap<>();
	// 配置不会被拦截的链接 顺序判断
	// 对静态资源设置匿名访问
	map.put("/static/**", "anon");
	map.put("/css/**", "anon");
	map.put("/js/**", "anon");

	// 过滤链定义，从上向下顺序执行，一般将/**放在最为下边
	// 进行身份认证后才能访问
	// authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问
	// user指的是用户认证通过或者配置了Remember Me记住用户登录状态后可访问
	map.put("/**", "user");
	shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
	return shiroFilterFactoryBean;
}
```

因为对登录页面做了一些样式，新增了静态资源文件static，这时候遇到了坑，页面引用的`js`和`css`都无效了，然后发现时因为被拦截了，我们需要在Shiro的拦截器中允许对静态资源的匿名`anon`访问。

注意到将`ShiroFilterFactoryBean`的`map.put("/**", "authc");`更改为`map.put("/**", "user");`user是指用户认证通过或配置了RememberMe记住用户登录状态后可访问。

解决过程查阅了一些资料，不光光只对`css`和`js`的放开，还需要对`static`也放开

对静态资源的拦截相关问题可以参照这里了解学习一下：[Spring Boot Shiro无法访问JS/CSS/IMG+自定义Filter无法访问完美方案](https://412887952-qq-com.iteye.com/blog/2392741)

回来继续，调用SimpleCookie，配置Cookie的基本属性：名称和过期时间。

```sql
/**
 * cookie对象
 * @return
 */
public SimpleCookie rememberMeCookie() {
	// 设置cookie名称，对应login.html页面的<input type="checkbox" name="rememberMe"/>
	SimpleCookie cookie = new SimpleCookie("rememberMe");
	// 设置cookie的过期时间，单位为秒，这里为一天
	cookie.setMaxAge(86400);
	return cookie;
}
```

SimleCookie参数中的名称为页面的name标签属性名称。

实现了Cookie对象属性配置，我们还需要通过`CookieRememberMeManager`进行管理起来。

```java
/**
 * cookie管理对象
 * rememberMeManager()方法是生成rememberMe管理器，而且要将这个rememberMe管理器设置到securityManager中
 * @return
 */
public CookieRememberMeManager rememberMeManager() {
	CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
	cookieRememberMeManager.setCookie(rememberMeCookie());
	// rememberMe cookie加密的密钥 建议每个项目都不一样 默认AES算法 密钥长度(128 256 512 位)
	cookieRememberMeManager.setCipherKey(Base64.decode("3AvVhmFLUs0KTA3Kprsdag=="));
	return cookieRememberMeManager;
}
```

接下来将cookie管理对象设置到`SecurityManager`中：

```java
@Bean
public SecurityManager securityManager() {
	DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
	// 设置realm
	securityManager.setRealm(authRealm());
	// 用户授权/认证信息Cache, 采用EhC//注入记住我管理器
	securityManager.setRememberMeManager(rememberMeManager());
	return securityManager;
}
```

### 加密

《Spring Boot2(十二)：手摸手教你搭建Shiro安全框架》这个项目中用的明文，这里我们升个级，使用MD5加密

新建MD5加密工具类。

```java
public class MD5Utils {

    private static final String SALT = "niaobulashi";

    private static final String ALGORITH_NAME = "md5";

    private static final int HASH_ITERATIONS = 2;

    public static String encrypt(String pwd) {
        String newPassword = new SimpleHash(ALGORITH_NAME, pwd, ByteSource.Util.bytes(SALT), HASH_ITERATIONS).toHex();
        return newPassword;
    }

    public static String encrypt(String username, String pwd) {
        String newPassword = new SimpleHash(ALGORITH_NAME, pwd, ByteSource.Util.bytes(username + SALT),
                HASH_ITERATIONS).toHex();
        return newPassword;
    }
    
    public static void main(String[] args) {
        System.out.println("MD5加密后的密文为：" + MD5Utils.encrypt("root", "root"));
    }
}
```

其中`SALT`是加密的盐，main方法中，根据登录名和密码明文，输出最终加密的密文，将输出内容粘贴到我们的数据库中，待后续登录时使用。

。。。To be continued

## 四、源码

emmm，私藏的可爱图片也给你们啦

源码地址：[spring-boot-learning](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-22-updownload)
欢迎star、fork，给作者一些鼓励