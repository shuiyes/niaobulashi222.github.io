---
layout: post
title: linux 同时启动多个tomcat
category: linux
tags: [java, linux]
copyright: java, linux

---

# linux同时启动多个tomcat

今天下班学习nginx负载均衡时，需要多个tomcat端口服务，在一台机启动多个tomcat服务，于是记录下操作过程。



## 复制tomcat

原tomcat端口默认8080，复制出的tomcat端口8081。

![1578923845682](https://images.niaobulashi.com/typecho/uploads/2020/01/2432263281.png)

## 编辑环境变量

![1578923929147](https://images.niaobulashi.com/typecho/uploads/2020/01/2341475484.png)

贴出来

``` bash
#tomcat_8080
export CATALINA_HOME=/root/service/tomcat_8080
export CATALINA_BASE=/root/service/tomcat_8080
export TOMCAT_HOME=/root/service/tomcat_8080

#tomcat_8081
export CATALINA_HOME2=/root/service/tomcat_8081
export CATALINA_BASE2=/root/service/tomcat_8081
export TOMCAT_HOME2=/root/service/tomcat_8081
```

使配置文件生效

`source .bash_profile`

## 修改第二份catalina.sh

``` bash
cd /root/service/tomcat_8081/bin
```
添加

``` bash
export CATALINA_BASE=$CATALINA_BASE2
export CATALINA_HOME=$CATALINA_HOME2
```

![1578924182554](https://images.niaobulashi.com/typecho/uploads/2020/01/4081550225.png)

## 修改第二份修改server.xml

```
cd /root/service/tomcat_8081/conf
```

将三个端口从上到下，一次修改为`8006`，`8081`，`8010`

OK，配置完成

## 启动两个tomcat

![1578924369993](https://images.niaobulashi.com/typecho/uploads/2020/01/2367557515.png)

