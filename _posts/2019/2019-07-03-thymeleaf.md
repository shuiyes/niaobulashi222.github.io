---
layout: post
title: Spring Boot2(五)：使用Spring Boot结合Thymeleaf模板引擎使用总结
category: springboot
tags: [springboot]
copyright: Java
---

一般来说，常用的模板引擎有JSP、Velocity、Freemarker、Thymeleaf 。

SpringBoot推荐的 Thymeleaf – 语法更简单，功能更强大；

Thymeleaf是一种Java XML/XHTML/HTML5模板引擎，可以在Web和非Web环境中使用。
它更适合在基于MVC的Web应用程序的视图层提供XHTML/HTML5，但即使在脱机环境中，它也可以处理任何XML文件。它提供了完整的Spring Framework集成。

## 一、 标准表达式语法

它们分为四类：

- 1.变量表达式
- 2.选择或星号表达式
- 3.文字国际化表达式
- 4.URL 表达式

### 变量表达式

变量表达式即 OGNL 表达式或 Spring EL 表达式(在 Spring 术语中也叫 model attributes)。如下所示：
`${session.user.name}`

它们将以HTML标签的一个属性来表示：

```
<span th:text="${book.author.name}">  
<li th:each="book : ${books}">  
```

### 选择(星号)表达式

选择表达式很像变量表达式，不过它们用一个预先选择的对象来代替上下文变量容器(map)来执行，如下：
`*{customer.name}`

被指定的 object 由 th:object 属性定义：

```
<div th:object="${book}">  
  ...  
  <span th:text="*{title}">...</span>  
  ...  
</div>  
```

### 文字国际化表达式

文字国际化表达式允许我们从一个外部文件获取区域文字信息(.properties)，用 Key 索引 Value，还可以提供一组参数(可选).

```
#{main.title}  
#{message.entrycreated(${entryId})}  
```

可以在模板文件中找到这样的表达式代码：

```
<table>  
  ...  
  <th th:text="#{header.address.city}">...</th>  
  <th th:text="#{header.address.country}">...</th>  
  ...  
</table>  
```

### URL 表达式

URL 表达式指的是把一个有用的上下文或回话信息添加到 URL，这个过程经常被叫做 URL 重写。 
`@{/order/list}`

URL还可以设置参数： 
`@{/order/details(id=${orderId})}`

相对路径： 
`@{../documents/report}`

让我们看这些表达式：

```
<form th:action="@{/createOrder}">  
<a href="main.html" th:href="@{/main}">
```

### 变量表达式和星号表达有什么区别吗？

如果不考虑上下文的情况下，两者没有区别；星号语法评估在选定对象上表达，而不是整个上下文 
什么是选定对象？就是父标签的值，如下：

```
<div th:object="${session.user}">
  <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
  <p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
  <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
</div>
```

这是完全等价于：

```
<div th:object="${session.user}">
  <p>Name: <span th:text="${session.user.firstName}">Sebastian</span>.</p>
  <p>Surname: <span th:text="${session.user.lastName}">Pepper</span>.</p>
  <p>Nationality: <span th:text="${session.user.nationality}">Saturn</span>.</p>
</div>
```

当然，美元符号和星号语法可以混合使用：

```
  <div th:object="${session.user}">
	  <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
  	  <p>Surname: <span th:text="${session.user.lastName}">Pepper</span>.</p>
      <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
  </div>
```

### 表达式支持的语法

#### 字面（Literals）

- 文本文字（Text literals）: `'one text', 'Another one!',…`
- 数字文本（Number literals）: `0, 34, 3.0, 12.3,…`
- 布尔文本（Boolean literals）:`true, false`
- 空（Null literal）:`null`
- 文字标记（Literal tokens）:`one, sometext, main,…`

#### 文本操作（Text operations）

- 字符串连接(String concatenation):`+`
- 文本替换（Literal substitutions）:`|The name is ${name}|`

#### 算术运算（Arithmetic operations）

- 二元运算符（Binary operators）:`+, -, *, /, %`
- 减号（单目运算符）Minus sign (unary operator):`-`

