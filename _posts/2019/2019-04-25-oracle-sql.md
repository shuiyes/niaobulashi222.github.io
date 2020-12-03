---
layout: post
title: Oracle常用SQL语句工作总结（持续更新中）
category: oracle
tags: [java, oracle]
copyright: java, oracle
---


主要是记录工作中遇到的一些各种'常用'和'变态'的SQL语句(๑•̀ㅁ•́ฅ)

---
## 查询
#### 1.统计成功失败总数
```
select sum(正确数)+sum(错误数) as 总记录数,sum(正确数),sum(错误数)
from (
    select count(1) 正确数,0 错误数
    from tb
    where status=1
union all 
    select 0 正确数,count(1) 错误数
    from tb
    where status=0) a;
```
#### 2.相同的id，取最新一条
```
// 例子
select *
  from (select row_number() over(partition by id order by time desc) rn, a.*
           from table a)
 where rn = 1;

// 实际使用
select 
       temp.c_notice_record_id as "noticeRecordId",
       temp.c_notice_task_id as "noticeTaskId",
       temp.c_receiver_id as "receiverId",
       temp.c_notice_result as "noticeResult",
       temp.create_time as "createTime"
  from (select t.c_notice_record_id,
               t.c_notice_task_id,
               t.c_receiver_id,
               t.c_notice_result,
               t.create_time,
               row_number() OVER(PARTITION BY t.c_receiver_id order by t.create_time desc) as row_flg
          from t_notice_record t) temp
 where temp.row_flg = '1';
```
#### 3.相同ID 合并字段，页面换行显示
```
select t.project_code,
       listagg(to_char(t.sqmx), '<br>') within GROUP(order by t.project_code) as sqmx
  from (select trrs.project_code,
               trrs.reveal_rate as sqmx
          from t_reveal_report_scheme trrs
         where 1 = 1
           and trrs.delete_flag = '0') t
 group by t.project_code;
```
如图所示：
![请输入图片描述][1]

#### 4.根据表table_b的name去重查询的结果集查询表table_a
```
select a.*
  from table_a a
 where a.name in (select distinct tc.name
                            from table_b b
                           where b.delete_flag = '0')
   and a.delete_flag = '0';
```

#### 5.使用BETWEEN_DATE_YMD
```
Date[] betweenDate = new Date[2];
betweenDate[0] = ToolUtil.formateDate(startDate, "yyyy-MM-dd");
betweenDate[1] = ToolUtil.formateDate(endDate, "yyyy-MM-dd");
investWhereProp.add(new ColumnBase("settlementDate", betweenDate, ColumnBase.BETWEEN_DATE_YMD));
```
#### 6.查询临时表SQL
```
SELECT TU.TABLESPACE_NAME                                    AS "TABLESPACE_NAME",
       TT.TOTAL - TU.USED                                    AS "FREE(G)",
       TT.TOTAL                                              AS "TOTAL(G)",
       ROUND(NVL(TU.USED, 0) / TT.TOTAL * 100, 3)            AS "USED(%)",
       ROUND(NVL(TT.TOTAL - TU.USED, 0) * 100 / TT.TOTAL, 3) AS "FREE(%)"
FROM (SELECT TABLESPACE_NAME, 
              SUM(BYTES_USED) / 1024 / 1024 / 1024 USED
       FROM V$TEMP_SPACE_HEADER
       GROUP BY TABLESPACE_NAME) TU ,
     (SELECT TABLESPACE_NAME,
              SUM(BYTES) / 1024 / 1024 / 1024 AS TOTAL
       FROM DBA_TEMP_FILES
       GROUP BY TABLESPACE_NAME) TT
WHERE TU.TABLESPACE_NAME = TT.TABLESPACE_NAME;
```

## 更新
#### 1.Oracle两表关联（join）更新字段值一张表到另一张表

```
update (select a.name aname, b.name bname 
       from A a, B b where a.id = b.id)
   set aname = bname;
//注：两表关联属性id必须为unique index或primary key
```
#### 2.两表关联更新
```
//查询TA中是否按单位净值成交，0-否，或者为空
select tf.c_tradebynetvalue, tpp.*
  from t_polling_product tpp
 inner join tfundinfo@HSTA tf
    on tpp.c_product_code = tf.c_fundcode
   and (tf.c_tradebynetvalue is null
    or tf.c_tradebynetvalue = '0')
    where tpp.c_polling_approve_status is not null;
    
//更新
update t_polling_product t
   set t.c_stock_type_level1 = '1', t.c_stock_type_level2 = '07'
 where exists (select 1
          from tfundinfo@HSTA tf
         where tf.c_fundcode = t.c_product_code
         and (tf.c_tradebynetvalue is null or tf.c_tradebynetvalue = '0'))
   and t.c_polling_approve_status is not null;
```
#### 3.A表关联B表更新A的两个字段
```
update A t1
   set (t1.name, t1.phone, t1.age) =
       (select t2.name, t2.phone, t2.age from B t2 where t2.id = t1.id)
 where t1.id in (select t2.id from B t2 where t2.id = t1.id);
```

  [1]: https://niaobulashi.com/usr/uploads/sina/5d1c5145642a1.jpg