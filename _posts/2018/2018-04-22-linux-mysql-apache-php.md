---
layout: post
title: Linux搭建mysql、apache、php服务总结
category: life
tags: [life]
copyright: life
---

##阿里云ESC实例配置
>对于新手，比如我，了解云服务无非就是阿里云和腾讯云，对于国外的服务器了解甚少。学生时代估计会有很实惠的打折优惠吧，反正我没遇到过（我大学的时候只知道玩，也没关注过这些~）。废话说了这么多，现在开始吧…
####实名认证
>阿里云进行实名认证之后，入口：产品弹性计算云服务器ECS，选择合适的参数配置规格点击购买即可。
####查看实例
>购买成功之后，可以从管理控制台云计算基础服务实例中查看。

![Alt text](/usr/image/article/bashProfile/01/aliyunManage.png)
####重启
>初始状态下点击更多，重置密码，该密码即为root密码，哦对了，设置完还需要重启，也在更多，点击重启即可，大概需要40秒吧。
####SecureCRT登录
>登陆服务，个人推荐使用SecureCRT进行管理配置开发，键入root密码，好，现在进入Linux命令行的世界啦~

##新建用户
>Root用户拥有决定的权限，主要用户安全软件服务，修改系统环境属性等，不利于开发使用，所以让我们先给自己新建一个用户吧。
####使用Root新建用户
>下列是root用户下键入的命令行：

```
[root@XXX ~]# adduser testuser		#创建用户testuser
[root@XXX ~]# passwd testuser		 #为用户testuser创建密码
```
>此时在/home目录下已经创建了一个用户目录testuser
```
[root@XXX ~]# userdel testuser		#删除用户testuser
[root@XXX ~]# rm –rf * testuser	   #删除用户testuser所在的目录
```
![Alt text](/usr/image/article/bashProfile/01/addUser.png)