#### 布尔操作（Boolean operations）

- 二元运算符（Binary operators）:`and, or`
- 布尔否定（一元运算符）Boolean negation (unary operator):`!, not`

#### 比较和等价(Comparisons and equality)

- 比较（Comparators）:`>, <, >=, <= (gt, lt, ge, le)`
- 等值运算符（Equality operators）:`==, != (eq, ne)`

#### 条件运算符（Conditional operators）

- If-then:`(if) ? (then)`
- If-then-else:`(if) ? (then) : (else)`
- Default: (value) ?:`(defaultvalue)`

所有这些特征可以被组合并嵌套：

```
'User is of type ' + (${user.isAdmin()} ? 'Administrator' : (${user.type} ?: 'Unknown'))
```

## 二、常用的th标签

官方文档详细的一批：
<https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html>

| 关键字      | 功能介绍                                     | 案例                                                         |
| :---------- | :------------------------------------------- | :----------------------------------------------------------- |
| th:id       | 替换id                                       | `<input th:id="'xxx' + ${collect.id}"/>`                     |
| th:text     | 文本替换                                     | `<p th:text="${collect.description}">description</p>`        |
| th:utext    | 支持html的文本替换                           | `<p th:utext="${htmlcontent}">conten</p>`                    |
| th:object   | 替换对象                                     | `<div th:object="${session.user}"> `                         |
| th:value    | 属性赋值                                     | `<input th:value="${user.name}" /> `                         |
| th:with     | 变量赋值运算                                 | `<div th:with="isEven=${prodStat.count}%2==0"></div> `       |
| th:style    | 设置样式                                     | `th:style="'display:' + @{(${sitrue} ? 'none' : 'inline-block')} + ''" ` |
| th:onclick  | 点击事件                                     | `th:onclick="'getCollect()'" `                               |
| th:each     | 属性赋值                                     | `tr th:each="user,userStat:${users}"> `                      |
| th:if       | 判断条件                                     | ` <a th:if="${userId == collect.userId}" > `                 |
| th:unless   | 和th:if判断相反                              | `<a th:href="@{/login}" th:unless=${session.user != null}>Login</a>` |
| th:href     | 链接地址                                     | `<a th:href="@{/login}" th:unless=${session.user != null}>Login</a> /> ` |
| th:switch   | 多路选择 配合th:case 使用                    | `<div th:switch="${user.role}"> `                            |
| th:case     | th:switch的一个分支                          | `<p th:case="'admin'">User is an administrator</p>`          |
| th:fragment | 布局标签，定义一个代码片段，方便其它地方引用 | `<div th:fragment="alert">`                                  |
| th:include  | 布局标签，替换内容到引入的文件               | `<head th:include="layout :: htmlhead" th:with="title='xx'"></head> /> ` |
| th:replace  | 布局标签，替换整个标签到引入的文件           | `<div th:replace="fragments/header :: title"></div> `        |
| th:selected | selected选择框 选中                          | `th:selected="(${xxx.id} == ${configObj.dd})"`               |
| th:src      | 图片类地址引入                               | `<img class="img-responsive" alt="App Logo" th:src="@{/img/logo.png}" /> ` |
| th:inline   | 定义js脚本可以使用变量                       | `<script type="text/javascript" th:inline="javascript">`     |
| th:action   | 表单提交的地址                               | `<form action="subscribe.html" th:action="@{/subscribe}">`   |
| th:remove   | 删除某个属性                                 | `<tr th:remove="all"> 1.all:删除包含标签和所有的孩子。2.body:不包含标记删除,但删除其所有的孩子。3.tag:包含标记的删除,但不删除它的孩子。4.all-but-first:删除所有包含标签的孩子,除了第一个。5.none:什么也不做。这个值是有用的动态评估。` |
| th:attr     | 设置标签属性，多个属性可以用逗号分隔         | 比如`th:attr="src=@{/image/aa.jpg},title=#{logo}"`，此标签不太优雅，一般用的比较少。 |

