---
layout: post
title: oracle存储过程 关于update的动态SQL-工作心得
category: oracle
tags: [java, oracle]
copyright: java, oracle
---

花了我一下午时间，外加晚上加班
终于把存储过程的主要功能给写出来了！！！
emmmmm，做个笔记
后面有时间就细细分析下，先把代码贴一下。

 
```
    create or replace procedure P_SYNC_TA_PRODUCT_TEST(v_res           OUT NUMBER,
                                                           v_errorCode     OUT NVARCHAR2,
                                                           v_errorMsg      OUT NVARCHAR2,
                                                           d_dateStartDate IN NVARCHAR2,
                                                           d_dateEndDate   IN NVARCHAR2) IS
          --查询是否存在已经同步的TA产品信息的个数
          v_count_updatePolling number(8) := 0;
          --收益率ID自动生成序列
          v_yieldRateId NVARCHAR2(36);
          --受益级别表的产品代码
          v_productCode varchar2(36);
          --受益级别表的受益级别
          v_yieldLevelCode NVARCHAR2(10);
          --受益级别表的预期年化收益率
          v_yieldRate NVARCHAR2(20);
          v_yieldRateStr NVARCHAR2(20);
          --受益级别表的投资区间（开始）
          v_investmentIntervalS NVARCHAR2(20);
          v_investmentIntervalSStr NVARCHAR2(20);
          --受益级别表的投资区间（结束）
          v_investmentIntervalD NVARCHAR2(20);
          v_investmentIntervalDStr NVARCHAR2(20);
          --受益级别表的投资期限
          v_dueTime NVARCHAR2(20);
          --受益级别表的投资期限单位
          v_dueTimeUnit NVARCHAR2(2);
          --受益级别表的年化计算天数
          v_annualDays NVARCHAR2(5);
          --拼接的SQL语句
          v_confirmSql varchar2(600);
          v_confirmCommaSql varchar2(600);
          v_updateYieldRateSql varchar2(700);
          
          --用于判断是否新增或更新
          /*v_total_pro number(8) := 0;
          --游标tfundInfos的个数
          v_total number(8) := 0;
          --入池产品ID
          v_pollingProductId NVARCHAR2(36);
          --用于判断产品表是否存在该产品
          v_count_pollingProduct number(8) := 0;
          --用于判断是否存在已入池且当日改动的产品
          v_count_polling number(8) := 0;
          --用于判断TA是否存在受益级别信息
          v_count_productRateYiled number(8) := 0;*/
        
          --1.定义游标：查询出产品名称、成立日、预计到期日其中和TA不一致的产品信息
          CURSOR updateProductInfos IS
            select t.c_fundcode as fundCode, --产品代码
                   t.c_fundname as fundName, --产品名称（TA）
                   to_char(t.d_setupdate, 'yyyy-MM-dd') as setupDateStr, --成立日（TA）
                   to_char(t.d_contractenddate, 'yyyy-MM-dd') as contractendDateStr --预计到期日期（TA）
              from tfundinfo@HSTA t
              inner join t_polling_product tpp
                on tpp.c_product_code = t.c_fundcode
                and tpp.delete_flag = '0'
             where 1 = 1
               and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') >=
                   to_char(to_date(d_dateStartDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
               and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') <=
                   to_char(to_date(d_dateEndDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
               and t.c_fundstatus <> '6'
               and (t.c_fundname <> tpp.c_product_name
               or to_char(t.d_setupdate, 'yyyy-MM-dd') <> to_char(tpp.c_found_date, 'yyyy-MM-dd')
               or to_char(t.d_contractenddate, 'yyyy-MM-dd') <> to_char(tpp.c_end_date, 'yyyy-MM-dd'));
               --排除已经结束的产品，0-募集期，1-正常，6-已结束
        
          --2.定义游标：核对受益级别数据后更新或新增家族信托产品受益级别
          CURSOR margeProductYieldRates IS
            select t.c_fundcode AS fundCode, --产品代码
                   tp.c_profitclass AS profitClass, --收益级别
                   tp.f_profit*100 AS profit, --从TA获取的收益率需要乘以100
                   tp.f_amountmin AS amountMin, --投资下限（含）
                   tp.f_amountmax AS amountMax, --投资上限（不含）
                   tp.c_durationtime AS durationTime, --投资期限
                   case tp.c_durationtimeunit
                     when '0' then 'Y'  --年
                     when '1' then 'M'  --月
                     when '2' then 'D'  --日
                     else ''
                   end as durationTimeUnit, --期限单位
                   case tp.c_bonusfrequency
                     when '0' then '04'
                     when '1' then '04'
                     when '2' then '04'
                     when '3' then '04'
                     when '4' then '04'
                     when '5' then '04' --特定周期
                     when '6' then '02' --到期一次性清算
                     when '7' then '03' --不定期
                     else ''
                   end as bonusFrequencyType, --分配频率类型
                   case tp.c_bonusfrequency
                     when '2' then 'M' --每月
                     when '3' then 'Q' --每季度
                     when '4' then 'H' --每半年
                     when '5' then 'Y' --每年
                     else ''
                   end as bonusFrequency, --分配频率_特定
                   tp.l_incomeyeardays as incomeYearDays, --年化天数
                   NVL(to_char(t.d_setupdate, 'yyyy-MM-dd'),
                       to_char(sysdate, 'yyyy-MM-dd')) AS yieldStartDateStr --有效开始日期
              from tfundinfo@HSTA t
              inner join ttrustfundprofit@HSTA tp
                on tp.c_fundcode = t.c_fundcode
             where 1 = 1
               and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') >=
                   to_char(to_date(d_dateStartDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
               and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') <=
                   to_char(to_date(d_dateEndDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
               and t.c_fundstatus <> '6' and t.c_fundcode in ('CA0FX6','CA0FUC'); --排除已经结束的产品，0-募集期，1-正常，6-已结束
               
          
        begin
          --================================================================================
          -------------------------------【执行sql文】--------------------------------------
          --================================================================================
          --开启日志输出缓冲
          DBMS_OUTPUT.ENABLE(buffer_size => null);
          DBMS_OUTPUT.put_line('----------------start------------------');
        
          
          --1.查询是否存在已经同步的TA产品信息
          select count(*)
            into v_count_updatePolling
            from tfundinfo@HSTA t
            inner join t_polling_product tpp
              on tpp.c_product_code = t.c_fundcode
           where 1 = 1
             and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') >=
                 to_char(to_date(d_dateStartDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
             and to_char(t.d_lastmodifydate, 'yyyy-MM-dd') <=
                 to_char(to_date(d_dateEndDate, 'yyyy-MM-dd'), 'yyyy-MM-dd')
             and t.c_fundstatus <> '6';     --排除已经结束的产品，0-募集期，1-正常，6-已结束
        
          --如果查询到存在
          if v_count_updatePolling > 0 then
            
            --1.1 调用游标：查询出产品名称、成立日、预计到期日其中和TA不一致的产品信息
            /*for updateProductInfo in updateProductInfos loop
              --非空判断，如果查询到的数据存在，表示，存在家族信托和TA不一致的产品信息，进行更新处理
              if updateProductInfo.Fundcode is not null then
                --查询存在说明存在和TA不一致的产品基本信息，则更新和TA一致
                update t_polling_product t
                   set t.c_product_name       = updateProductInfo.Fundname,
                       t.c_end_date           = to_date(updateProductInfo.Contractenddatestr, 'yyyy-MM-dd'),
                       t.update_time          = sysdate,
                       t.c_found_date         = to_date(updateProductInfo.Setupdatestr, 'yyyy-MM-dd')
                 where t.c_product_code = updateProductInfo.Fundcode;
              end if;
            end loop;*/
            
            
            --1.2 调用游标：核对受益级别数据后更新或新增家族信托产品受益级别
            for margeProductYieldRate IN margeProductYieldRates loop
              if margeProductYieldRate.Fundcode is not null then
                /*
                 * 1.2.1
                 * 查询TA中的受益级别，家族信托是否存在
                 * 若存在，则表示家族和TA都存在该受益级别，进行核对比较关键信息，发现不同则更新
                 * 若不存在，表示家族这边缺少TA的受益级别，需要从TA进行同步该受益级别
                 */
                select tpy.c_product_code,
                       tpy.c_yield_level_code,
                       tpy.c_yield_rate,
                       tpy.c_investment_interval_s,
                       tpy.c_investment_interval_d,
                       tpy.c_due_time,
                       tpy.c_due_time_unit,
                       tpy.c_annual_days
                  into v_productCode, --产品代码
                       v_yieldLevelCode, --受益级别
                       v_yieldRate, --预期收益率
                       v_investmentIntervalS, --投资下限
                       v_investmentIntervalD, --投资上限
                       v_dueTime, --投资期限
                       v_dueTimeUnit, --投资期限单位
                       v_annualDays --年化天数
                  from t_product_yield_rate tpy
                 where tpy.c_yield_level_code = margeProductYieldRate.Profitclass
                   and tpy.c_product_code = margeProductYieldRate.Fundcode
                   and tpy.delete_flag = '0';
                
                --若存在，则表示家族和TA都存在该受益级别，进行核对比较关键信息，发现不同则更新
                IF v_productCode is not null then
                  /*
                   * 将匹配的数据进行核对，发现关键字段不匹配的进行更新
                   * 对受益级别的关键字段进行一一对比
                   * 对比不一样的，进行拼接SQL
                   */
                  dbms_output.put_line('开始拼接SQL，产品代码为：' || margeProductYieldRate.Fundcode||','||'受益级别为：'||margeProductYieldRate.Profitclass);
                  
                  --oracle  数据库 字段值为小于1的小数时，使用varchar2类型处理，会丢失小数点前面的0
                  v_yieldRateStr := to_char(v_yieldRate, 'fm99999999990.00');
                  v_investmentIntervalSStr := to_char(v_investmentIntervalS, 'fm99999999990.00');
                  v_investmentIntervalDStr := to_char(v_investmentIntervalD, 'fm99999999990.00');
                  
                  dbms_output.put_line('1.收益率，family:' ||v_yieldRateStr||';'||'TA:'||to_char(margeProductYieldRate.Profit, 'fm99999999990.00'));
                  -- 比较年化收益率
                  if v_yieldRateStr is not null and margeProductYieldRate.Profit is not null then
                    if v_yieldRateStr <> to_char(margeProductYieldRate.Profit, 'fm99999999990.00') then
                      v_confirmSql := v_confirmSql||'t.c_yield_rate='||to_char(margeProductYieldRate.Profit, 'fm99999999990.00')||',';
                    end if;
                  elsif v_yieldRateStr is null and to_char(margeProductYieldRate.Profit, 'fm99999999990.00') is not null then
                    v_confirmSql := v_confirmSql||'t.c_yield_rate='||to_char(margeProductYieldRate.Profit, 'fm99999999990.00')||',';
                  elsif v_yieldRateStr is not null and to_char(margeProductYieldRate.Profit, 'fm99999999990.00') is null then
                    v_confirmSql := v_confirmSql||'t.c_yield_rate=null,';
                  end if;
                  --比较投资上限（含）
                  dbms_output.put_line('2.投资上限，family:' || v_investmentIntervalSStr||';'||'TA:'||to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00'));
                  if v_investmentIntervalSStr is not null and to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00') is not null then
                    if v_investmentIntervalSStr <> to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00') then
                      v_confirmSql := v_confirmSql||'t.c_investment_interval_s='||to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00')||',';
                    end if;
                  elsif v_investmentIntervalSStr is null and to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00') is not null then
                    v_confirmSql := v_confirmSql||'t.c_investment_interval_s='||to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00')||',';
                  elsif v_investmentIntervalSStr is not null and to_char(margeProductYieldRate.Amountmin, 'fm99999999990.00') is null then
                    v_confirmSql := v_confirmSql||'t.c_investment_interval_s=null,';
                  end if;
                  --比较投资下限（不含）          
                  dbms_output.put_line('2.投资下限，family:' || v_investmentIntervalDStr||';'||'TA:'||to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00'));
                  if v_investmentIntervalDStr is not null and to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00') is not null then
                    if v_investmentIntervalDStr <> to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00') then
                      v_confirmSql := v_confirmSql||'t.c_investment_interval_d='||to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00')||',';
                    end if;
                  elsif v_investmentIntervalDStr is null and to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00') is not null then
                    v_confirmSql := v_confirmSql||'t.c_investment_interval_d='||to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00')||',';
                  elsif v_investmentIntervalDStr is not null and to_char(margeProductYieldRate.Amountmax, 'fm99999999990.00') is null then
                    v_confirmSql := v_confirmSql||'t.c_investment_interval_d=null,';
                  end if;
                  --比较投资期限
                  dbms_output.put_line('3.投资期限，family:' || v_dueTime||';'||'TA:'||margeProductYieldRate.Durationtime);
                  if v_dueTime is not null and margeProductYieldRate.Durationtime is not null then
                    if v_dueTime <> margeProductYieldRate.Durationtime then
                      v_confirmSql := v_confirmSql||'t.c_due_time='||margeProductYieldRate.Durationtime||',';
                    end if;
                  elsif v_dueTime is null and margeProductYieldRate.Durationtime is not null then
                    v_confirmSql := v_confirmSql||'t.c_due_time='||margeProductYieldRate.Durationtime||',';
                  elsif v_dueTime is not null and margeProductYieldRate.Durationtime is null then
                    v_confirmSql := v_confirmSql||'t.c_due_time=null,';
                  end if;
                  --比较投资期限单位
                  dbms_output.put_line('4.投资期限单位，family:' || v_dueTimeUnit||';'||'TA:'||margeProductYieldRate.Durationtimeunit);
                  if v_dueTimeUnit is not null and margeProductYieldRate.Durationtimeunit is not null then
                    if v_dueTimeUnit <> margeProductYieldRate.Durationtimeunit then
                      v_confirmSql := v_confirmSql||'t.c_due_time_unit='||''''||margeProductYieldRate.Durationtimeunit||''''||',';
                    end if;
                  elsif v_dueTimeUnit is null and margeProductYieldRate.Durationtimeunit is not null then
                    v_confirmSql := v_confirmSql||'t.c_due_time_unit='||''''||margeProductYieldRate.Durationtimeunit||''''||',';
                  elsif v_dueTimeUnit is not null and margeProductYieldRate.Durationtimeunit is null then
                    v_confirmSql := v_confirmSql||'t.c_due_time_unit=null,';
                  end if;
                  --比较年化计算天数
                  dbms_output.put_line('5.年化计算天数，family:' || v_annualDays||';'||'TA:'||margeProductYieldRate.Incomeyeardays);
                  if v_annualDays is not null and margeProductYieldRate.Incomeyeardays is not null then
                    if v_annualDays <> margeProductYieldRate.Incomeyeardays then
                      v_confirmSql := v_confirmSql||'t.c_annual_days'||''''||margeProductYieldRate.Incomeyeardays||''''||',';
                    end if;
                  elsif v_annualDays is null and margeProductYieldRate.Incomeyeardays is not null then
                    v_confirmSql := v_confirmSql||'t.c_annual_days='||''''||margeProductYieldRate.Incomeyeardays||''''||',';
                  elsif v_annualDays is not null and margeProductYieldRate.Incomeyeardays is null then
                    v_confirmSql := v_confirmSql||'t.c_annual_days=null,';
                  end if;
                  
                  --判断存在不一致的受益级别字段
                  if v_confirmSql is not null then
                    --将得到的拼接字符串，去掉最后的一个句号','
                    select substr(v_confirmSql,1,length(v_confirmSql)-1) into v_confirmCommaSql from dual;
                    v_updateYieldRateSql := 'update t_product_yield_rate t set '||v_confirmCommaSql||' where t.c_product_code=:1 and t.c_yield_level_code=:2';
                    dbms_output.put_line('拼接的SQL为：' || v_updateYieldRateSql);
                    
                    --执行更新SQL
                    execute immediate v_updateYieldRateSql USING margeProductYieldRate.Fundcode, margeProductYieldRate.Profitclass;
                    
                    --本条记录处理完成，将v_confirmSql、v_confirmSqlCount清空准备下条处理
                    v_confirmSql := '';
                    v_confirmCommaSql := '';
                    v_updateYieldRateSql := '';
                  end if;
                  
                  --若不存在，表示家族这边缺少TA的受益级别，需要从TA进行同步该受益级别
                else
                  --生成收益率ID主键的序列号
                  select concat(to_char(sysdate, 'yyyyMMddHHmmss'),
                                trunc(dbms_random.value(100000, 999999)))
                    into v_yieldRateId
                    from dual;
                  dbms_output.put_line('存在新增的受益级别，产品代码为：'||margeProductYieldRate.Fundcode||',受益级别为：'||margeProductYieldRate.Profitclass);
                  --将该受益级别同步到家族信托中
                  insert into T_PRODUCT_YIELD_RATE
                  (T_PRODUCT_YIELD_RATE.C_YIELD_RATE_ID,
                   T_PRODUCT_YIELD_RATE.C_PRODUCT_CODE,
                   T_PRODUCT_YIELD_RATE.C_PRODUCT_MANAGER_CODE,
                   T_PRODUCT_YIELD_RATE.C_INVESTMENT_INTERVAL_S,
                   T_PRODUCT_YIELD_RATE.C_INVESTMENT_INTERVAL_D,
                   T_PRODUCT_YIELD_RATE.C_YIELD_RATE,
                   T_PRODUCT_YIELD_RATE.DELETE_FLAG,
                   T_PRODUCT_YIELD_RATE.CREATE_USER_ID,
                   T_PRODUCT_YIELD_RATE.CREATE_TIME,
                   T_PRODUCT_YIELD_RATE.UPDATE_USER_ID,
                   T_PRODUCT_YIELD_RATE.UPDATE_TIME,
                   T_PRODUCT_YIELD_RATE.C_YIELD_START_DATE,
                   T_PRODUCT_YIELD_RATE.C_YIELD_END_DATE,
                   T_PRODUCT_YIELD_RATE.C_CHANGE_REASON,
                   T_PRODUCT_YIELD_RATE.C_YIELD_LEVEL_CODE,
                   T_PRODUCT_YIELD_RATE.C_ALLOCATION_TYPE,
                   T_PRODUCT_YIELD_RATE.C_ALLOCATION_TYPE_SP,
                   T_PRODUCT_YIELD_RATE.C_ALLOCATION_TYPE_MD,
                   T_PRODUCT_YIELD_RATE.C_ANNUAL_DAYS,
                   T_PRODUCT_YIELD_RATE.C_DUE_TIME,
                   T_PRODUCT_YIELD_RATE.C_DUE_TIME_UNIT)
                values
                  (v_yieldRateId,
                   margeProductYieldRate.Fundcode,
                   '0',
                   margeProductYieldRate.Amountmin,
                   margeProductYieldRate.Amountmax,
                   margeProductYieldRate.Profit,
                   '0',
                   '00000000000000000000',
                   sysdate,
                   '00000000000000000000',
                   sysdate,
                   to_date(margeProductYieldRate.Yieldstartdatestr, 'yyyy-MM-dd'),
                   to_date('9999-12-31', 'yyyy-MM-dd'),
                   '',
                   margeProductYieldRate.Profitclass,
                   margeProductYieldRate.Bonusfrequencytype,
                   margeProductYieldRate.Bonusfrequency,
                   '',
                   margeProductYieldRate.Incomeyeardays,
                   margeProductYieldRate.Durationtime,
                   margeProductYieldRate.Durationtimeunit);
                end if;
              end if;
            end loop;
          end if;
        
          
          --输出放回状态信息
          v_res       := 0;
          v_errorCode := SQLCODE;
          v_errorMsg  := 'P_SYNC_TA_PRODUCT_TEST' || ':' || TO_CHAR(SQLERRM);
          DBMS_OUTPUT.put_line('----------------end------------------');
          --提交
          COMMIT;
          --异常处理
        EXCEPTION
          WHEN OTHERS THEN
            DBMS_OUTPUT.put_line('----------------error------------------');
            --事务回滚
            ROLLBACK;
            v_res       := -1;
            v_errorCode := SQLCODE;
            v_errorMsg  := 'P_SYNC_TA_PRODUCT_TEST' || ':' || TO_CHAR(SQLERRM);
        end P_SYNC_TA_PRODUCT_TEST;
```
