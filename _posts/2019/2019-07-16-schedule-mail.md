---
layout: post
title: Spring Boot2(十三)：整合定时任务发送邮件
category: springboot
tags: [springboot]
copyright: java
---

主要玩一下SpringBoot的定时任务和发送邮件的功能。定时发送邮件，这在实际生成环境下主要用户系统性能监控时，当超过设定的阙值，就发送邮件通知预警功能。这里只通过简单的写个定时结合邮件通知进行学习。

## 一、准备

### 添加maven依赖

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>

<!--mail邮件-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-mail</artifactId>
</dependency>

<!--thymeleaf前端模板-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

### 配置文件application.yml

```yaml
spring:
  mail:
    #邮箱服务器地址
    host: smtp.qq.com
    username: hulang6666@qq.com
    password: **********
    default-encoding: UTF-8

mail:
  #以谁来发送邮件
  fromMail:
    addr: hulang6666@qq.com
```

这里的`spring.mail.password`为你的邮箱开启smtp服务需要设置客户端授权码，此处的password为你的验证密码。注意不是你的qq登录密码。

这里需要注意的一点是`spring.mail.host`为**邮箱服务地址**

### 常见的邮件服务器扩展

（SMTP、POP3）地址、端口如下：

**gmail(google.com)**
POP3服务器地址:pop.gmail.com（SSL启用 端口：995）
SMTP服务器地址:smtp.gmail.com（SSL启用 端口：587）

**Foxmail:** 
POP3服务器地址:pop.foxmail.com（端口：110）
SMTP服务器地址:smtp.foxmail.com（端口：25）

**sina.com:** 
POP3服务器地址:pop3.sina.com.cn（端口：110）
SMTP服务器地址:smtp.sina.com.cn（端口：25） 

**163.com:** 
POP3服务器地址:pop.163.com（端口：110）
SMTP服务器地址:smtp.163.com（端口：25）

**QQ邮箱**
POP3服务器地址:pop.qq.com（端口：110）
SMTP服务器地址:smtp.qq.com（端口：25）

**QQ企业邮箱**
POP3服务器地址:pop.exmail.qq.com（端口：995）
SMTP服务器地址:smtp.exmail.qq.com（端口：587/465）

**HotMail**
POP3服务器地址:pop.live.com（端口：995）
SMTP服务器地址:smtp.live.com（端口：587）

**sohu.com:** 
POP3服务器地址:pop3.sohu.com（端口：110）
SMTP服务器地址:smtp.sohu.com（端口：25）

## 二、邮件服务

我们使用html模板并且带有附件的例子。

### MailService

```
public interface MailService {

    void sendHtmlMail(String to, String subject, String content, String filePath);

}
```

### MailServiceImpl

```
@Component
public class MailServiceImpl implements MailService {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Resource
    private JavaMailSender mailSender;

    @Value("${mail.fromMail.addr}")
    private String from;

    /**
     * 发送html邮件
     * @param to
     * @param subject
     * @param content
     */
    @Override
    public void sendHtmlMail(String to, String subject, String content, String filePath) {
        MimeMessage message = mailSender.createMimeMessage();

        try {
            //true表示需要创建一个multipart message
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(from);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(content, true);

            // 判断是否带有附件
            if (filePath != null) {
                FileSystemResource file = new FileSystemResource(new File(filePath));
                String fileName = filePath.substring(filePath.lastIndexOf(File.separator));
                helper.addAttachment(fileName, file);
            }

            mailSender.send(message);
            logger.info("html邮件发送成功");
        } catch (MessagingException e) {
            logger.error("发送html邮件时发生异常！", e);
        }
    }
}
```

### 新增邮件模板

sendMail.html

```
<!DOCTYPE html>
<html lang="zh" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>Title</title>
</head>
<body>
<div style="border-radius:5px;font-size:19px;width:680px;font-family:微软雅黑,'Helvetica Neue',Arial,sans-serif;margin:10px auto 0px;border:1px solid #eee;max-width:100%;">
    <div style="width:100%;background:#49BDAD;color:#FFFFFF;border-radius:5px 5px 0 0;">
        <p style="font-size:22px;word-break:break-all;padding:20px 32px;margin:0;"><span th:text="${siteTitle}"></span>《<a style="color:#FFFFFF;font-weight:bold;text-decoration:none;" target="_blank" th:href="${permalink}"><span th:text="${title}"></span></a>》一文有新的评论啦！</p>
    </div>
    <div style="margin:0px auto;width:90%">
        <p><span th:text="${author}"/> 于 <span th:text="${time}"/></p>
        <p>&nbsp;&nbsp;&nbsp;&nbsp;在《<span th:text="${title}"/>》评论说：</p>
        <p style="background:#EFEFEF;margin:15px 0px;padding:20px;border-radius:5px;font-size:20px;color:#333;"><span th:text="${text}"/></p>
        <p>IP：<span th:text="${ip}"/>，邮箱：<span th:text="${mail}"/>，审核：<span th:text="${status}"/>。</p>
        <p>可登录<a th:href="${manage}" target='_blank'>网站后台</a>管理评论。</p>
    </div>
</div>

</body>
</html>
```

### 测试

