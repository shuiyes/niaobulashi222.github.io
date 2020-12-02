---
layout: post
title: junit测试类防止事务回滚-工作心得
category: java
tags: [java]
copyright: java
---

在编写测试类时，调用service层，存在数据库操作

需要实现数据库的新增或者修改。

不添加关键注解的话，会出现下列的日志报告

可以看到关键日志部分：Rolled back transaction for test

出现了回滚操作

![微信截图_20181206094409.png][1]


这时如果需要在测试类中修改数据，就要添加注解，防止自动回滚

    @Rollback(false)

添加位置为类名上方

添加了返回自动回滚注解之后，看下打印的日志

![微信截图_20181206094745.png][2]


    Committed transaction for test

说明我们的sql已经commit了。实现数据库的变更。

哦啦~

![QQ图片20181206083603.gif][3]


  [1]: https://niaobulashi.com/usr/uploads/2018/12/2931689383.png
  [2]: https://niaobulashi.com/usr/uploads/2018/12/3909371742.png
  [3]: https://niaobulashi.com/usr/uploads/2018/12/1587088613.gif