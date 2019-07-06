---
layout: post
title: Spring Boot2(七)：过滤器拦截器的使用
category: springboot
tags: [springboot]
copyright: java


---

## 一、前言

过滤器和拦截器两者都具有AOP的切面思想，关于aop切面，可以看上一篇文章。过滤器filter和拦截器interceptor都属于面向切面编程的具体实现。

## 二、过滤器

### 过滤器工作原理

![1](https://niaobulashi.com/usr/uploads/2019/07/1012433218.png)

从上图可以看出，当浏览器发送请求到服务器时，先执行过滤器，然后才访问Web资源。服务器响应Response，从Web资源抵达浏览器之前，也会途径过滤器。

过滤器是一个实现javax.servlet.Filter接口的Java类。javax.servlet.Filter接口定义了三个方法

| **方法**                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public void init(FilterConfig filterConfig)                  | web 应用程序启动时，web 服务器将创建Filter 的实例对象，并调用其init方法，读取web.xml配置，完成对象的初始化功能，从而为后续的用户请求作好拦截的准备工作（filter对象只会创建一次，init方法也只会执行一次）。开发人员通过init方法的参数，可获得代表当前filter配置信息的FilterConfig对象。 |
| public void doFilter (ServletRequest, ServletResponse, FilterChain) | 该方法完成实际的过滤操作，当客户端请求方法与过滤器设置匹配的URL时，Servlet容器将先调用过滤器的doFilter方法。FilterChain用户访问后续过滤器。 |
| public void destroy()                                        | Servlet容器在销毁过滤器实例前调用该方法，在该方法中释放Servlet过滤器占用的资源。 |

SpringBoot摒弃了繁琐的xml配置的同时，提示了几种注册组件：ServletRegistrationBean，
FilterRegistrationBean，ServletListenerRegistrationBean，DelegatingFilterProxyRegistrationBean，用于注册自对应的组件，如过滤器，监听器等。

### 代码实现

#### 1、添加maven依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
<!--web-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--devtools热部署-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-devtools</artifactId>
	<scope>runtime</scope>
</dependency>
```

#### 2、添加拦截器

```java
@Configuration
public class WebConfig {
    @Bean
    public RemoteIpFilter remoteIpFilter() {
        return new RemoteIpFilter();
    }
    
	/**
     * 注册第三方过滤器
     * 功能与spring mvc中通过配置web.xml相同
     * 可以添加过滤器锁拦截的 URL，拦截更加精准灵活
     * @return
     */
    @Bean
    public FilterRegistrationBean testFilterRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new MyFilter());
        // 过滤应用程序中所有资源,当前应用程序根下的所有文件包括多级子目录下的所有文件，注意这里*前有“/”
        registration.addUrlPatterns("/*");
        registration.addInitParameter("paramName", "paramValue");
        registration.setName("MyFilter");
        // 过滤器顺序
        registration.setOrder(1);
        return registration;
    }

    // 定义过滤器
    public class MyFilter implements Filter {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
            System.out.println("init");
        }

        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
            HttpServletRequest request = (HttpServletRequest) servletRequest;
            System.out.println("this is MyFilter,url :" + request.getRequestURI());
            filterChain.doFilter(servletRequest, servletResponse);
        }

        @Override
        public void destroy() {
            System.out.println("destroy");
        }
    }
}
```

#### 3、controller层

```java
@RestController
public class HelloController {

    @GetMapping("/filter")
    public String testFilter(){
        return "filter is ok";
    }
}
```

#### 4、测试

通过发送post请求：127.0.0.1:8081/filter

查看日志可以看到过滤器已经开始工作了。![测试截图Filter](https://niaobulashi.com/usr/uploads/2019/07/3298742304.png)

## 三、拦截器

### 拦截器概念

不同于过滤器，具体区别我们下面再将，先讲一讲拦截器实现的机制。

在AOP（Aspect-Oriented Programming）中用于在某个方法或字段被访问之前，进行拦截，然后在之前或之后加上某些操作。拦截是AOP的一种实现策略。

### 拦截器作用

有什么作用呢？AOP面向切面有什么作用，那么拦截器就有什么作用。

- 日志记录：记录请求信息的日志，以便进行信息监控、信息统计、计算PV...
- 权限检查：认证或者授权等检查
- 性能监控：通过拦截器在进入处理器之前记录开始时间，处理完成后记录结束时间，得到请求处理时间。
- 通用行为：读取cookie得到用户信息并将用户对象放入请求头中，从而方便后续流程使用。

### 拦截器实现

拦截器集成接口`HandlerInterceptor`，实现拦截，接口方法有下面三种：

1. `preHandler(HttpServletRequest request, HttpServletResponse response, Object handler)`
    方法将在**请求处理之前**进行调用。SpringMVC中的`Interceptor`同Filter一样都是**链式调用**。每个Interceptor的调用会依据它的声明顺序依次执行，而且最先执行的都是Interceptor中的preHandle方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求的一个预处理，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。该方法的返回值是布尔值Boolean 类型的，当它返回为false时，表示请求结束，后续的Interceptor和Controller都不会再执行；当返回值为true时就会继续调用下一个Interceptor 的preHandle 方法，如果已经是最后一个Interceptor 的时候就会是调用当前请求的Controller 方法。

2. `postHandler(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)`
    在当前**请求进行处理之后**，也就是Controller 方法调用之后执行，但是它会在DispatcherServlet 进行视图返回渲染之前被调用，所以我们可以在这个方法中对Controller 处理之后的ModelAndView 对象进行操作。

3. `afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handle, Exception ex)`
    该方法也是需要当前对应的Interceptor的preHandle方法的返回值为true时才会执行。顾名思义，该方法将在整个请求结束之后，也就是在DispatcherServlet **渲染了对应的视图之后执行**。这个方法的主要作用是用于进行资源清理工作的。

总结一点就是：

preHandle是请求执行前执行

postHandle是请求结束执行

afterCompletion是视图渲染完成后执行

### 代码实现

#### 1、添加Maven依赖

和过滤器一样

#### 2、添加拦截器类

其中`LogInterceptor`实现`HandlerInterceptor`接口的三个方法，同时需要`preHandle`返回true，该方法通常用于清理资源等工作。

主方法继承`WebMvcConfigurer`

注意不用用`WebMvcConfigurerAdapter`，该方法已经被官方标注过时了，在java8是默认实现的。

所以我们需要使用的是`WebMvcConfigurer`进行静态资源的配置。

配置的主要有两项：一个是制定拦截器，第二个是指定拦截的URL

```java
@Slf4j
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    /**
     * 拦截器注册类
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor()).addPathPatterns("/**");
    }
    /**
     * 定义拦截器
     */
    public class LogInterceptor implements HandlerInterceptor {
        long start = System.currentTimeMillis();

        /**
         * 请求执行前执行
         */
        @Override
        public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
            log.info("preHandle");
            start = System.currentTimeMillis();
            return true;
        }
        /**
         * 请求结束执行
         */
        @Override
        public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
            log.info("Interceptor cost="+(System.currentTimeMillis()-start));
        }
        /**
         * 视图渲染完成后执行
         */
        @Override
        public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
            log.info("afterCompletion");
        }
    }
}
```

#### 3、controller层

```java
@RestController
public class HelloController {

