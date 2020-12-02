---
layout: post
title: Mysql的Explain使用及索引总结
category: Java
tags: [Java]
copyright: Java

---

## Explain工具介绍

使用EXPLAIN关键字可以模拟优化器执行SQL语句，分析你的查询语句或是结构的性能瓶颈 
在 select 语句之前增加 explain 关键字，MySQL 会在查询上设置一个标记，执行查询会返
回执行计划的信息，而不是执行这条SQL
注意：如果 from 中包含子查询，仍会执行该子查询，将结果放入临时表中。

actor建表语句

``` sql
DROP TABLE IF EXISTS `actor`;
CREATE TABLE `actor` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of actor
-- ----------------------------
INSERT INTO `actor` VALUES ('1', 'a', '2020-01-11 19:57:26');
INSERT INTO `actor` VALUES ('2', 'b', '2020-01-11 19:57:38');
INSERT INTO `actor` VALUES ('3', 'c', '2020-01-11 19:57:57');
```

film建表语句

``` sql
DROP TABLE IF EXISTS `film`;
CREATE TABLE `film` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of film
-- ----------------------------
INSERT INTO `film` VALUES ('1', 'film0');
INSERT INTO `film` VALUES ('2', 'film1');
INSERT INTO `film` VALUES ('3', 'film2');
```

film_actor建表语句

``` sql
DROP TABLE IF EXISTS `file_actor`;
CREATE TABLE `file_actor` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `film_id` int(11) NOT NULL,
  `actor_id` int(11) NOT NULL,
  `remark` varchar(25) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_film_actor_id` (`film_id`,`actor_id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of file_actor
-- ----------------------------
INSERT INTO `file_actor` VALUES ('1', '1', '1', null);
INSERT INTO `file_actor` VALUES ('2', '1', '2', null);
INSERT INTO `file_actor` VALUES ('3', '2', '1', null);
```

Explain展示的字段

## explain列

展示explain中的每个列的信息。

#### 1、id列

id列的编号是select的序号，有几个select就有几个id，并且id的顺序是按select出现的顺序增长的。

id值越大优先级越高，id相同则从上往下执行，id为NULL最后执行。

#### 2、select_type

select_type表示对应行是简单还是复杂的查询

1) simple：简单查询。查询不包含子查询和union

如上图

2) primary：复杂查询中最外层的select

3）subquery：包含在 select 中的子查询（不在 from 子句中）
4）derived：包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为
派生表（derived的英文含义）
用这个例子来了解 primary、subquery 和 derived 类型
mysql> set session optimizer_switch='derived_merge=off';   #关闭mysql5.7新特性对衍
生表的合并优化
mysql> explain select (select 1 from actor where id = 1) from (select * from film
where id = 1) der;