还有非常多的标签，这里只列出最常用的几个,由于一个标签内可以包含多个th:x属性，其生效的优先级顺序为:`include,each,if/unless/switch/case,with,attr/attrprepend/attrappend,value/href,src ,etc,text/utext,fragment,remove。 `

## 三、表达式

**简单表达式**

- 变量表达式：${…}
- 选择变量表达式：*{…}
- 消息表达式：#{…}
- 链接表达式：@{…}
- 片段表达：~{…}

**数据的类型**

- 文字：’one text’, ‘Another one!’,…
- 数字文字：0, 34, 3.0, 12.3,…
- 布尔文字：true, false
- NULL文字：null
- 文字标记：one, sometext, main,…

**文本操作**

- 字符串拼接：+
- 字面替换：|The name is ${name}|

**算术运算**

- 二进制运算符：+, -, *, /, %
- 减号(一元运算符)：-

**布尔运算**

- 二进制运算符：and, or
- 布尔否定(一元运算符)：!, false

**条件运算符**

- 比较值：>, <, >=, <=
- 相等判断： ==, !=

**条件判断**

- (if) ? (then)
- (if) ? (then) : (else)
- 三元：(value) ? value : defaultvalue

## 四、表达式对象

表达式里面的对象可以帮助我们处理要展示的内容，比如表达式的工具类dates可以格式化时间，这些内置类的熟练使用，可以让我们使用Thymeleaf的效率提高很多。

- \#ctx: 操作当前上下文.
- \#vars: 操作上下文变量.
- \#request: (仅适用于Web项目) HttpServletRequest对象.
- \#response: (仅适用于Web项目) HttpServletResponse 对象.
- \#session: (仅适用于Web项目) HttpSession 对象.
- \#servletContext: (仅适用于Web项目) ServletContext 对象.

表达式实用工具类：

- \#execInfo: 操作模板的工具类，包含了一些模板信息，比如：${ #execInfo.templateName }
- \#uris: url处理的工具
- \#conversions: methods for executing the configured *conversion service* (if any).
- \#dates: 方法来源于 java.util.Date 对象，用于处理时间，比如：格式化.
- \#calendars: 类似于 #dates, 但是来自于 java.util.Calendar 对象.
- \#numbers: 用于格式化数字.
- \#strings: methods for String objects: contains, startsWith, prepending/appending, etc.
- \#objects: 普通的object对象方法.
- \#bools: 判断bool类型的工具.
- \#arrays: 数组操作工具.
- \#lists: 列表操作数据.
- \#sets: Set操作工具.
- \#maps: Map操作工具.
- \#aggregates: 操作数组或集合的工具.

## 五、几种常用的使用方法

### 1、赋值、字符串拼接

```
<p  th:text="${collect.description}">description</p>
<span th:text="'Welcome to our application, ' + ${user.name} + '!'">
```

字符串拼接还有另外一种简洁的写法

```
<span th:text="|Welcome to our application, ${user.name}!|">
```

### 2、条件判断 If/Unless

Thymeleaf中使用th:if和th:unless属性进行条件判断，下面的例子中，`<a>`标签只有在`th:if`中条件成立时才显示：

```
<a th:if="${myself=='yes'}" > </i> </a>
<a th:unless=${session.user != null} th:href="@{/login}" >Login</a>
```

`th:unless` 于 `th:if` 恰好相反，只有表达式中的条件不成立，才会显示其内容。

也可以使用 `(if) ? (then) : (else)`这种语法来判断显示的内容

### 3、for 循环

```
<tr  th:each="collect,iterStat : ${collects}"> 
   <th scope="row" th:text="${collect.id}">1</th>
   <td >
      <img th:src="${collect.webLogo}"/>
   </td>
   <td th:text="${collect.url}">Mark</td>
   <td th:text="${collect.title}">Otto</td>
   <td th:text="${collect.description}">@mdo</td>
   <td th:text="${terStat.index}">index</td>
</tr>
```

iterStat称作状态变量，属性有：

