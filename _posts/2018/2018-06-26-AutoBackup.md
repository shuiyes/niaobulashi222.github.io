---
layout: post
title: Linux下mysql自动备份脚本
category: life
tags: [life]
copyright: life
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;逛了蛮多博客网站的，亲眼看到一个博客网站数据丢失之后的模样，挺为他心痛的。于是就打算弄个mysql定时备份的脚本，可以自行设计crontab定时执行时间，可以是周一和周四每周备份两次就可以了。
##脚本
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;新建一个mysql_backup.sh文件

```
#!/bin/bash
#设置mysql的备份保存目录
folder=/root/mysql_back
cd $folder
day=`date +%Y%m%d`
mkdir -p "$day"
cd $day
#数据库服务器ip，填写服务器的公网地址即可
host=xx.xx.xx.xx
#用户名
user=root
#密码
password=password
#要备份的数据库名
db=test

#执行备份语句
mysqldump -h$host -u$user -p$password $db > ./test.sql
```
***
##注意
上方需要你 **修改** 并且 **注意** 的地方有：
 - folder设置的目录你可以自行设置
 - day=date +%Y%m%d，"+"和"date"必须有个空格，否则会有语法错误
 - host为你的服务器公网IP
 - user一般都是root
 - password为root的密码
 - db为博客的数据库名

测试脚本是否正确，执行脚本
```
sh mysql_backup.sh
```

![QQ截图20180626230755.png][1]
 
![QQ截图20180626230835.png][2]

因为我是为了测试crontab的定时任务执行是否有效，我设置的是1分钟执行一次，其中day=\`date +%Y%m%d_%H%M%S\`。（为了执行效果而截的图，可忽略）

![QQ截图20180626231009.png][3]

##定时任务

设置好定时任务crontab执行时间，一般ESC都会自带crontab服务的。查看crontab服务状态

>service crond status

![QQ截图20180626225857.png][4]

有蓝色指示灯说明服务运行正常，OK，开始设置定时任务
```
crontab -e
```
键入i，进入编辑模式：
敲入下列命令：每周1和周4凌晨2点会执行定时脚本
```
0 2 * * 1,4 /etc/profile;/bin/sh /root/mysqlbackup.sh
```
重启crontab服务使之生效
```
/bin/systemctl restart crond.service
```
OK了，之后查看备份的文件就在上面脚本定义的目录上查看即可
```
cd /root/mysql_back
```
为你的博客进行备份，不再为数据丢失而烦恼啦。

是不是so easy。有啥问题尽情留言，秒回

>20180725前来更新
这是部署脚本之后的执行效果，每个周一和周四凌晨2点执行的效果图

![微信截图_20180725140456.png][5]

不再为忘记备份担心数据丢失啦~

##推荐阅读
[关于定时执行任务：Crontab的20个例子][6]
[Linux crontab定时执行任务 命令格式与详细例子][7]
[Linux crontab命令][8]


  [1]: https://niaobulashi.com/usr/uploads/2018/06/1464004124.png
  [2]: https://niaobulashi.com/usr/uploads/2018/06/2870057410.png
  [3]: https://niaobulashi.com/usr/uploads/2018/06/2495043674.png
  [4]: https://niaobulashi.com/usr/uploads/2018/06/4180787049.png
  [5]: https://niaobulashi.com/usr/uploads/2018/07/1640153408.png
  [6]: https://www.jianshu.com/p/d93e2b177814
  [7]: https://www.jb51.net/LINUXjishu/19905.html
  [8]: http://www.runoob.com/linux/linux-comm-crontab.html