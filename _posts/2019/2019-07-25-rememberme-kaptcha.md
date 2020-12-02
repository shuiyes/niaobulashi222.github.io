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

实现了Cookie对象属性配置，还需要通过`CookieRememberMeManager`进行管理起来。

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

### 加密处理

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

其中`SALT`是加密的盐，可自行定义。

main方法中，根据登录名和密码明文，输出最终加密的密文，将输出内容粘贴到我们的数据库中，待后续登录时使用。

### 新增登录页面和主页面

登录页login.html

添加Remember Me checkbox

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
    <link rel="stylesheet" th:href="@{/static/css/login.css}" type="text/css">
    <script th:src="@{/static/js/jquery-1.11.1.min.js}"></script>
</head>
<body>
<div class="login-page">
    <div class="form">
        <input type="text" placeholder="用户名" name="account" required="required"/>
        <input type="password" placeholder="密码" name="password" required="required"/>
        <p><input type="checkbox" name="rememberMe"/>记住我</p>
        <button onclick="login()">登录</button>
    </div>
</div>
</body>
<script th:inline="javascript">var ctx = [[@{/}]];</script>
<script th:inline="javascript">
    function login() {
        var account = $("input[name='account']").val();
        var password = $("input[name='password']").val();
        var rememberMe = $("input[name='rememberMe']").is(':checked');
        $.ajax({
            type: "post",
            url: ctx + "login",
            data: {
                "account": account,
                "password": password,
                "rememberMe": rememberMe
            },
            success: function(r) {
                if (r.code == 100) {
                    location.href = ctx + 'index';
                } else {
                    alert(r.message);
                }
            }
        });
    }
</script>
</html>
```

静态资源js和css可以在源码中查看

![登录页面](https://images.niaobulashi.com/typecho/uploads/2019/07/2309019532.png)

首页index.html

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>首页</title>
</head>
<body>
<p>你好！[[${user.getUsername()}]]</p>
<a th:href="@{/logout}">注销</a>
</body>
</html>
```

### Controller层

在原来的基础上，新增参数rememberMe，同时对用户名和明文密码进行MD5加密处理获得密文。

登录接口

```java
/**
 * 登录操作
 * @param account
 * @param password
 * @param rememberMe
 * @return
 */
@PostMapping("/login")
@ResponseBody
public ResponseCode login(String account, String password, Boolean rememberMe) {
	logger.info("登录请求-start");
	password = MD5Utils.encrypt(account, password);
	Subject userSubject = SecurityUtils.getSubject();
	UsernamePasswordToken token = new UsernamePasswordToken(account, password, rememberMe);
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
	} catch (AuthenticationException e) {
		return ResponseCode.error(StatusEnums.AUTH_ERROR);
	} catch (Throwable e) {
		e.printStackTrace();
		return ResponseCode.error(StatusEnums.SYSTEM_ERROR);
	}
}
```

注销接口

```
/**
 * 登出
 * @return
 */
@GetMapping("/logout")
public String logout() {
	getSubject().logout();
	return "login";
}
```

启动项目，进行测试可以看到效果如下：