注意各种密码要拿个别人看不到的小本本或者云笔记记着哦，找密码什么的最烦了。
##安装MySQL
>首先我都是先把数据库搭建好，个人偏爱mysql，navicat进行客户端管理，以下操作在root用户下进行。
####下载前准备
>下载之前先检查是否已经安装过mysql
![Alt text](/usr/image/article/bashProfile/01/checkInstallMySQL.png)
>无输出内容说明系统没有检测到安装过mysql。
>若检查到存在安装文件，则先卸载，卸载前先停止mysql服务。
```
[root@XXX ~]# service mysql status	#查看mysql服务启动状态
[root@XXX ~]# service mysql stop	  #停止mysql服务
```
>卸载之前的版本
```
[root@XXX ~]# rpm –qa|grep –i mysql
[root@XXX ~]# rpm –e xxxx[之前安装的版本] --nodeps	#卸载mysql版本
```
####下载
>直接使用yum命令下载mysql8.0来进行安装，安装过程会有问题，这里我们需要使用rpm命令先来进行下载。下载路径为：http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
>在Linux中下载命令为：
```
[root@XXX ~]# rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```
![Alt text](/usr/image/article/bashProfile/01/downlaodMySQLURL.png)
>下载完成之后查看一下都有哪些mysql的文件，使用命令：
```
[root@XXX ~]# yum repolist enabled | grep "mysql.*-community.*"
```
>效果如图：
![Alt text](/usr/image/article/bashProfile/01/queryMySQLfiles.png)
####安装
>下面就可以使用yum进行安装。输入命令如下：
```
[root@XXX ~]# yum install mysql-community-server
```
>效果如图
![Alt text](/usr/image/article/bashProfile/01/installMySQL01.png)
>安装过程中会提示安装一些相关的软件，这里点击”y”回车就可以了，如图：
![Alt text](/usr/image/article/bashProfile/01/installMySQL02.png)
>还有一个文件需要安装，继续点击”y”回车，如图：
![Alt text](/usr/image/article/bashProfile/01/installMySQL03.png)
>这样mysql就安装成功啦。
####基础信息配置
>在配置信息之前，我们先去阿里云实例进行安全组配置，开放3306mysql服务端口。
![Alt text](/usr/image/article/bashProfile/01/aliyunManageSecurity.png)
>不过还没有结束，还需要进行一些配置信息哦。
>首先将mysql服务启动，开启mysql的进程，使用命令：
>service mysqld start，效果图如下：
![Alt text](/usr/image/article/bashProfile/01/mysqldStart.png)
>查看mysql服务进程信息，使用命令：
>service mysqld status，效果图如下：
![Alt text](/usr/image/article/bashProfile/01/mysqldStatus.png)
>Mysql服务启动之后，还选哟一些基本信息的配置。输入设置命令：
```
[root@XXX ~]# mysql_secure_installation
```
>效果如图：
![Alt text](/usr/image/article/bashProfile/01/mysqlSecureInstallation.png)
>这里需要注意一下：
>初次安装时，只需要回车即可，如果以前安装过，这里会提示需要输入root密码，键入root密码回车。这点请稍微注意一下。。
>下列几处需要设置的地方如图：
![Alt text](/usr/image/article/bashProfile/01/mysqlSecureInstallationSettings.png)
>登陆mysql，命令如下：
```
[root@XXX ~]# mysql –u root –p
```
>输入数据库root的密码回车，如下图：
![Alt text](/usr/image/article/bashProfile/01/loginMySQL.png)
>Mysql就正式安装设置完毕啦，是不是so easy!
####Navicat连接数据库
>很显然命令行方式很不适合开发使用，可视化也不强，个人推荐使用Navicat Premium数据库连接工具连接mysql数据库，方便！
>输入连接信息，如图下：
![Alt text](/usr/image/article/bashProfile/01/checkNavicatLoginMysql.png)
>点击连接测试，如图下：
![Alt text](/usr/image/article/bashProfile/01/mysqlError1130.png)
>会出现”Host is not allowed to connect to this MySQL server”
>如何解决这个问题呢？很明显这是不允许远程登录，只能在localhost主机进行登录。所以需要授权，命令方法如下：
```
[root@XXX ~]# mysql –u root -p
Enter password: 
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
mysql> QUIT;
```
>如下图所示：
![Alt text](/usr/image/article/bashProfile/01/solveMysqlError1130.png)
>回来再点击“连接测试”，会出现下图报错：
![Alt text](/usr/image/article/bashProfile/01/mysqlError1045.png)
>不着急，现在就解决这个问题。
>在SecureCRT中登录mysql，查询mysql.user表信息，将密码为空的数据删除掉即可，删除空账号和空密码的数据，命令如下：
```
[root@XXX ~]# mysql –u root –p
Enter password: 
mysql> use mysql;
mysql> select Host,User,Password from user;
```
![Alt text](/usr/image/article/bashProfile/01/selectMysqlUser.png)
>删除User和Password为空的数据，放心，可以删的，命令如下：
>delete from mysql.user where Password=’’;
>在刚刚的mysql_secure_installation中我们已经配置了root的登录密码，在这里我们也还可以设置root密码，命令如下：
>Update mysql.user set password=password('root密码') where Host='%';
>最后请一定要做的操作是：刷新权限
>Flush privileges;
>退出：quit;
>再回来连接测试，如下图：
![Alt text](/usr/image/article/bashProfile/01/checkNavicatLoginMysqlSuccess.png)
>Mysql现在是彻底弄好啦。请尽情的增删改查吧骚年~
####解决中文乱码
>进入目录
```
[root@XXX ~]# cd /usr/share/mysql
[root@XXX ~]# vi my-default.cnf
```
>添加如下配置信息：
![Alt text](/usr/image/article/bashProfile/01/mysqlUTF8.png)
>重启mysql服务
```
[root@XXX ~]# service mysqld restart
```
##安装Apache
####检查、删除、安装
```
[root@XXX ~]# rpm –qa|grep httpd     #检查是否安装apache
[root@XXX ~]# rpm –e 包名 –nodeps    #若有则删除
```
![Alt text](/usr/image/article/bashProfile/01/checkInstallApache.png)
```
[root@XXX ~]# yum install httpd	 #安装，根据提示，输入Y即可
```
![Alt text](/usr/image/article/bashProfile/01/installApache01.png)
>需要确认安装一些组件，输入Y即可：
![Alt text](/usr/image/article/bashProfile/01/installApache02.png)
####启动、测试
>启动命令如下：
```
[root@XXX ~]# service httpd start
```
![Alt text](/usr/image/article/bashProfile/01/httpdStart.png)
>查看apache服务停启情况如下：
```
[root@XXX ~]# Service httpd status
```
![Alt text](/usr/image/article/bashProfile/01/httpdStatus.png)
>此时需要注意一点的是，安全组规则需要添加端口80的安全组：
![Alt text](/usr/image/article/bashProfile/01/aliyunManageSecurity01.png)
>在浏览器中输入服ip，如下表示apache安装成功：
![Alt text](/usr/image/article/bashProfile/01/httpdTest.png)
##安装PHP
####安装PHP
```
[root@XXX ~]#yum install php
```
![Alt text](/usr/image/article/bashProfile/01/installPHP01.png)
>输入”y”回车
![Alt text](/usr/image/article/bashProfile/01/installPHP02.png)
>安装成功
>安装组件，支持mysql
```
yum install php-mysql php-gd libjpeg* php-imap php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-mcrypt php-bcmath php-mhash libmcrypt
```
>根据提示，输入Y即可
####启动HTTPD
>重启httpd
```
service httpd restart
```
在浏览器中访问ip
![Alt text](/usr/image/article/bashProfile/01/phpTest.png)
>OK啦
##参考文章
[0][【PHP】linux搭建PHP运行环境](https://www.cnblogs.com/zhaoxd07/p/5580126.html)