    @RequestMapping("/interceptor")
    public String home(){
        return "interceptor is ok";
    }
}
```

#### 4、测试

可以看到，我们通过拦截器实现了同样的功能。不过这里还要说明一点的是，其实这个实现是有问题的，因为preHandle和postHandle是两个方法，所以我们这里不得不设置一个共享变量start来存储开始值，但是这样就会存在线程安全问题。当然，我们可以通过其他方法来解决，比如通过ThreadLocal就可以很好的解决这个问题，有兴趣的同学可以自己实现。不过通过这一点我们其实可以看到，虽然拦截器在很多场景下优于过滤器，但是在这种场景下，过滤器比拦截器实现起来更简单。

![测试截图Interceptor](https://niaobulashi.com/usr/uploads/2019/07/1217141912.png)



## 四、过滤器和拦截器的区别

Spring的拦截器与Servlet的Filter有相似之处，比如二者都是AOP编程思想的体现，都能实现权限检查、日志记录等。

不同的是:

- 使用范围不同：Filter是Servlet规范规定的，只能用于Web程序中。而拦截器既可以用于Web程序，也可以用于Application、Swing程序中。
- 规范不同: Filter是在Servlet规范中定义的，是Servlet容器支持的。而拦截器是在Spring容器内的，是Spring框架支持的。
- 使用的资源不同：同其他的代码块一样，拦截器也是一个Spring的组件，归Spring管理，配置在Spring文件中，因此能使用Spring里的任何资源、对象，例如Service对象、数据源、事务管理等，通过loC注入到拦截器即可:而Filter则不能。
- 深度不同：Filter在只在Servlet前后起作用。而拦截器能够深入到方法前后、异常抛出前后等，因此拦截器的使用具有更大的弹性。所以在Spring构架的程序中，要优先使用拦截器。

## 五、总结
注意：过滤器的触发时机是容器后，servlet之前，所以过滤器的doFilter(ServletRequest request, ServletResponse response, FilterChain chain)的入参是ServletRequest，而不是HttpServletRequest，因为过滤器是在HttpServlet之前。下面这个图，可以让你对Filter和Interceptor的执行时机有更加直观的认识。
![过滤器和拦截器关系图](https://niaobulashi.com/usr/uploads/2019/07/880490333.png)
只有经过DispatcherServlet 的请求，才会走拦截器链，自定义的Servlet请求是不会被拦截的，比如我们自定义的Servlet地址。

过滤器依赖于Servlet容器，而Interceptor则为SpringMVC的一部分。过滤器能够拦截所有请求，而Interceptor只能拦截Controller的请求，所以从覆盖范围来看，Filter应用更广一些。但是在Spring逐渐一统Java框架、前后端分离越演越烈，实际上大部分的应用场景，拦截器都可以满足了。

## 六、源码
[SpringBoot-过滤器spring-boot-16-filter](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-16-filter)

[SpringBoot-拦截器spring-boot-17-interceptor](https://github.com/niaobulashi/spring-boot-learning/tree/master/spring-boot-17-interceptor)

## 七、参考

[SpringBoot实现过滤器、拦截器与切片](https://juejin.im/post/5c6901206fb9a049af6dcdcf)
[Spring Boot实战：拦截器与过滤器](https://www.cnblogs.com/paddix/p/8365558.html)
[Spring Boot使用过滤器和拦截器分别实现REST接口简易安全认证](https://www.cnblogs.com/jeffwongishandsome/p/spring-boot-use-filter-and-interceptor-to-implement-an-easy-auth-system.html)