![登录操作](https://images.niaobulashi.com/typecho/uploads/2019/07/2305576115.gif)

## 二、验证码Kaptcha

kaptcha 是一个非常实用的验证码生成工具。有了它，你可以生成各种样式的验证码，因为它是可配置的。kaptcha工作的原理是调用 com.google.code.kaptcha.servlet.KaptchaServlet，生成一个图片。同时将生成的验证码字符串放到 HttpSession中。

Kaptcha官网：https://code.google.com/archive/p/kaptcha/

使用kaptcha可以方便的配置：

- 验证码的字体
- 验证码字体的大小
- 验证码字体的字体颜色
- 验证码内容的范围(数字，字母，中文汉字！)
- 验证码图片的大小，边框，边框粗细，边框颜色
- 验证码的干扰线(可以自己继承com.google.code.kaptcha.NoiseProducer写一个自定义的干扰线)
- 验证码的样式(鱼眼样式、3D、普通模糊……当然也可以继承com.google.code.kaptcha.GimpyEngine自定义样式)

### kaptcha配置详解

|kaptcha对象属性|作用|默认值|
| ---- | ---- | ---- |
|kaptcha.border|是否有边框|默认为true|
|kaptcha.border.color|边框颜色|默认为Color.BLACK|
|kaptcha.border.thickness|边框粗细度|默认为1|
|kaptcha.producer.impl|验证码生成器|默认为DefaultKaptcha|
|kaptcha.textproducer.impl|验证码文本生成器|默认为DefaultTextCreator|
|kaptcha.textproducer.char.string|验证码文本字符内容范围|默认为abcde2345678gfynmnpwx|
|kaptcha.textproducer.char.length|验证码文本字符长度|默认为5|
|kaptcha.textproducer.font.names|验证码文本字体样式|宋体,楷体,微软雅黑，默认为new Font("Arial", 1, fontSize), new Font("Courier", 1, fontSize)|
|kaptcha.textproducer.font.size|验证码文本字符大小|默认为40|
|kaptcha.textproducer.font.color|验证码文本字符颜色|默认为Color.BLACK|
|kaptcha.textproducer.char.space|验证码文本字符间距|默认为2|
|kaptcha.noise.impl|验证码噪点生成对象|默认为DefaultNoise|
|kaptcha.noise.color|验证码噪点颜色|默认为Color.BLACK|
|kaptcha.obscurificator.impl|验证码样式引擎|默认为WaterRipple|
|kaptcha.word.impl|验证码文本字符渲染|默认为DefaultWordRenderer|
|kaptcha.background.impl|验证码背景生成器|默认为DefaultBackground|
|kaptcha.background.clear.from|验证码背景颜色渐进|默认为Color.LIGHT_GRAY|
|kaptcha.background.clear.to|验证码背景颜色渐进|默认为Color.WHITE|
|kaptcha.image.width|验证码图片宽度|默认为200|
|kaptcha.image.height|验证码图片高度|默认为50|

### 添加maven依赖

```yaml
<!--验证码-->
<dependency>
	<groupId>com.github.penggle</groupId>
	<artifactId>kaptcha</artifactId>
	<version>2.3.2</version>
</dependency>
```

### 新增验证码图片样式配置器

具体配置可以参考上面的**kaptche配置详情**，针对不同的常见配置。

```java
@Configuration
public class KaptchaConfig {

    @Bean(name="captchaProducer")
    public DefaultKaptcha getKaptchaBean(){
        DefaultKaptcha defaultKaptcha=new DefaultKaptcha();
        Properties properties=new Properties();
        //验证码字符范围
        properties.setProperty("kaptcha.textproducer.char.string", "23456789");
        //图片边框颜色
        properties.setProperty("kaptcha.border.color", "245,248,249");
        //字体颜色
        properties.setProperty("kaptcha.textproducer.font.color", "black");
        //文字间隔
        properties.setProperty("kaptcha.textproducer.char.space", "1");
        //图片宽度
        properties.setProperty("kaptcha.image.width", "100");
        //图片高度
        properties.setProperty("kaptcha.image.height", "35");
        //字体大小
        properties.setProperty("kaptcha.textproducer.font.size", "30");
        //session的key
        //properties.setProperty("kaptcha.session.key", "code");
        //长度
        properties.setProperty("kaptcha.textproducer.char.length", "4");
        //字体
        properties.setProperty("kaptcha.textproducer.font.names", "宋体,楷体,微软雅黑");
        Config config=new Config(properties);
        defaultKaptcha.setConfig(config);
        return defaultKaptcha;
    }
}
```

### 新增图片验证码Controller层

是一个创建文件图片流的过程，使用ServletOutPutStream输出最后的图片。

开头声明的`@Resource(name = "captchaProducer")`，是验证码图片样式配置器启动时配置的Bean：`captchaProducer`。

```java
@Controller
@RequestMapping("/captcha")
public class KaptchaController {

    private static final Logger logger = LoggerFactory.getLogger(KaptchaController.class);

    @Resource(name = "captchaProducer")
    private Producer captchaProducer;

    @GetMapping("/captchaImage")
    public ModelAndView getKaptchaImage(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ServletOutputStream out = response.getOutputStream();
        try {
            HttpSession session = request.getSession();
            response.setDateHeader("Expires", 0);
            // Set standard HTTP/1.1 no-cache headers.
            response.setHeader("Cache-Control", "no-store, no-cache, must-revalidate");
            // Set IE extended HTTP/1.1 no-cache headers (use addHeader).
            response.addHeader("Cache-Control", "post-check=0, pre-check=0");
            // Set standard HTTP/1.0 no-cache header.
            response.setHeader("Pragma", "no-cache");
            // return a jpeg
            response.setContentType("image/jpeg");
            // create the text for the image
            String capText = captchaProducer.createText();
            //将验证码存到session
            session.setAttribute(Constants.KAPTCHA_SESSION_KEY, capText);
            logger.info(capText);
            // 创建一张文本图片
            BufferedImage bi = captchaProducer.createImage(capText);
            // 响应
            out = response.getOutputStream();
            // 写入数据
            ImageIO.write(bi, "jpg", out);

            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (out != null) {
                    out.close();
                }
            }
            catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```

注意最后都需要将流关闭`out.close()`

### 放开图片验证码的拦截

重启会发现，图片验证码的接口请求无法访问，还是跳转到了localhost:8081/login登录页面

因为Shiro配置的拦截器没有放开，需要再`ShiroConfig`中允许匿名访问改请求资源

```java
map.put("/captcha/captchaImage**", "anon");
```

### 登录页面添加图片验证码

```html
<div class="login-page">
    <div class="form">
        <input type="text" placeholder="用户名" name="account" required="required"/>
        <input type="password" placeholder="密码" name="password" required="required"/>
        <p>
            <label>验证码<br/>
                <input type="text" name="validateCode" id="validateCode" class="validateCode" required="required"/>
                <a href="javascript:void(0);">
                    <img src="/captcha/captchaImage" onclick="this.src='/captcha/captchaImage?'+Math.random()"/>
                </a>
            </label>
        </p>
        <br>
        <p><input type="checkbox" name="rememberMe"/>记住我</p>
        <button onclick="login()">登录</button>
    </div>
</div>
```

上面`div`为body的全部部分

我在请求`/captcha/captchaImage`后面添加随机值`Math.random()`。是因为客户浏览器会缓存URL相同的资源，故使用随机数来重新请求。这和前端上线时，请求后缀都会变更一个版本号一样，不需要让客户手动刷新浏览器就可以获取最新资源一样。

![验证码请求](https://images.niaobulashi.com/typecho/uploads/2019/07/2338195235.gif)

### 修改登录请求接口

主要是验证后台生成的验证码，与前台输入的验证码进行比较，验证是否相同

这里只粘贴出验证码验证的逻辑，源码在文章最后。

可以看出`validateCode`是前端请求过来的参数，先校验是否为空。

然后从session中获取后台生成的验证码。

最后通过比较前端输入的验证码和后台生成的是否一致。

```java
//1、检验验证码
if(validateCode == null || validateCode == ""){
	return ResponseCode.error(StatusEnums.PARAM_NULL);
}
Session session = SecurityUtils.getSubject().getSession();
//转化成小写字母
validateCode = validateCode.toLowerCase();
String v = (String) session.getAttribute(Constants.KAPTCHA_SESSION_KEY);
//还可以读取一次后把验证码清空，这样每次登录都必须获取验证码
//session.removeAttribute("_come");
if(!validateCode.equals(v)){
	return ResponseCode.error(StatusEnums.VALIDATECODE_ERROR);
}
```

下图是登录校验验证码的debug过程。

![kaptcha验证码校验](https://images.niaobulashi.com/typecho/uploads/2019/07/2915987549.gif)

## 三、源码

源码地址：[**spring-boot-23-shiro-remember**](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-23-shiro-remember)
欢迎star、fork，给作者一些鼓励

---

菜鸟也要成为架构师，一起努力

欢迎关注我微信公众号【鸟不拉诗】

谢谢，一起学习，共同进步，成为优秀的人。

![微信公众号：鸟不拉诗](https://niaobulashi.com/usr/uploads/2019/07/2427016822.png)