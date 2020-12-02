---
layout: post
title: Linux中常用Shell命令
category: java
tags: [java]
copyright: java
---

作为项目工程师，接触Linux还是蛮多的，总结下常用的Shell命令

##打包与解压
项目上需要打包或者解压，常常用于备份或者上版，而打包常见格式为tar包、tar.gz包、war包这三种
```
打.tar包：tar -cvf backup_20180504.tar ./etc ./src
.tar包打.gz包：gzip backup_20180504.tar

解压.tar包：tar -xvf backup_20180504.tar
解压.tar.gz包：tar -xzvf backup_20180504.tar.gz

打.war包：jar -cvf backup_20180504.war ./etc ./src
解压.war包：jar -xvf backup_20180504.war

查看.tar文件内容：tar -tvf backup_20180504.tar
查看.gz文件内容：tar -tvzf backup_20180504.tar.gz
```

##查看系统性能
主要查看Linux系统磁盘、内存、服务进行占用空间等信息，详细就不多说
```
查看系统负载：df
性能分析：top
查看进程：ps -ef|grep 进程名称
查看主机ip：ifconfig
查看主机域名：hostname
```

##比较大小
```
-ne 不等于
-gt 大于
-ge 大于等于
-lt 小于
-le 小于等于
-eq 等于 
```

##文件操作
```
移动文件：mv test1 相对路径或者绝对路径
替换文件名称：mv test1 test2
动态查看tomcat日志：tail -f catalina.log
复制文件：cp test1 test2
查看文件：cat test1
编辑文件：vi test1    启动编辑模式：i
保存退出：:wq
检查脚本语法：sh -n test.sh
```

##查看端口服务
```
查看启用端口：netstat -lntp
查看启动服务：systemctl list-unit-files|grep enabled
搜索端口：netstat -aon|findstr "8080"
```

##文件与用户所有者权限变更
首先要名称文件的权限：读、写、执行

![Alt text](/usr/image/article/developmentSkills/Shell/fileSafety.png)
```
其中： 最前面那个 - 代表的是类型  
   中间那三个 rw- 代表的是所有者（user）  
   然后那三个 rw- 代表的是组群（group）  
   最后那三个 r--    代表的是其他人（other）  
  
然后我再解释一下后面那9位数：  
   r 表示文件可以被读（read）  
   w 表示文件可以被写（write）  
   x 表示文件可以被执行（如果它是程序的话）  
   - 表示相应的权限还没有被授予  
```
修改文件权限控制
```
chmod -R 700 ./test
chmod -R 500 ./test
```

持续更新...