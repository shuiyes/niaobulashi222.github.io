---
layout: post
title: 解析json报文，获取key的value
category: oracle
tags: [java, oracle]
copyright: java, oracle

---

新增解析json报文，获取key的value

## 建立如下两种类型

``` sql
CREATE OR REPLACE TYPE ty_row_str_split  as object (strValue VARCHAR2 (4000));
CREATE OR REPLACE TYPE ty_tbl_str_split AS TABLE OF ty_row_str_split;
```

## 新建json截取通用方法

``` sql
CREATE OR REPLACE FUNCTION fun_split(p_str       IN VARCHAR2,
                                     p_delimiter IN VARCHAR2)
  RETURN ty_tbl_str_split IS
  --名称：json截取通用方法
  --传入参数：
  --p_str json报文内容
  --p_delimiter json报文中的key值
  j         INT := 0;
  i         INT := 1;
  len       INT := 0;
  len1      INT := 0;
  str       VARCHAR2(4000);
  str_split ty_tbl_str_split := ty_tbl_str_split();
BEGIN
  --获取json长度len
  len  := LENGTH(p_str);
  --获取key长度len1
  len1 := LENGTH(p_delimiter);

  WHILE j < len LOOP
    j := INSTR(p_str, p_delimiter, i);

    IF j = 0 THEN
      j   := len;
      str := SUBSTR(p_str, i);
      str_split.EXTEND;
      str_split(str_split.COUNT) := ty_row_str_split(strValue => str);

      IF i >= len THEN
        EXIT;
      END IF;
    ELSE
      str := SUBSTR(p_str, i, j - i);
      i   := j + len1;
      str_split.EXTEND;
      str_split(str_split.COUNT) := ty_row_str_split(strValue => str);
    END IF;
  END LOOP;

  RETURN str_split;
END fun_split;
```


## 解析json通用方法
``` sql
create or replace FUNCTION fun_parsejson(p_jsonstr varchar2, p_key varchar2)
  RETURN VARCHAR2 AS
  --名称：解析json通用方法
  rtnVal    VARCHAR2(1000);
  i         NUMBER(2);
  jsonkey   VARCHAR2(500);
  jsonvalue VARCHAR2(1000);
  json      VARCHAR2(3000);
BEGIN
  IF p_jsonstr IS NOT NULL THEN
    --将前后的括号去掉
    json := REPLACE(p_jsonstr, '{', '');
    json := REPLACE(json, '}', '');
    json := replace(json, '"', '');
    FOR temprow IN (SELECT strvalue AS VALUE FROM TABLE(fun_split(json, ','))) LOOP
      IF temprow.VALUE IS NOT NULL THEN
        i         := 0;
        jsonkey   := '';
        jsonvalue := '';
        FOR tem2 IN (SELECT strvalue AS VALUE
                       FROM TABLE(fun_split(temprow.value, ':'))) LOOP
          IF i = 0 THEN
            jsonkey := tem2.VALUE;
          END IF;
          IF i = 1 THEN
            jsonvalue := tem2.VALUE;
          END IF;
        
          i := i + 1;
        END LOOP;
      
        IF (jsonkey = p_key) THEN
          rtnVal := jsonvalue;
        END if;
      END IF;
    END LOOP;
  END IF;
  RETURN rtnVal;
END fun_parsejson;
```

## 测试
``` sql
--更新
update t_confirm_letter t
   set t.d_confirm_date = to_date(fun_parsejson(t.report_data, 'confirmDate'),'yyyy-MM-dd')
 where 1=1
   and t.d_confirm_date is null
   and t.report_data is not null;
```



