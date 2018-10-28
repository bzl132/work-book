# 

[TOC]



##Mysql-性能优化

###**MySQL架构总览** 

![img](https://images2015.cnblogs.com/blog/701942/201512/701942-20151210224128402-1287669438.png) 

### Mysql sql语句执行顺序

1. 连接
   - 客户端发起Query请求,监听客户端的'连接管理模块'接收请求
   - 将请求转发到"连接进/线程模块"
   - 调用 用户模块 进行授权检查
2. 处理
   - 先查询缓存, 检查Query 语句是否完全匹配, 如果匹配再检查是否有该表的权限,有则返回
   - 查询缓存失败则转交给'命令解析器',经过词法分析,语法分析后生成解析树
   - 接下来是预处理阶段，处理`解析器无法解决`的语义，检查权限等，生成新的解析树
   - 再转交给对应的模块处理
   - 如果是SELECT查询还会经由‘查询优化器’做大量的优化，生成执行计划
   - 模块收到请求后，通过‘访问控制模块’检查所连接的用户是否有访问目标表和目标字段的权限
   - 有则调用‘表管理模块’，先是查看table cache中是否存在，有则直接对应的表和获取锁，否则重新打开表文件
   - 根据表的meta数据，获取表的存储引擎类型等信息，通过接口调用对应的存储引擎处理
   - 上述过程中产生数据变化的时候，若打开日志功能，则会记录到相应二进制日志文件中
3. 结果
   - Query请求完成后，将结果集返回给‘连接进/线程模块’
   - 返回的也可以是相应的状态标识，如成功或失败等
   - ‘连接进/线程模块’进行后续的清理工作，并继续等待请求或断开与客户端的连接

![img](https://images2015.cnblogs.com/blog/701942/201512/701942-20151210224221011-1559007674.png)

###SQL解析顺序

　　接下来再走一步，让我们看看一条SQL语句的前世今生。

　　首先看一下示例语句

```
SELECT DISTINCT
    < select_list >
FROM
    < left_table > < join_type >
JOIN < right_table > ON < join_condition >
WHERE
    < where_condition >
GROUP BY
    < group_by_list >
HAVING
    < having_condition >
ORDER BY
    < order_by_condition >
LIMIT < limit_number >
```

　　然而它的执行顺序是这样的

```
 1 FROM <left_table>
 2 ON <join_condition>
 3 <join_type> JOIN <right_table>
 4 WHERE <where_condition>
 5 GROUP BY <group_by_list>
 6 HAVING <having_condition>
 7 SELECT 
 8 DISTINCT <select_list>
 9 ORDER BY <order_by_condition>
10 LIMIT <limit_number>
```

　　虽然自己没想到是这样的，不过一看还是很自然和谐的，从哪里获取，不断的过滤条件，要选择一样或不一样的，排好序，那才知道要取前几条呢。

**1. FROM**

当涉及多个表的时候，左边表的输出会作为右边表的输入，之后会生成一个虚拟表VT1。

(1-J1)笛卡尔积

计算两个相关联表的笛卡尔积(CROSS JOIN) ，生成虚拟表VT1-J1。

(1-J2)ON过滤
基于虚拟表VT1-J1这一个虚拟表进行过滤，过滤出所有满足ON 谓词条件的列，生成虚拟表VT1-J2。
注意：这里因为语法限制，使用了'WHERE'代替，从中读者也可以感受到两者之间微妙的关系；

(1-J3)添加外部列

如果使用了外连接(LEFT,RIGHT,FULL)，主表（保留表）中的不符合ON条件的列也会被加入到VT1-J2中，作为外部行，生成虚拟表VT1-J3。

**2. WHERE**

对VT1过程中生成的临时表进行过滤，满足WHERE子句的列被插入到VT2表中。

注意：

​	此时因为分组，不能使用聚合运算；也不能使用SELECT中创建的别名；

​	与ON的区别：

​		如果有外部列，ON针对过滤的是关联表，主表（保留表）会返回所有的列；
​		如果没有添加外部列，两者的效果是一样的；

应用：

​	对主表的过滤应该放在WHERE；
​	对于关联表，先条件查询后连接则用ON，先连接后条件查询则用WHERE；

**3. GROUP BY**

这个子句会把VT2中生成的表按照GROUP BY中的列进行分组。生成VT3表。

注意：

​	其后处理过程的语句，如SELECT,HAVING，所用到的列必须包含在GROUP BY中，对于没有出现的，得用聚合函	数；

​	原因：

​		GROUP BY改变了对表的引用，将其转换为新的引用方式，能够对其进行下一级逻辑操作的列会减少；

​	我的理解是：

​	根据分组字段，将具有相同分组字段的记录归并成一条记录，因为每一个分组只能返回一条记录，除非是被过滤掉了，而不在分组字段里面的字段可能会有多个值，多个值是无法放进一条记录的，所以必须通过聚合函数将这些具有多值的列转换成单值；

**4. HAVING**

这个子句对VT3表中的不同的组进行过滤，只作用于分组后的数据，满足HAVING条件的子句被加入到VT4表中。

**5. SELECT**

这个子句对SELECT子句中的元素进行处理，生成VT5表。
(5-J1)计算表达式 计算SELECT 子句中的表达式，生成VT5-J1
(5-J2)DISTINCT
寻找VT5-1中的重复列，并删掉，生成VT5-J2

如果在查询中指定了DISTINCT子句，则会创建一张内存临时表（如果内存放不下，就需要存放在硬盘了）。这张临时表的表结构和上一步产生的虚拟表VT5是一样的，不同的是对进行DISTINCT操作的列增加了一个唯一索引，以此来除重复数据。

**7.LIMIT**

LIMIT子句从上一步得到的VT6虚拟表中选出从指定位置开始的指定行数据。
注意：
	offset和rows的正负带来的影响；
当偏移量很大时效率是很低的，可以这么做：
	采用子查询的方式优化，在子查询里先从索引获取到最大id，然后倒序排，再取N行结果集
	采用INNER JOIN优化，JOIN子句里也优先从索引获取ID列表，然后直接关联查询获得最终结果

**ORDER BY**

从VT5-J2中的表中，根据ORDER BY 子句的条件对结果进行排序，生成VT6表。

注意：

唯一可使用SELECT中别名的地方；

### Mysql优化点

#### 表级锁

#### 行级锁

### Mysql体系认识

#### 1.引擎

##### MyISAM 

select密集型 只读 冷数据查询
- 不支持行级锁,读取时对需要督导的所有表加锁,写入时则对表加排它锁 表级锁
- 不支持事物
- 不支持外键
- 不支持崩溃后的安全恢复
- 在表有读取查询的同时,支持网表中插入新记录
- 支持BLOB和TEXT的钱500个字符索引,支持全文索引
- 支持延迟更新索引,极大提升写入性能

##### InnoDB

insert/update密集型  UNDO/MVCC 共享锁(读)和排它锁 MVCC 行级锁
UNDO日志 历史数据 版本控制 多个版本的数据
不再对查询加锁了 提供了查询并发能力

三个隐藏字段:
- DB_ROW_ID 如果表中没有显示定义主键或者没有唯一索引则Mysql会自动创建一个6字节的Row id 存在记录中
- DB_TRX_ID 事物ID
- DB_DOLL_PTR 回滚段指针

#### 2.索引
mysql 唯一索引 查询自动索引 磁盘空间 访问数据时候提供了一种快捷方式 cpu

索引的弊端:

1. 索引本身很大,要占用很大的内存空间和硬盘空间 (通常识硬盘)
2. 索引不是所有情况均适用: a.少量数据 b.频繁更新的字段 c. 很少使用的字段
3. 索引会减低增删改的效率

索引的优点:

1. 提高查询效率(降低IO)
2. 降低cpu使用率 (order by 时 直接用索引的话,就不用再进行排序了)

索引的结构: 

​	树   B树  Hash

​	Btree:一般都是指B+ ,数据全部存放到叶子节点中

​	B+树种查询任意数据次数: n次(B树的高度)

索引分类

- 主键索引: 不能重复,不能为null
- 单值索引:单列,可以有多个
- 唯一索引:不能重复
- 复合索引:多列

创建方式:

1. 方式一:
   - 单值索引:   create index index_name table_name(column_name);
   - 唯一索引:   create unique index index_name table_name(column_name);
   - 符合索引:  create index index_name table_name(column_name1, column_name2);
2. 方式二:
   - 单值索引: alter table table_name add index index_name(column_name);
   - 唯一索引: alter table table_name add unique index index_name(column_name);
   - 符合索引: alter table table_name add index index_name(column_name1, column_name2);

删除索引:

drop index 索引名 on 表名;

查询索引

show index from table_name;

####sql执行计划

1. id 值相同 嵌套等级相同

   - 多表连表时,数据量小的表现查询

   - 连表时先查小表时,产生的中间表更小

2. select_type

   - PRIMARY: 包含子查询sql中的主查询(最外层)
   - SUBQUERY: 包含自擦洗sql 中的子查询(非最外层)
   - simple:简单查询
   - UNION: union 连表
   - UNION RESULT:

3. type

   - system          表只有一行

   - const            表最多只有一行匹配，通用用于主键或者唯一索引比较时。如将主键置于where列表中，MySQL就能将该查询转换为一个常量

   - eq_ref          每次与之前的表合并行都只在该表读取一行，这是除了system，const之外最好的一种，特点是使用=，而且索引的所有部分都参与join且索引是主键或非空唯一键的索引

   - ref               如果每次只匹配少数行，那就是比较好的一种，使用=或<=>，可以是左覆盖索引或非主键或非唯一键

   - fulltext        全文搜索

   - ref_or_null 与ref类似，但包括NULL

   - index_merge     表示出现了索引合并优化(包括交集，并集以及交集之间的并集)，但不包括跨表和全文索引。 这个比较复杂，目前的理解是合并单表的范围索引扫描（如果成本估算比普通的range要更优的话

   -  unique_subquery 在in子查询中，就是value in (select...)把形如“select unique_key_column”的子查询替换。PS：所以不一定in子句中使用子查询就是低效的！

   - index_subquery  同上，但把形如”select non_unique_key_column“的子查询替换

   - range           常数值的范围（索引范围扫描），对索引的扫描开始于某一点，返回匹配值域的行，常见于between、<、>,in等的查询   in 是比较特殊的查询

   -  index           a.当查询是索引覆盖的，即所有数据均可从索引树获取的时候（Extra中有UsingIndex）；
                             b.以索引顺序从索引中查找数据行的全表扫描（无 UsingIndex）；
                            c.如果Extra中Using Index与Using Where同时出现的话，则是利用索引查找键值的意思；
                            d.如单独出现，则是用读索引来代替读行，但不用于查找

   -  all              遍历全表以找到匹配的行

   - null：MySQL在优化过程中分解语句，执行时甚至不用访问表或索引

     结果值从好到坏依次是：

     system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
     --------------------- 

4. possible_keys:  可能用到的索引，是一种预测，不准。如果 possible_key/key是NULL，则说明没用索引

5. key ：实际使用到的索引

6. key_len ：索引的长度 ;

   - 作用：用于判断复合索引是否被完全使用  （a,b,c）。
   - 在utf8：1个字符站3个字节  
   - 如果索引字段可以为Null,则会使用1个字节用于标识。
   - varchar(20)   20*3=60 +  1(null)  +2(用2个字节 标识可变长度)  =63

7. ref:   注意与type中的ref值区分。
   	作用： 指明当前表所 参照的 字段。
   		select ....where a.c = b.x ;(其中b.x可以是常量，const)

8. rows: 被索引优化查询的 数据个数 (实际通过索引而查询到的 数据个数)

9. Extra：

   - using filesort ： 性能消耗大；需要“额外”的一次排序（查询）  。常见于 order by 语句中。小结：对于单索引， 如果排序和查找是同一个字段，则不会出现using filesort；如果排序和查找不是同一个字段，则会出现using filesort；
     	避免： where哪些字段，就order by那些字段2

   - using temporary:性能损耗大 ，用到了临时表。一般出现在group by 语句中。

     ​	explain select a1 from test02 where a1 in ('1','2','3') group by a1 ;
     	explain select a1 from test02 where a1 in ('1','2','3') group by a2 ; --using temporary
     	避免：查询那些列，就根据那些列 group by .

   - using index : 性能提升; 索引覆盖（覆盖索引）。原因：不读取原文件，只从索引文件中获取数据 （不需要回表查询）只要使用到的列 全部都在索引中，就是索引覆盖using index

   - using where （需要回表查询）
     	假设age是索引列
     	但查询语句select age,name from ...where age =...,此语句中必须回原表查Name，因此会显示using where.

   - impossible where ： where子句永远为false
     		explain select * from test02 where a1='x' and a1='y'  ;

     ​		

#### 优化案例

1. 单表优化

   - 优化 : 加索引
   - 根据SQL实际解析的顺序，调整索引的顺序
   - 再次优化（之前是index级别）：思路。因为范围查询in有时会实现，因此交换 索引的顺序，将typeid in(2,3) 放到最后。
   - 小结：a.最佳做前缀，保持索引的定义和使用的顺序一致性  b.索引需要逐步优化  c.将含In的范围查询 放到where条件的最后，防止失效。(in 条件)
   - 本例中同时出现了Using where（需要回原表）; Using index（不需要回原表）：原因，where  authorid=1 and  typeid in(2,3)中authorid在索引(authorid,typeid,bid)中，因此不需要回原表（直接在索引表中能查到）；而typeid虽然也在索引(authorid,typeid,bid)中，但是含in的范围查询已经使该typeid索引失效，因此相当于没有typeid这个索引，所以需要回原表（using where）；
     	例如以下没有了In，则不会出现using where
     	explain select bid from book where  authorid=1 and  typeid =3 order by typeid desc ;

2. 两表优化

   - 索引往哪张表加？   -小表驱动大表  
     		          -索引建立经常使用的字段上 （本题 t.cid=c.cid可知，t.cid字段使用频繁，因此给该字段加索引） [一般情况对于左外连接，给左表加索引；右外连接，给右表加索引]
     	小表：10
     	大表：300
     	where   小表.x 10 = 大表.y 300;  --循环了几次？10

   - Using join buffer: extra 中的一个选项, 作用: Mysql 引擎使用了 连接缓存.

3. 三表优化

   - 小表驱动大表
   - 索引建立在经常查询的字段上

#### 避免索引失效的原则

1. 复合索引
   - 符合索引,不要跨列或无序使用(最佳左前缀)
   - 符合索引,尽量使用全索引匹配
2. 不要再索引上进行任何操作(计算/函数/类型转换),否则索引失效
3. 复合索引不能使用不等于（!=  <>）或is null (is not null)，否则自身以及右侧所有全部失效。
   		复合索引中如果有>，则自身和右侧索引全部失效。
4. 

sql优化原则

```
禁用select *
使用select count(*) 统计行数
尽量少运算
尽量避免全表扫描，如果可以，在过滤列建立索引
尽量避免在where子句对字段进行null判断
尽量避免在where子句使用!= 或者<>
尽量避免在where子句使用or连接
尽量避免对字段进行表达式计算
尽量避免对字段进行函数操作
尽量避免使用不是复合索引的前缀列进行过滤连接
尽量少排序，如果可以，建立索引
尽量少join
尽量用join代替子查询
尽量避免在where子句中使用in,not in或者having，使用exists,not exists代替
尽量避免两端模糊匹配 like %***%
尽量用union all代替union
尽量早过滤
避免类型转换
尽量批量insert
优先优化高并发sql，而不是频率低的大sql
尽可能对每一条sql进行explain
尽可能从全局出发
```