![1578737130125](https://images.niaobulashi.com/typecho/uploads/2020/01/2716589489.png)

mysql> set session optimizer_switch='derived_merge=on'; #还原默认配置

5）union：在 union 中的第二个和随后的 select

mysql> explain select 1 union all select 1;

![1578737906626](https://images.niaobulashi.com/typecho/uploads/2020/01/1013800051.png)

#### 3、table列

这一列表示 explain 的一行正在访问哪个表。
当 from 子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查
询，于是先执行 id=N 的查询。
当有 union 时，UNION RESULT 的 table 列的值为<union1,2>，1和2表示参与 union 的
select 行id。

#### 4、type列

这一列表示关联类型或访问类型，即MySQL决定如何查找表中的行，查找数据行记录的大概
范围。
依次从**最优到最差**分别为：system > const > eq_ref > ref > range > index > ALL
一般来说，得保证查询达到range级别，最好达到ref
NULL：mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在
索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表
mysql> explain select min(id) from film;

![1578738042346](https://images.niaobulashi.com/typecho/uploads/2020/01/521284259.png)
const, system：mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show
warnings 的结果）。用于 primary key 或 unique key 的所有列与常数比较时，所以表最多
有一个匹配行，读取1次，速度比较快。system是const的特例，表里只有一条元组匹配时为
system
mysql> explain extended select * from (select * from film where id = 1) tmp;

![1578738056468](https://images.niaobulashi.com/typecho/uploads/2020/01/1588839168.png)
mysql> show warnings;

![1578738061877](https://images.niaobulashi.com/typecho/uploads/2020/01/902622157.png)
eq_ref：primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合
条件的记录。这可能是在 const 之外最好的联接类型了，简单的 select 查询不会出现这种
type。
mysql> explain select * from film_actor left join film on film_actor.film_id = film.id;

![1578738068602](https://images.niaobulashi.com/typecho/uploads/2020/01/296224253.png)
ref：相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要
和某个值相比较，可能会找到多个符合条件的行。

1. 简单 select 查询，name是普通索引（非唯一索引）
   mysql> explain select * from film where name = 'film1';

   ![1578744663729](https://images.niaobulashi.com/typecho/uploads/2020/01/2580514485.png)

2. 关联表查询，idx_film_actor_id是film_id和actor_id的联合索引，这里使用到了film_actor
   的左边前缀film_id部分。
   mysql> explain select film_id from film left join film_actor on film.id =
   film_actor.film_id;

   ![1578744633297](https://images.niaobulashi.com/typecho/uploads/2020/01/173014895.png)

   range：范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定
   范围的行。
   mysql> explain select * from actor where id > 1;

   ![1578744638020](https://images.niaobulashi.com/typecho/uploads/2020/01/3698856681.png)

   index：扫描全表索引，这通常比ALL快一些。
   mysql> explain select * from film;

   

   ALL：即全表扫描，意味着mysql需要从头到尾去查找所需要的行。通常情况下这需要增加索
   引来进行优化了
   mysql> explain select * from actor;

![1578744600423](https://images.niaobulashi.com/typecho/uploads/2020/01/1898987101.png)

#### 5、possible_keys列

这一列显示查询可能使用哪些索引来查找。
explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中
数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。
如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可
以创造一个适当的索引来提高查询性能，然后用 explain 查看效果。

#### 6、key列

这一列显示mysql实际采用哪个索引来优化对该表的访问。
如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索
引，在查询中使用 force index、ignore index。



#### 7、key_len列

这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些
列。
举例来说，film_actor的联合索引 idx_film_actor_id 由 film_id 和 actor_id 两个int列组成，
并且每个int是4字节。通过结果中的key_len=4可推断出查询使用了第一个列：film_id列来执
行索引查找。
mysql> explain select * from film_actor where film_id = 2;

![1578745125184](https://images.niaobulashi.com/typecho/uploads/2020/01/45810768.png)

key_len计算规则如下：

- 字符串

  char(n)：n字节长度

  varchar(n)：2字节存储字符串长度，如果是utf-8，则长度3n+2

- 数值类型

  tinyint：1字节
  smallint：2字节
  int：4字节
  bigint：8字节

- 时间类型

  date：3字节
  timestamp：4字节
  datetime：8字节

- 如果字段允许为 NULL，需要1字节记录是否为 NULL

索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半
部分的字符提取出来做索引。

#### 8、ref列

这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：const（常
量），字段名（例：film.id）

#### 9、rows列

这一列是mysql估计要读取并检测的行数，注意这个不是结果集里的行数。

#### 10、Extra列

这一列展示的是额外信息。常见的重要值如下：
1）Using index：使用覆盖索引
mysql> explain select film_id from film_actor where film_id = 1;

![1578745277257](https://images.niaobulashi.com/typecho/uploads/2020/01/3956387964.png)

2）Using where：使用 where 语句来处理结果，查询的列未被索引覆盖

mysql> explain select * from actor where name = 'a';

![1578745291241](https://images.niaobulashi.com/typecho/uploads/2020/01/1722679615.png)

3）Using index condition：查询的列不完全被索引覆盖，where条件中是一个前导列的范
围；

mysql> explain select * from film_actor where film_id > 1;

![1578746022429](https://images.niaobulashi.com/typecho/uploads/2020/01/3678510800.png)

4）Using temporary：mysql需要创建一张临时表来处理查询。出现这种情况一般是要进行
优化的，首先是想到用索引来优化。

1、actor.name没有索引，此时创建了张临时表来distinct
mysql> explain select distinct name from actor;

![1578746041902](https://images.niaobulashi.com/typecho/uploads/2020/01/3121044303.png)

2、film.name建立了idx_name索引，此时查询时extra是using index,没有用临时表
mysql> explain select distinct name from film;

![1578746057709](https://images.niaobulashi.com/typecho/uploads/2020/01/941771548.png)

5）Using filesort：将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘
完成排序。这种情况下一般也是要考虑使用索引来优化的。

1、actor.name未创建索引，会浏览actor整个表，保存排序关键字name和对应的id，然后排
序name并检索行记录
mysql> explain select * from actor order by name;

![1578746090990](https://images.niaobulashi.com/typecho/uploads/2020/01/4045994156.png)

2、film.name建立了idx_name索引,此时查询时extra是using index
mysql> explain select * from film order by name;

![1578746100958](https://images.niaobulashi.com/typecho/uploads/2020/01/919562841.png)

6）Select tables optimized away：使用某些聚合函数（比如 max、min）来访问存在索引
的某个字段是
mysql> explain select min(id) from film;

![1578746115229](https://images.niaobulashi.com/typecho/uploads/2020/01/2021516824.png)

## Using filesort文件排序原理详解

#### filesort文件排序方式

单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；用trace工具可
以看到sort_mode信息里显示< sort_key, additional_fields >或者< sort_key,
packed_additional_fields >

双路排序（又叫回表排序模式）：是首先根据相应的条件取出相应的排序字段和可以直接定位行
数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段；用trace工具
可以看到sort_mode信息里显示< sort_key, rowid >

MySQL 通过比较系统变量 max_length_for_sort_data(默认1024字节) 的大小和需要查询的字段总大小来
判断使用哪种排序模式。
如果 max_length_for_sort_data 比查询字段的总长度大，那么使用 单路排序模式；
如果 max_length_for_sort_data 比查询字段的总长度小，那么使用 双路排序模式。

我们先看单路排序的详细过程：

1. 从索引name找到第一个满足 name = ‘xx’ 条件的主键 id

2. 根据主键 id 取出整行，取出所有字段的值，存入 sort_buffer 中

3. 从索引name找到下一个满足 name = ‘xx’ 条件的主键 id

4. 重复步骤 2、3 直到不满足 name = ‘xx’

5. 对 sort_buffer 中的数据按照字段 position 进行排序

6. 返回结果给客户端

   
我们再看下双路排序的详细过程：

1. 从索引 name 找到第一个满足 name = ‘xx’  的主键id
2. 根据主键 id 取出整行，把排序字段 position 和主键 id 这两个字段放到 sort buffer 中
3. 从索引 name 取下一个满足 name = ‘xx’  记录的主键 id
4. 重复 3、4 直到不满足 name = ‘xx’
5. 对 sort_buffer 中的字段 position 和主键 id 按照字段 position 进行排序
6. 遍历排序好的 id 和字段 position，按照 id 的值回到原表中取出 所有字段的值返回给客户端

其实对比两个排序模式，单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键
和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。
如果 MySQL 排序内存配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配
置小点，让优化器选择使用双路排序算法，可以在sort_buffer 中一次排序更多的行，只是需要再根据主键
回到原表取数据。
如果 MySQL 排序内存有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器
优先选择全字段排序(单路排序)，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查
询结果了。
所以，MySQL通过 max_length_for_sort_data 这个参数来控制排序，在不同场景使用不同的排序模式，
从而提升排序效率。
注意，如果全部使用sort_buffer内存排序一般情况下效率会高于磁盘文件排序，但不能因为这个就随便增
大sort_buffer(默认1M)，mysql很多参数设置都是做过优化的，不要轻易调整。

## Join关联查询优化

mysql的表关联常见有两种算法

- Nested-Loop Join 算法
- Block Nested-Loop Join 算法

#### 嵌套循环连接 Nested-Loop Join(NLJ) 算法

一次一行循环地从第一张表（称为驱动表）中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（被驱动
表）里取出满足条件的行，然后取出两张表的结果合集。
mysql> EXPLAIN select*from t1 inner join t2 on t1.a= t2.a;

![1578748848466](https://images.niaobulashi.com/typecho/uploads/2020/01/3786110290.png)

从执行计划中可以看到这些信息：
驱动表是 t2，被驱动表是 t1。先执行的就是驱动表(执行计划结果的id如果一样则按从上到下顺序执行sql)；优
化器一般会优先选择小表做驱动表。所以使用 inner join 时，排在前面的表并不一定就是驱动表。
使用了 NLJ算法。一般 join 语句中，如果执行计划 Extra 中未出现 Using join buffer 则表示使用的 join 算
法是 NLJ。
上面sql的大致流程如下：

1. 从表 t2 中读取一行数据；
2. 从第 1 步的数据中，取出关联字段 a，到表 t1 中查找；
3. 取出表 t1 中满足条件的行，跟 t2 中获取到的结果合并，作为结果返回给客户端；
4. 重复上面 3 步。

整个过程会读取 t2 表的所有数据(扫描100行)，然后遍历这每行数据中字段 a 的值，根据 t2 表中 a 的值索引扫描 t1 表
   中的对应行(扫描100次 t1 表的索引，1次扫描可以认为最终只扫描 t1 表一行完整数据，也就是总共 t1 表也扫描了100
   行)。因此整个过程扫描了 200 行。
   如果被驱动表的关联字段没索引，使用NLJ算法性能会比较低(下面有详细解释)，mysql会选择Block Nested-Loop Join
   算法。

#### 基于块的嵌套循环连接 Block Nested-Loop Join( BNL )算法

把驱动表的数据读入到 join_buffer 中，然后扫描被驱动表，把被驱动表每一行取出来跟 join_buffer 中的数据做对比。
mysql>EXPLAIN select*from t1 inner join t2 on t1.b= t2.b;

![1578748917735](https://images.niaobulashi.com/typecho/uploads/2020/01/139960535.png)


Extra 中 的Using join buffer (Block Nested Loop)说明该关联查询使用的是 BNL 算法。
上面sql的大致流程如下：

1. 把 t2 的所有数据放入到 join_buffer 中
2. 把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比
3. 返回满足 join 条件的数据

整个过程对表 t1 和 t2 都做了一次全表扫描，因此扫描的总行数为10000(表 t1 的数据总量) + 100(表 t2 的数据总量) =10100。并且 join_buffer 里的数据是无序的，因此对表 t1 中的每一行，都要做 100 次判断，所以内存中的判断次数是100 * 10000= 100 万次。

被驱动表的关联字段没索引为什么要选择使用 BNL 算法而不使用 Nested-Loop Join 呢？

如果上面第二条sql使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是磁盘扫描。
很显然，用BNL磁盘扫描次数少很多，相比于磁盘扫描，BNL的内存计算会快得多。
因此MySQL对于被驱动表的关联字段没索引的关联查询，一般都会使用 BNL 算法。如果有索引一般选择 NLJ 算法，有
索引的情况下 NLJ 算法比 BNL算法性能更高

## 对于关联sql的优化

关联字段加索引，让mysql做join操作时尽量选择NLJ算法
小标驱动大表，写多表连接sql时如果明确知道哪张表是小表可以用straight_join写法固定连接驱动方式，省去
mysql优化器自己判断的时间
straight_join解释：straight_join功能同join类似，但能让左边的表来驱动右边的表，能改表优化器对于联表查询的执
行顺序。
比如：select * from t2 straight_join t1 on t2.a = t1.a; 代表制定mysql选着 t2 表作为驱动表。
straight_join只适用于inner join，并不适用于left join，right join。（因为left join，right join已经代表指
定了表的执行顺序）
尽可能让优化器去判断，因为大部分情况下mysql优化器是比人要聪明的。使用straight_join一定要慎重，因
为部分情况下人为指定的执行顺序并不一定会比优化引擎要靠谱。
in和exsits优化
原则：小表驱动大表，即小的数据集驱动大的数据集
in：当B表的数据集小于A表的数据集时，in优于exists

``` sql
select * from A where id in (select id from B)
 #等价于：
   for(select id from B){
  select * from A where A.id = B.id
  }
```

exis`ts：当A表的数据集小于B表的数据集时，exists优于in
　　将主查询A的数据，放到子查询B中做条件验证，根据验证结果（true或false）来决定主查询的数据是否保留

``` sql
 select * from A where exists (select 1 from B where B.id = A.id)
 #等价于:
for(select * from A) {
  select * from B where B.id = A.id
}
```

#A表与B表的ID字段应建立索引
1、EXISTS (subquery)只返回TRUE或FALSE,因此子查询中的SELECT * 也可以用SELECT 1替换,官方说法是实际执行时会
忽略SELECT清单,因此没有区别
2、EXISTS子查询的实际执行过程可能经过了优化而不是我们理解上的逐条对比
3、EXISTS子查询往往也可以用JOIN来代替，何种最优需要具体问题具体分析

## 索引最佳实践

1.全值匹配

2.最左前缀法则（指的是查询从索引的最左前列开始并且不跳过索引
中的列）

3.不在索引列上做任何操作（计算、函数、（自动or手动）类型转换），会导致索引失效而转
向全表扫描

4.存储引擎不能使用索引中范围条件右边的列

5.尽量使用覆盖索引（只访问索引的查询（索引列包含查询列）），减少select *语句

6.mysql在使用不等于（！=或者<>）的时候无法使用索引会导致全表扫描

7.is null,is not null 也无法使用索引

8.like以通配符开头（'$abc...'）mysql索引失效会变成全表扫描操作

9.字符串不加单引号索引失效

10.少用or或in，用它查询时，mysql不一定使用索引，mysql内部优化器会根据检索比例、
表大小等多个因素整体评估是否使用索引，详见范围查询优化

11.范围查询优化(缩小范围)

12.查询个数推荐使用count(*)

![1578746314207](https://images.niaobulashi.com/typecho/uploads/2020/01/4263609658.png)

like KK%相当于=常量，%KK和%KK% 相当于范围





