---
layout: post
title: ORA-02049：oracle超时分布式事务处理等待锁
category: java
tags: [java]
copyright: java
---

## 查询死锁信息
```
SELECT username,
       lockwait,
       status,
       machine,
       program
  FROM v$session
 WHERE sid IN (SELECT session_id FROM v$locked_object);
```

## 查找被锁的SQL语句
```
SELECT sql_text
  FROM v$sql
 WHERE hash_value IN (SELECT sql_hash_value
                        FROM v$session
                       WHERE sid IN (SELECT session_id FROM v$locked_object));

```

## 查找被死锁的进程
```
SELECT s.username,
       l.OBJECT_ID,
       l.SESSION_ID,
       s.SERIAL#,
       l.ORACLE_USERNAME,
       l.OS_USER_NAME,
       l.PROCESS
  FROM V$LOCKED_OBJECT l, V$SESSION S
 WHERE l.SESSION_ID = S.SID;

--或者

SELECT 
 S.USERNAME,
 DECODE(L.TYPE, 'TM', 'TABLE LOCK', 'TX', 'ROW LOCK', NULL) LOCK_LEVEL,
 O.OWNER,
 O.OBJECT_NAME,
 O.OBJECT_TYPE,
 S.SID,
 S.SERIAL#,
 S.TERMINAL,
 S.MACHINE,
 S.PROGRAM,
 S.OSUSER
  FROM V$SESSION S, V$LOCK L, DBA_OBJECTS O
 WHERE L.SID = S.SID
   AND L.ID1 = O.OBJECT_ID(+)
   AND S.USERNAME IS NOT NULL;
```

## 根据实际情况决定是否kill
```
alter system kill session 'sid,serial#';
```