- index:当前迭代对象的 index（从0开始计算）
- count: 当前迭代对象的 index(从1开始计算)
- size:被迭代对象的大小
- current:当前迭代变量
- even/odd:布尔值，当前循环是否是偶数/奇数（从0开始计算）
- first:布尔值，当前循环是否是第一个
- last:布尔值，当前循环是否是最后一个

### 4、URL

URL 在 Web 应用模板中占据着十分重要的地位，需要特别注意的是 Thymeleaf 对于 URL 的处理是通过语法 `@{...}`来处理的。 如果需要 Thymeleaf 对 URL 进行渲染，那么务必使用 `th:href`，`th:src` 等属性，下面是一个例子

```
<!-- Will produce 'http://localhost:8080/standard/unread' (plus rewriting) -->
 <a  th:href="@{/standard/{type}(type=${type})}">view</a>

<!-- Will produce '/gtvg/order/3/details' (plus rewriting) -->
<a href="details.html" th:href="@{/order/{orderId}/details(orderId=${o.id})}">view</a>
```

设置背景

```
<div th:style="'background:url(' + @{/<path-to-image>} + ');'"></div>
```

根据属性值改变背景

```
 <div class="media-object resource-card-image"  th:style="'background:url(' + @{(${collect.webLogo}=='' ? 'img/favicon.png' : ${collect.webLogo})} + ')'" ></div>
```

几点说明：

- 上例中 URL 最后的`(orderId=${o.id}) `表示将括号内的内容作为 URL 参数处理，该语法避免使用字符串拼接，大大提高了可读性
- `@{...}`表达式中可以通过`{orderId}`访问 Context 中的 orderId 变量
- `@{/order}`是 Context 相关的相对路径，在渲染时会自动添加上当前 Web 应用的 Context 名字，假设 context 名字为 app，那么结果应该是 `/app/order`

### 5、内联 js

内联文本：[[…]] 内联文本的表示方式，使用时，必须先用`th:inline="text/javascript/none"`激活，`th:inline`可以在父级标签内使用，甚至作为 body 的标签。内联文本尽管比`th:text`的代码少，不利于原型显示。

```
<script th:inline="javascript">
/*<![CDATA[*/
...
var username = /*[[${sesion.user.name}]]*/ 'Sebastian';
var size = /*[[${size}]]*/ 0;
...
/*]]>*/
</script>
```

js 附加代码：

```
/*[+
var msg = 'This is a working application';
+]*/
```

js 移除代码：

```
/*[- */
var msg = 'This is a non-working template';
/* -]*/
```

### 6、内嵌变量

为了模板更加易用，Thymeleaf 还提供了一系列 Utility 对象（内置于 Context 中），可以通过 # 直接访问：

- dates ： *java.util.Date的功能方法类。*
- calendars : *类似#dates，面向java.util.Calendar*
- numbers : *格式化数字的功能方法类*
- strings : *字符串对象的功能类，contains,startWiths,prepending/appending等等。*
- objects: *对objects的功能类操作。*
- bools: *对布尔值求值的功能方法。*
- arrays：*对数组的功能类方法。*
- lists: *对lists功能类方法*
- sets
- maps
  …

下面用一段代码来举例一些常用的方法：

#### dates

```
/*
 * Format date with the specified pattern
 * Also works with arrays, lists or sets
 */
${#dates.format(date, 'dd/MMM/yyyy HH:mm')}
${#dates.arrayFormat(datesArray, 'dd/MMM/yyyy HH:mm')}
${#dates.listFormat(datesList, 'dd/MMM/yyyy HH:mm')}
${#dates.setFormat(datesSet, 'dd/MMM/yyyy HH:mm')}

/*
 * Create a date (java.util.Date) object for the current date and time
 */
${#dates.createNow()}

/*
 * Create a date (java.util.Date) object for the current date (time set to 00:00)
 */
${#dates.createToday()}
```

#### strings

