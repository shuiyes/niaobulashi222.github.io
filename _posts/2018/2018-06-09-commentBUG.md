---
layout: post
title: 修复使用微信内置浏览器评论报错的BUG
category: java
tags: [java]
copyright: java
---

>昨晚在测试评论邮件通知的时候，我把http://niaobulashi.com/index.php/bbs.html 留言板发给朋友进行测试，用的微信发送的，所以朋友直接用的微信内置浏览器打开的链接进行评论
出现了下面问题

这是朋友的截图

![使用微信自带浏览器进行评论bug复现](/usr/image/article/Typecho/01/wechaterror.png)


```
[Sat Jun 09 09:26:31.057599 2018] [:error] [pid 23230] [client 123.139.18.5:25297] SQLSTATE[22001]: String data, right truncated: 1406 Data too long for column 'agent' at row 1, referer: http://niaobulashi.com/index.php/bbs.html
```

关键信息：1406 Data too long for column 'agent' at row 1

很明显微信内置的浏览器的user agent的字段过长导致的错误。
评论的表typecho_comments.agent

对了，我把Chrome浏览器的agent和微信内置浏览器的agent贴出来供参考:
Chrome浏览器agent：
```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36
```
微信内置浏览器：
```
Mozilla/5.0 (Linux; Android 8.1; ONEPLUS A5010 Build/OPM1.171019.011; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/57.0.2987.132 MQQBrowser/6.2 TBS/044103 Mobile Safari/537.36 MicroMessenger/6.6.7.1320(0x26060737) NetType/WIFI Language/zh_CN
```
将原来的长度为200改为400即可

修改之后，进行测试。
![修复bug测试前截图](/usr/image/article/Typecho/01/testbefore.png)

![修复bug测试后截图](/usr/image/article/Typecho/01/testafter.png)

OK~
问题解决
emmmm，就是将agent字段长度加长点...
于是我又水了一篇???

