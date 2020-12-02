---
layout: post
title: oracle的多表合并查询-工作心得
category: java
tags: [java]
copyright: java
---


刚刚开发需求写了个SQL，记个笔记，学习下关于数据库的多表合并查询的用法
---

---

      select t.* from A t 
        UNION ALL/UNION/Intersect/MINUS
      select s.* from B s;

##UNION ALL


![772156-20170328165135826-1065021997.png][1]

使用**UNION ALL**，表示取A、B的合集,不过滤重复的数据行

##UNION

![772156-20170328170010639-1715535324.png][2]

使用**UNION**，会将结果集A和结果集B进行UNION ALL运算,然后取两者交集的余集作为结果集

##Intersect
![772156-20170328170456248-1010642438.png][3]


使用Intersect,会将结果集A和结果集B进行UNION ALL运算,**然后两者之间的集交集作为结果集和UNION刚好相反**

##MINUS
![772156-20170328171317045-1304191687.png][4]

使用MINUS,取结果集A减去结果集B留下的差集,注:如果结果集A小于等于结果集B,返回空结果集.


---

好啦，下面进入实战阶段，我就直接将我写的SQL贴出来吧

    select a.*
      from (select t.c_fund_account_name as "fundAccountNo", --基金账号
                   tfp.project_code as "projectCode", --项目编号
                   tfp.project_name as "projectName", --项目名称
                   tfp.project_shortname as "projectShortName", --项目简称
                   c.c_fund_name as "fundName", --基金产品名称
                   c.c_fund_code as "fundCode", --基金产品代码
                   nvl(thold.subsistAssetsShare, 0) as "subsisAssetsShare", --份额（家族）
                   to_char(thold.updateDate, 'yyyy-MM-dd') as "updateDate", --日期
                   nvl(c.c_current_share, 0) as "currentShare", --份额（基金网站）
                   to_char(c.d_date, 'yyyy-MM-dd') as "dateDate", --日期
                   (nvl(thold.subsistAssetsShare, 0) - nvl(c.c_current_share, 0)) as "diffValue", --差值
                   CAST((CASE
                          WHEN (nvl(thold.subsistAssetsShare, 0) -
                               nvl(c.c_current_share, 0)) = 0 THEN
                           '1'
                          WHEN (nvl(thold.subsistAssetsShare, 0) -
                               nvl(c.c_current_share, 0)) <> 0 THEN
                           '0'
                        END) as nvarchar2(2)) as "identical" --是否一致
              from t_fund_account t
             inner join (select fhs.*,
                               row_number() over(partition by fhs.c_fund_account_no, fhs.c_project_code order by fhs.d_date desc) rn
                          from td_fund_holding_share fhs) c
                on t.c_fund_account_name = c.c_fund_account_no
               and t.c_project_code = c.c_project_code
              LEFT JOIN (SELECT tha.project_code as projectCode,
                               sum(tha.current_share) as subsistAssetsShare, -- 持有份额
                               sum(tha.current_cost) as currentCost, --当前成本
                               sum(tha.accumulated_profit) as accumulatedProfit, --累积利润
                               max(tha.update_time) as updateDate
                          FROM t_hold_assets tha
                          left join t_polling_product p
                            on tha.c_product_code = p.c_product_code
                         WHERE 1 = 1
                           and tha.delete_flag = '0'
                           and p.c_stock_type_level1 = '0'
                           and p.c_stock_type_level2 = '01'
                         GROUP BY tha.project_code) thold
                on thold.projectCode = t.c_project_code
              left join t_family_project tfp
                on tfp.project_code = t.c_project_code
               and tfp.delete_flag = '0'
             where rn = 1
               AND t.c_fund_account_type = '1' --基金账户
               AND t.delete_flag = '0'
            
            UNION ALL
            
            select t.c_fund_account_name as "fundAccountNo", --基金账号
                   tfp.project_code as "projectCode", --项目编号
                   tfp.project_name as "projectName", --项目名称
                   tfp.project_shortname as "projectShortName", --项目简称
                   CAST('' as nvarchar2(50)) as "fundName", --基金产品名称
                   CAST('' as nvarchar2(50)) as "fundCode", --基金产品代码
                   nvl(thold.subsistAssetsShare, 0) as "subsisAssetsShare", --份额（家族）
                   to_char(thold.updateDate, 'yyyy-MM-dd') as "updateDate", --日期
                   to_number(nvl('', 0)) as "currentShare", --份额（基金网站）
                   to_char(CAST('' as nvarchar2(50)), 'yyyy-MM-dd') as "dateDate", --日期
                   nvl(thold.subsistAssetsShare, 0) - nvl('', 0) as "diffValue", --差值
                   CAST((CASE
                          WHEN (nvl(thold.subsistAssetsShare, 0) - nvl('', 0)) = 0 THEN
                           '1'
                          WHEN (nvl(thold.subsistAssetsShare, 0) - nvl('', 0)) <> 0 THEN
                           '0'
                        END) as nvarchar2(2)) as "identical" --是否一致
              from t_fund_account t
              LEFT JOIN (SELECT tha.project_code as projectCode,
                               sum(tha.current_share) as subsistAssetsShare, -- 持有份额
                               max(tha.update_time) as updateDate
                          FROM t_hold_assets tha
                          left join t_polling_product p
                            on tha.c_product_code = p.c_product_code
                         WHERE 1 = 1
                           and tha.delete_flag = '0'
                           and p.c_stock_type_level1 = '0'
                           and p.c_stock_type_level2 = '01'
                         GROUP BY tha.project_code) thold
                on thold.projectCode = t.c_project_code
              left join t_family_project tfp
                on t.c_project_code = tfp.project_code
               and tfp.delete_flag = '0'
             where t.c_fund_account_name not in
                   (select td.c_fund_account_no from td_fund_holding_share td)
               and t.c_fund_account_type = '1'
               and t.delete_flag = '0') a
     order by a."diffValue" desc, a."projectCode" desc

> 上面sql具体意思是：查询出基金信息，A和基金对比先查询其中有的数据，再union all A有基金没有的数据，一起取个并集。OK，需求完成
> 哈哈哈哈，其实只要你SQL写得牛逼，然后了解下业务流程，什么都好说哈哈哈


> 哦对了，最后啰嗦一句。 对于这些并集计算之后，需要排序 则语法为：
> select t.* from (语句1 union all 语句2)
> t order by t.id;


![44aba224bc315c60b71173b480b1cb13485477bf.jpg][5]


  [1]: https://niaobulashi.com/usr/uploads/2018/11/4230521173.png
  [2]: https://niaobulashi.com/usr/uploads/2018/11/2605756222.png
  [3]: https://niaobulashi.com/usr/uploads/2018/11/718117220.png
  [4]: https://niaobulashi.com/usr/uploads/2018/11/1648560351.png
  [5]: https://niaobulashi.com/usr/uploads/2018/11/593399938.jpg