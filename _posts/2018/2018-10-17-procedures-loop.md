---
layout: post
title: 存储过程关于LOOP循环问题
category: java
tags: [java]
copyright: java
---

>###**存储过程LOOP疑问**
---
今天在开发需求时，需要编写一个存储过程，实现数据的初始化功能。

发现遇到循环处理时，跳出循环的条件，Loop的处理


```
LOOP EXIT WHRN(P1 > P2);
    ...
END LOOP;
```

emmmmmm，因为之前没有怎么学习存储过程，只是看了一下存储过程的原理，就上手开发了（所谓了敏捷开发哈哈哈哈哈哈）

就引起我的疑问：是直接进入循环呢？还是先判断条件？

我上网查阅资料。。。好像没有查阅到额

不过看到有一个类型的，我贴出来

```
loop
    v_sal := v_sal + 1;
    dbms_output.put_line(v_sal);
    exit when v_sal = 8000;
end loop
```

这个类比Java的do-while形式是一样的：先进入循环，满足条件了跳出循环

我就猜测

第一种也是这种类型的

于是我写了个测试例子，存储过程代码如下：

```
CREATE OR REPLACE PROCEDURE P_TEST(v_res OUT NUMBER,
                                   v_errorCode  OUT NVARCHAR2,
                                   v_errorMsg   OUT NVARCHAR2,
                                   c_startDate  IN NVARCHAR2) IS

c_firstMonthDay NVARCHAR2(10);     -- 所在月的第一天
c_firstClearDate NVARCHAR2(10);    -- 第一次生成的披露日期
c_clearDate NVARCHAR2(10);         -- 披露日期
c_N NUMBER(10):=1;                 -- 计算使用的倍数，从0开始


BEGIN
  --================================================================================
  -------------------------------【执行sql文】--------------------------------------
  --================================================================================
  
/* 期间管理报告清算提示 Start */

  -- 开启日志输出缓冲
  DBMS_OUTPUT.ENABLE(buffer_size => null);
  -- 获取当月的第一天和最后一天
  SELECT to_char(last_day(to_date(substr(c_startDate,0,7)||'-01', 'yyyy-mm-dd')), 'yyyy-mm-dd') INTO c_firstMonthDay FROM dual;
  --SELECT to_char(to_date(substr(c_startDate,0,7)||'-01', 'yyyy-mm-dd'), 'yyyy-mm-dd') INTO c_lastMonthDay FROM dual;
  c_N := 1;
  
  -- 生成第一次提示日期
   SELECT extract(YEAR FROM to_date(c_startDate, 'yyyy-mm-dd'))||'-'||'01'||'-'||'01'
         INTO c_firstClearDate FROM dual;
   c_clearDate := c_firstClearDate;
   
   -- 披露日期小于系统日期生成期间管理报告清算提示
   LOOP EXIT WHEN (c_clearDate > c_startDate);
        -- 直接生成收益分配提示
        dbms_output.put_line('披露日期为：' || c_clearDate);
        dbms_output.put_line('传入日期为：' || c_startDate);
        dbms_output.put_line('进入循环，说明首次进入循环不做条件判断');
        -- 用第一次提示日期计算下一次的提示日期
        SELECT to_char(add_months(to_date(c_firstClearDate, 'yyyy-mm-dd'), c_N*12), 'yyyy-MM-')||'01' INTO c_clearDate FROM dual;
        c_N := c_N + 1;
   END LOOP;

  
/* 期间管理报告清算提示 End */

  --输出放回状态信息
  v_res       := 0;
  v_errorCode := SQLCODE;
  v_errorMsg  := 'P_TEST' || ':' || TO_CHAR(SQLERRM);

  COMMIT;

--异常处理
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    v_res       := -1;
    v_errorCode := SQLCODE;
    v_errorMsg  := 'P_TEST' || ':' || TO_CHAR(SQLERRM);
END P_TEST; 
```

编写完成，通过测试代码：
```
begin
  -- Call the procedure
  p_test(v_res => :v_res,
         v_errorcode => :v_errorcode,
         v_errormsg => :v_errormsg,
         c_startdate => '2018-10-17');
end;
```
可以通过debug模式一步一步观察代码执行，

我把存储过程执行结果的日志贴出来

![QQ截图20181017100342.png][1]

通过打印的日志也可以得出结论：先进入循环，当满足条件，跳出结束循环~

就OK的啦~~~~

 ::aru:shy:: 




  [1]: https://niaobulashi.com/usr/uploads/2018/10/2971597046.png