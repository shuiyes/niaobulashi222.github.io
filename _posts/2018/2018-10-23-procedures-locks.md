---
layout: post
title: 存储过程关于LOOP循环问题
category: oracle
tags: [java, oracle]
copyright: java, oracle
---


##存储过程
先科普一下存储过程，在项目开发过程可能会遇到。

>存储过程（Stored Procedure）是在大型数据库系统中，一组为了完成特定功能的SQL 语句集，存储在数据库中，经过第一次编译后再次调用不需要再次编译，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。 存储过程是数据库中的一个重要对象。

----------


今天开发过程中，遇到之前写好的存储过程重新进行编译出现**锁死**的情况，于是我瞬间不淡定了。明明之前都已经写好了呀！

在数据库开发的过程中，经常碰到包、存储过程、函数无法编译或编译时导致PL/SQL无法响应的问题。碰到这种问题，基本上都要重启数据库解决，严重浪费开发时间。

##问题分析

从事数据库开发的都知道锁的概念，如：执行 Update Table xxx Where xxx 的时候就会产生锁。这种常见的锁在Oracle里面被称为DML锁。在Oracle中还有一种DDL锁，主要用来保证存储过程、表结构、视图、包等数据库对象的完整性，这种锁的信息可以在DBA_DDL_LOCKS中查到。注意：V$LOCKED_OBJECT记录的是DML锁信息，DDL锁的信息不在里面。

对应DDL锁的是DDL语句，DDL语句全称数据定义语句（Data Define Language）。用于定义数据的结构或Schema，如：CREATE、ALTER、DROP、TRUNCATE、COMMENT、RENAME。当我们在执行某个存储过程、或者编译它的时候Oracle会自动给这个对象加上DDL锁，同时也会对这个存储过程所引用的对象加锁。
了解了以上知识以后，我们可以得出结论：编译包长时间无响应说明产生了死锁。我们可以轻易的让这种死锁发生，举例：

    打开一个PL/SQL，开始调试某个函数（假设为：FUN_CORE_SERVICECALL），并保持在调试状态


    打开一个SQL Window，输入Select *From dba_ddl_locks aWhere a.name ='FUN_CORE_SERVICECALL'会发现一行记录：


    打开一个新的PL/SQL，重新编译这个函数。我们会发现此时已经无法响应了


    回到第一个PL/SQL ,重新执行Select *From dba_ddl_locks aWhere a.name ='FUN_CORE_SERVICECALL'我们将会看到如下记录：
上述的情况表明发生了锁等待的情况。


在Oracle中DDL锁分为：Exclusive DDL Locks（排他的DDL）、Share DDL Locks（共享DDL锁）、Breakable Parse Locks（可被打破的解析锁）几类。篇幅所限，这里就不再详细介绍了。根据这个例子推理一下，当我们试图编译、修改存储过程、函数、包等对数据对象的时候，如果别人也正在编译或修改他们时就会产生锁等待；或者我们在编译某个存储过程的时候，如果它所引用的数据库对象正在被修改应该也会产生锁等待。这种假设有兴趣的兄弟可以测试下，不过比较困难。


##解决方案

碰到这种问题，如果知道是被谁锁定了（可以查出来的），可以让对方尽快把锁释放掉；实在查不出来只能手工将这个锁杀掉了。

**死锁是数据库经常发生的问题，数据库一般不会无缘无故产生死锁，死锁通常都是由于我们应用程序的设计本身造成的。**

产生死锁时，如何解决呢，下面是常规的解决办法（区分是否知道谁被锁定）：
 - 知道死锁对象
 - 不知道死锁对象

####知道死锁对象
  
比如我所遇到的问题，P_REVEAL_REPORT_CLEAR_TIP编译响应失败。

 - 查询V$DB_OBJECT_CACHE


    SELECT * FROM V$DB_OBJECT_CACHE WHERE name='P_REVEAL_REPORT_CLEAR_TIP' AND LOCKS!='0';

*注意：P_REVEAL_REPORT_CLEAR_TIP为存储过程的名称。发现locks＝6，说明有6个死锁sid*

 - 按对象查出sid的值


    select /*+ rule*/  SID from V$ACCESS WHERE object='P_REVEAL_REPORT_CLEAR_TIP';

*注意：CRM_LASTCHGINFO_DAY为存储过程的名称。查询出对应6个的sid*

 - 查sid,serial#


    SELECT SID,SERIAL#,PADDR FROM V$SESSION WHERE SID='刚才查到的SID';

 - 清除session


    alter system kill session 'sid值,serial#值' immediate;

####不知道死锁对象

 -  执行下面SQL，先查看哪些表被锁住了


    select b.owner,b.object_name,a.session_id,a.locked_mode from v$locked_object a,dba_objects b where b.object_id = a.object_id;

 - 查处引起死锁的会话


    select b.username,b.sid,b.serial#,logon_time from v$locked_object a,v$session b where a.session_id = b.sid order by b.logon_time;
*这里会列出SID*

 - 查出SID和SERIAL#


    SELECT SID,SERIAL#,PADDR FROM V$SESSION WHERE SID='刚才查到的SID';

*查V$SESSION视图，这一步将得到PADDR*


再次执行存储过程，错误没有了。语句执行成功！