```
/*
 * Check whether a String is empty (or null). Performs a trim() operation before check
 * Also works with arrays, lists or sets
 */
${#strings.isEmpty(name)}
${#strings.arrayIsEmpty(nameArr)}
${#strings.listIsEmpty(nameList)}
${#strings.setIsEmpty(nameSet)}

/*
 * Check whether a String starts or ends with a fragment
 * Also works with arrays, lists or sets
 */
${#strings.startsWith(name,'Don')}                  // also array*, list* and set*
${#strings.endsWith(name,endingFragment)}           // also array*, list* and set*

/*
 * Compute length
 * Also works with arrays, lists or sets
 */
${#strings.length(str)}

/*
 * Null-safe comparison and concatenation
 */
${#strings.equals(str)}
${#strings.equalsIgnoreCase(str)}
${#strings.concat(str)}
${#strings.concatReplaceNulls(str)}

/*
 * Random
 */
${#strings.randomAlphanumeric(count)}
```

## 六、使用基本步骤

我认为可以大致分为四步：

1. pom.xml 添加 Thymeleaf 模板引擎。
2. application.yml 配置 Thymeleaf 信息。
3. 创建controller类，编写代码。
4. 创建模板，编写html代码。

#### 1. pom.xml 添加依赖

```
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

#### 2. .properties 配置 Thymeleaf 信息

```yaml
server:
  port: 8081
spring:
  thymeleaf:
    # 是否启用
    enabled: true
    # 模板编码
    encoding: UTF-8
    # 模板模式
    mode: HTML5
    # 模板存放路径
    prefix: classpath:/templates/
    # 模板后缀
    suffix: .html
    # 启用缓存，建议生产开启
    cache: false
    # 校验模板是否存在
    check-template-location: true
    # Content-type值
    servlet:
      content-type: text/html
  # 加配置静态资源
  resources:
    static-locations: classpath:/

```

#### 3. 创建controller类，编写代码

```
@RequestMapping("/me")
public String kownMe(Map<String,Object> map) {
    List<String> list = new ArrayList<String>();
    list.add("鸟不拉诗：一个正在努力Coding的未来架构师");
    list.add("记录菜鸟的成长");
    list.add("个人博客：https://niaobulashi.com");
    list.add("github博客：https://niaobulashi.github.io");
    map.put("msg", "Yoyoyoyoyo");
    map.put("images", "Yoyoyoyoyo");
    map.put("lists", list);
    return "me";
}
```

注意：返回的”me″是我HTML文件 me.html的名称哦

#### 4. 创建 page1.html 。编写html代码

只要把写好的HTML页面放在 classpath:/templates/ 下，thymeleaf就能自动渲染。

注意导入：

```
<html lang="en" xmlns:th="http://www.thymeleaf.org">
```

否则没提示哦~

```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" >
<head>
    <meta charset="UTF-8">
    <meta content="text/html;charset=UTF-8"/>
    <title>鸟不拉诗</title>
    <link rel="stylesheet" href="static/layui/css/layui.css">
</head>
<body>
<div class="layui-container">
    <h1>templates示例</h1>
    <!-- th:text="" 将h2的文本值视为指定参数 -->
    <hr>
    <h2 th:text="${msg}">这是h2</h2>
    <fieldset class="layui-elem-field layui-field-title" style="margin-top: 20px;">
        <legend>循环</legend>
    </fieldset>
    <table style="text-align: center" class="layui-table" lay-skin="line"  >
        <colgroup>
            <col width="350">
        </colgroup>
        <thead>
        <tr>
            <th>NAME</th>
        </tr>
        </thead>
        <tbody>
        <!-- th循环遍历传来的参数 -->
        <tr th:each="str : ${lists}" >
            <th th:text="${str}" ></th>
        </tr>
        </tbody>
    </table>
    <img th:src="${images}" >
</div>
</body>
</html>
```

运行效果（样式用的layUI~~）：

![](https://niaobulashi.github.io/assets/images/2019/springboot/thymeleaf-05-01.png)

## 七、参考

[Thymeleaf 使用详解](http://www.ityouknow.com/springboot/2016/05/01/spring-boot-thymeleaf.html)

[SpringBoot中的Thymeleaf 模板引擎](https://lzyz.fun/thymeleaf/)

[Thymeleaf官方文档](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html)