```
@Test
public void sendTemplateMail() {
	//创建邮件字段
	Context context = new Context();
	context.setVariable("siteTitle", "鸟不拉诗");
	context.setVariable("permalink", "https://niaobulashi.com/archives/canteen.html/comment-page-1#comment-1152");
	context.setVariable("title", "公司食堂伙食看起来还不错的亚子（体重有所回升）");
	context.setVariable("author", "测试员");
	context.setVariable("time", "2019-07-16 08:52:46");
	context.setVariable("text", "真的很不错！");
	context.setVariable("ip", "127.0.0.1");
	context.setVariable("mail", "123321@qq.com");
	context.setVariable("status", "通过");
	context.setVariable("manage", "https://niaobulashi.com");
	// 将字段加载到页面模板中
	String emailContent = templateEngine.process("sendMail", context);
	// 添加附件
	String filePath="E:\\workspace\\javaWorkspace\\spring-boot-learning\\spring-boot-21-schedule-mail\\doc\\test.log";
	mailService.sendHtmlMail("hulang6666@qq.com","主题：这是模板邮件",emailContent, filePath);
}
```

## 三、定时任务

定时任务在SpringBoot默认的SpringBootStart包中已经存在

### 启动类开启定时任务

```
@SpringBootApplication
@EnableScheduling
public class ScheduleMailApplication {

    public static void main(String[] args) {
        SpringApplication.run(ScheduleMailApplication.class, args);
    }

}
```

### 创建定时任务

```
@Component
public class SchedulerTask {

    private int count=0;

    @Scheduled(cron="*/8 * * * * ?")
    private void process(){
        System.out.println("定时任务开启，以跑：  "+(count++));
    }

}
```

### Quart Cron表达式扩展

cron的表达式是字符串，实际上是由七子表达式，描述个别细节的时间表。

1. ​       **Seconds**
2. ​       **Minutes**
3. ​       **Hours**
4. ​       **Day-of-Month**
5. ​       **Month**
6. ​       **Day-of-Week**
7. ​      **Year (可选字段)**

​     1）Cron表达式的格式：秒 分 时 日 月 周 年(可选)。

​               字段名                 允许的值                        允许的特殊字符                 

​                 秒                      0-59                                   , - * /                 

​                 分                      0-59                                   , - * /                 

​               小时                     0-23                                   , - * /                 

​                 日                      1-31                                   , - * ? / L W C                 

​                 月                      1-12 or JAN-DEC                 , - * /                 

​                周几                    1-7 or SUN-SAT                  , - * ? / L C #                 

​              年 (可选字段)         empty, 1970-2099             , - * /

​             

​              “*” 代表整个时间段

​               “?”字符：表示不确定的值

​               “,”字符：指定数个值

​               “-”字符：指定一个值的范围

​               “/”字符：指定一个值的增加幅度。n/m表示从n开始，每次增加m

​               “L”字符：用在日表示一个月中的最后一天，用在周表示该月最后一个星期X

​               “W”字符：指定离给定日期最近的工作日(周一到周五)

​               “#”字符：表示该月第几个周X。6#3表示该月第3个周五

​        2）Cron表达式范例：

​                 每隔5秒执行一次：*/5 * * * * ?

​                 每隔1分钟执行一次：0 */1 * * * ?

​                 每天23点执行一次：0 0 23 * * ?

​                 每天凌晨1点执行一次：0 0 1 * * ?

​                 每月1号凌晨1点执行一次：0 0 1 1 * ?

​                 每月最后一天23点执行一次：0 0 23 L * ?

​                 每周星期天凌晨1点实行一次：0 0 1 ? * L

​                 在26分、29分、33分执行一次：0 26,29,33 * * * ?

​                 每天的0点、13点、18点、21点都执行一次：0 0 0,13,18,21 * * ?

**Corn表达式在线验证：http://cron.qqe2.com/**

 ![img](https://images2015.cnblogs.com/blog/903762/201609/903762-20160901193022230-1358890324.png)

## 四、定时发送邮件

定时1分钟发送邮件

### SchedulerTask

```
@Component
public class SchedulerTask {

    private int count=0;

    @Autowired
    private MailService mailService;

    @Autowired
    private TemplateEngine templateEngine;

    /**
     * 每隔一分钟执行一次
     */
    @Scheduled(cron="0 */1 * * * ?")
    private void process(){

        System.out.println("this is scheduler task runing  "+(count++));
        //创建邮件字段
        Context context = new Context();
        context.setVariable("siteTitle", "鸟不拉诗");
        context.setVariable("permalink", "https://niaobulashi.com/archives/canteen.html/comment-page-1#comment-1152");
        context.setVariable("title", "公司食堂伙食看起来还不错的亚子（体重有所回升）");
        context.setVariable("author", "测试员");
        context.setVariable("time", "2019-07-16 08:52:46");
        context.setVariable("text", "真的很不错！");
        context.setVariable("ip", "127.0.0.1");
        context.setVariable("mail", "123321@qq.com");
        context.setVariable("status", "通过");
        context.setVariable("manage", "https://niaobulashi.com");
        // 将字段加载到页面模板中
        String emailContent = templateEngine.process("sendMail", context);
        // 添加附件
        String filePath="E:\\workspace\\javaWorkspace\\spring-boot-learning\\spring-boot-21-schedule-mail\\doc\\test.log";
        mailService.sendHtmlMail("hulang6666@qq.com","主题：这是模板邮件",emailContent, filePath);
    }
}
```

### 测试

![第二次发送成功](https://user-gold-cdn.xitu.io/2019/7/16/16bfaf6843249cc6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

![邮件发送测试截图](https://user-gold-cdn.xitu.io/2019/7/16/16bfaf6a4ad95170?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



源码地址：<https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-21-schedule-mail>

---

