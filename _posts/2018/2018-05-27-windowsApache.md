---
layout: post
title: windows系统下安装和移除apache服务
category: java
tags: [java]
copyright: java
---

用windows系统本地搭建apache，有时会遇到糟心的问题，想卸载了重装，这时已经将服务挂载在windows服务列表中，于是需要将服务卸载掉，这里先安装在卸载吧


## 安装
直接放出官网链接：[windows版本apache下载地址][1]
选择相应的版本下载即可
解压到目标目录下
使用管理员进入cmd命令窗口模式下
cd apache的bin目录下

    cd c:\Apache24\bin
    .\httpd.exe -k install -n apache
提示：
Installing the 'apache' service 
The 'apache' service is successfully installed.

查看windows服务:
进入win + R，键入services.msc
此时会找到服务名称为：apache的服务
右键启动即可

## 卸载
同理安装一样。。。
命令不一样罢了

    cd c:\Apache24\bin
    .\httpd.exe -k uninstall -n apache
进入windows服务列表可以看到apache已移除

哦啦~

  [1]: https://www.apachehaus.com/cgi-bin/download.plx