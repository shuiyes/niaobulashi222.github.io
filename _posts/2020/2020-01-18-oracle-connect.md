---
layout: post
title: 解决Oracle服务端1521端口无法telnet，服务名未开启监听问题
category: java
tags: [java]
copyright: java

---

> **场景：oracle服务安装在windows本地，oracle客户端在虚拟机中，使用虚拟机连接windows的oracle数据库。**

## 问题1：使用虚拟机telnet我本地oracle1521端口，不通

解决思路：

- 关闭虚拟机linux防火墙（这个方法有点粗暴，建议在只需要修改防火墙的端口允许通过即可）

  ``` bash
  # 1:查看防火状态
  systemctl status firewalld
  service  iptables status
  # 2:暂时关闭防火墙
  systemctl stop firewalld
  service  iptables stop
  # 3:永久关闭防火墙
  systemctl disable firewalld
  chkconfig iptables off
  # 4:重启防火墙
  systemctl enable firewalld
  service iptables restart  
  # 5:永久关闭后重启
  chkconfig iptables on
  ```

- 防火墙配置规则 端口 允许得端口

  ```bash
  # 查看已打开的端口
  netstat -anp
  # 添加允许的端口
  firewall-cmd --add-port=1521/tcp --permanent
  # 若移除端口
  firewall-cmd --permanent --remove-port=1521/tcp
  # 策略修改完成，请重启： 
  systemctl restart firewalld
  ```

- 添加windows防火墙对1521的入站允许规则

  ![1579253672708](https://images.niaobulashi.com/typecho/uploads/2020/01/2275375255.png)

  

## 问题2：使用sqlplus登录报错，ORA-12514: TNS: 监听程序当前无法识别连接描述符中请求的服务

解决思路：关键字**`监听程序`**

- 查看监听服务状态

  ```bash
  # 关闭监听服务
  lsnrctl stop
  # 启动监听服务
  lsnrctl start
  # 查看监听服务状态
  lsnrctl stat
  ```

  查看监听服务如果出现下列问题

  ![1579254533200](https://images.niaobulashi.com/typecho/uploads/2020/01/1004579222.png)

  说明监听服务没有启动

  去启动oracle监听服务，监听服务有两个，这里只做单监听讲，随便启动一个即可。

  ![1579254625812](https://images.niaobulashi.com/typecho/uploads/2020/01/3169635404.png)

  再通过`lsnrctl stat`查看监听服务，如果出现下图情况

  ![1579254724154](https://images.niaobulashi.com/typecho/uploads/2020/01/1436926014.png)

  只看到一个服务名"CLRExtProc"启动了，而我们想要的是ORCL服务名

  这是需要修改`listener.ora` 文件

- 修改`listener.ora` 文件

  文件路径，我本地的路径是：D:\app\niaobulashi\product\11.2.0\dbhome_1\NETWORK\ADMIN

  需要添加以下<font color=red size=4>红色部分代码</font>，将服务名为ORCL添加到监听配置文件中

  ![1579254216172](https://images.niaobulashi.com/typecho/uploads/2020/01/108328150.png)

  贴出来如下：

  ``` java
  SID_LIST_LISTENER =
    (SID_LIST =
      (SID_DESC =
        (SID_NAME = CLRExtProc)
        (ORACLE_HOME = D:\app\niaobulashi\product\11.2.0\dbhome_1)
        (PROGRAM = extproc)
        (ENVS = "EXTPROC_DLLS=ONLY:D:\app\niaobulashi\product\11.2.0\dbhome_1\bin\oraclr11.dll")
      )
  	(SID_DESC=
  	  (SID_NAME = ORCL)
        (ORACLE_HOME = D:\app\niaobulashi\product\11.2.0\dbhome_1)
        (PROGRAM = extproc)
        (ENVS = "EXTPROC_DLLS=ONLY:D:\app\niaobulashi\product\11.2.0\dbhome_1\bin\oraclr11.dll")
      )
    )
  
  LISTENER =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = DESKTOP-NNDNCEI)(PORT = 1521))
    )
  ADR_BASE_LISTENER = D:\app\niaobulashi
  ```

  再查看监听服务状态，可以看到ORCL有了

  ![1579254808269](https://images.niaobulashi.com/typecho/uploads/2020/01/1361715811.png)

- 修改`tnsname.ora`的`HOST`为本地主机名

  ``` bash
  ORACLR_CONNECTION_DATA =
    (DESCRIPTION =
      (ADDRESS_LIST =
        (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
      )
      (CONNECT_DATA =
        (SID = CLRExtProc)
        (PRESENTATION = RO)
      )
    )
  
  LISTENER_ORCL =
    (ADDRESS = (PROTOCOL = TCP)(HOST = DESKTOP-NNDNCEI)(PORT = 1521))
  
  ORCL =
    (DESCRIPTION =
      (ADDRESS_LIST =
        (ADDRESS = (PROTOCOL = TCP)(HOST = DESKTOP-NNDNCEI)(PORT = 1521))
      )
      (CONNECT_DATA =
        (SERVICE_NAME = ORCL)
      )
    )
  ```

最后使用虚拟机就可以正常连接本地oracle服务了

![1579254894242](https://images.niaobulashi.com/typecho/uploads/2020/01/1794099600.png)