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


	（3）复合索引不能使用不等于（!=  <>）或is null (is not null)，否则自身以及右侧所有全部失效。
		复合索引中如果有>，则自身和右侧索引全部失效。

	explain select * from book where authorid = 1 and typeid =2 ;

	-- SQL优化，是一种概率层面的优化。至于是否实际使用了我们的优化，需要通过explain进行推测。

	explain select * from book where authorid != 1 and typeid =2 ;
	explain select * from book where authorid != 1 and typeid !=2 ;
		
	
	体验概率情况(< > =)：原因是服务层中有SQL优化器，可能会影响我们的优化。
	drop index idx_typeid on book;
	drop index idx_authroid on book;
	alter table book add index idx_book_at (authorid,typeid);
	explain select * from book where authorid = 1 and typeid =2 ;--复合索引at全部使用
	explain select * from book where authorid > 1 and typeid =2 ; --复合索引中如果有>，则自身和右侧索引全部失效。
	explain select * from book where authorid = 1 and typeid >2 ;--复合索引at全部使用
	----明显的概率问题---
	explain select * from book where authorid < 1 and typeid =2 ;--复合索引at只用到了1个索引
	explain select * from book where authorid < 4 and typeid =2 ;--复合索引全部失效

	--我们学习索引优化 ，是一个大部分情况适用的结论，但由于SQL优化器等原因  该结论不是100%正确。
	--一般而言， 范围查询（> <  in），之后的索引失效。

	（4）补救。尽量使用索引覆盖（using index）
			（a,b,c）
	select a,b,c from xx..where a=  .. and b =.. ;

	(5) like尽量以“常量”开头，不要以'%'开头，否则索引失效
	select * from xx where name like '%x%' ; --name索引失效
	
	explain select * from teacher  where tname like '%x%'; --tname索引失效

	explain select * from teacher  where tname like 'x%';
 
	explain select tname from teacher  where tname like '%x%'; --如果必须使用like '%x%'进行模糊查询，可以使用索引覆盖 挽救一部分。


	（6）尽量不要使用类型转换（显示、隐式），否则索引失效
	explain select * from teacher where tname = 'abc' ;
	explain select * from teacher where tname = 123 ;//程序底层将 123 -> '123'，即进行了类型转换，因此索引失效

	（7）尽量不要使用or，否则索引失效
	explain select * from teacher where tname ='' or tcid >1 ; --将or左侧的tname 失效。

8.一些其他的优化方法
	（1）
	exist和in
	select ..from table where exist (子查询) ;
	select ..from table where 字段 in  (子查询) ;

	如果主查询的数据集大，则使用In   ,效率高。
	如果子查询的数据集大，则使用exist,效率高。	

	exist语法： 将主查询的结果，放到子查需结果中进行条件校验（看子查询是否有数据，如果有数据 则校验成功）  ，
		    如果 复合校验，则保留数据；

	select tname from teacher where exists (select * from teacher) ; 
	--等价于select tname from teacher

	
	select tname from teacher where exists (select * from teacher where tid =9999) ;
	
	in:
	select ..from table where tid in  (1,3,5) ;
	

	（2）order by 优化
	using filesort 有两种算法：双路排序、单路排序 （根据IO的次数）
	MySQL4.1之前 默认使用 双路排序；双路：扫描2次磁盘（1：从磁盘读取排序字段 ,对排序字段进行排序（在buffer中进行的排序）   2：扫描其他字段 ）
		--IO较消耗性能
	MySQL4.1之后 默认使用 单路排序  ： 只读取一次（全部字段），在buffer中进行排序。但种单路排序 会有一定的隐患 （不一定真的是“单路|1次IO”，有可能多次IO）。原因：如果数据量特别大，则无法 将所有字段的数据 一次性读取完毕，因此 会进行“分片读取、多次读取”。
		注意：单路排序 比双路排序 会占用更多的buffer。
			单路排序在使用时，如果数据大，可以考虑调大buffer的容量大小：  set max_length_for_sort_data = 1024  单位byte

	如果max_length_for_sort_data值太低，则mysql会自动从 单路->双路   （太低：需要排序的列的总大小超过了max_length_for_sort_data定义的字节数）

	提高order by查询的策略：
	a.选择使用单路、双路 ；调整buffer的容量大小；
	b.避免select * ...  
	c.复合索引 不要跨列使用 ，避免using filesort
	d.保证全部的排序字段 排序的一致性（都是升序 或 降序）
		
	
9.SQL排查 - 慢查询日志:MySQL提供的一种日志记录，用于记录MySQL种响应时间超过阀值的SQL语句 （long_query_time，默认10秒）
		慢查询日志默认是关闭的；建议：开发调优是 打开，而 最终部署时关闭。
	
	检查是否开启了 慢查询日志 ：   show variables like '%slow_query_log%' ;

	临时开启：
		set global slow_query_log = 1 ;  --在内存种开启
		exit
		service mysql restart

	永久开启：
		/etc/my.cnf 中追加配置：
		vi /etc/my.cnf 
		[mysqld]
		slow_query_log=1
		slow_query_log_file=/var/lib/mysql/localhost-slow.log

	
	慢查询阀值：
		show variables like '%long_query_time%' ;

	临时设置阀值：
		set global long_query_time = 5 ; --设置完毕后，重新登陆后起效 （不需要重启服务）

	永久设置阀值：
			
		/etc/my.cnf 中追加配置：
		vi /etc/my.cnf 
		[mysqld]
		long_query_time=3


	select sleep(4);
	select sleep(5);
	select sleep(3);
	select sleep(3);
	--查询超过阀值的SQL：  show global status like '%slow_queries%' ;
	
	(1)慢查询的sql被记录在了日志中，因此可以通过日志 查看具体的慢SQL。
	cat /var/lib/mysql/localhost-slow.log

	(2)通过mysqldumpslow工具查看慢SQL,可以通过一些过滤条件 快速查找出需要定位的慢SQL
	mysqldumpslow --help
	s：排序方式
	r:逆序
	l:锁定时间
	g:正则匹配模式		


	--获取返回记录最多的3个SQL
		mysqldumpslow -s r -t 3  /var/lib/mysql/localhost-slow.log

	--获取访问次数最多的3个SQL
		mysqldumpslow -s c -t 3 /var/lib/mysql/localhost-slow.log

	--按照时间排序，前10条包含left join查询语句的SQL
		mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/localhost-slow.log
	
	语法：
		mysqldumpslow 各种参数  慢查询日志的文件


10.分析海量数据
   
	a.模拟海量数据  存储过程（无return）/存储函数（有return）
	create database testdata ;
	use testdata
create table dept
(
dno int(5) primary key default 0,
dname varchar(20) not null default '',
loc varchar(30) default ''
)engine=innodb default charset=utf8;

create table emp
(
eid int(5) primary key,
ename varchar(20) not null default '',
job varchar(20) not null default '',
deptno int(5) not null default 0
)engine=innodb default charset=utf8;
	通过存储函数 插入海量数据：
	创建存储函数：
		randstring(6)  ->aXiayx  用于模拟员工名称


	delimiter $ 
	create function randstring(n int)   returns varchar(255) 
	begin
		declare  all_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ' ;
		declare return_str varchar(255) default '' ;
		declare i int default 0 ; 
		while i<n		 
		do									
			set return_str = concat(  return_str,      substring(all_str,   FLOOR(1+rand()*52)   ,1)       );
			set i=i+1 ;
		end while ;
		return return_str;
		
	end $ 
	

--如果报错：You have an error in your SQL syntax，说明SQL语句语法有错，需要修改SQL语句；

 如果报错This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA in its declaration and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)
	是因为 存储过程/存储函数在创建时 与之前的 开启慢查询日志冲突了 
	解决冲突：
	临时解决( 开启log_bin_trust_function_creators )
		show variables like '%log_bin_trust_function_creators%';
		set global log_bin_trust_function_creators = 1;
	永久解决：
	/etc/my.cnf 
	[mysqld]
	log_bin_trust_function_creators = 1


	--产生随机整数
	create function ran_num() returns int(5)
	begin
		declare i int default 0;
		set i =floor( rand()*100 ) ;
		return i ;

	end $
	


	--通过存储过程插入海量数据：emp表中  ，  10000,   100000
	create procedure insert_emp( in eid_start int(10),in data_times int(10))
	begin 
		declare i int default 0;
		set autocommit = 0 ;
		
		repeat
			
			insert into emp values(eid_start + i, randstring(5) ,'other' ,ran_num()) ;
			set i=i+1 ;
			until i=data_times
		end repeat ;
		commit ;
	end $


	--通过存储过程插入海量数据：dept表中  
		create procedure insert_dept(in dno_start int(10) ,in data_times int(10))
		begin
			declare i int default 0;
			set autocommit = 0 ;
			repeat
			
				insert into dept values(dno_start+i ,randstring(6),randstring(8)) ;
				set i=i+1 ;
				until i=data_times
			end repeat ;
		commit ;
			

		end$


	--插入数据
		delimiter ; 
		call insert_emp(1000,800000) ;
		call insert_dept(10,30) ;

	
	b.分析海量数据:
	（1）profiles
	show profiles ; --默认关闭
	show variables like '%profiling%';
	set profiling = on ; 
	show profiles  ：会记录所有profiling打开之后的  全部SQL查询语句所花费的时间。缺点：不够精确，只能看到 总共消费的时间，不能看到各个硬件消费的时间（cpu  io ）

	(2)--精确分析:sql诊断
	 show profile all for query 上一步查询的的Query_Id
	 show profile cpu,block io for query 上一步查询的的Query_Id

	(3)全局查询日志 ：记录开启之后的 全部SQL语句。 （这次全局的记录操作 仅仅在调优、开发过程中打开即可，在最终的部署实施时 一定关闭）
		show variables like '%general_log%';
		
		--执行的所有SQL记录在表中
		set global general_log = 1 ;--开启全局日志
		set global log_output='table' ; --设置 将全部的SQL 记录在表中

		--执行的所有SQL记录在文件中
		set global log_output='file' ;
		set global general_log = on ;
		set global general_log_file='/tmp/general.log' ;
		

		开启后，会记录所有SQL ： 会被记录 mysql.general_log表中。
			select * from  mysql.general_log ;

11.锁机制 ：解决因资源共享 而造成的并发问题。
	示例：买最后一件衣服X
	A:  	X	买 ：  X加锁 ->试衣服...下单..付款..打包 ->X解锁
	B:	X       买：发现X已被加锁，等待X解锁，   X已售空

	分类：
	操作类型：
		a.读锁（共享锁）： 对同一个数据（衣服），多个读操作可以同时进行，互不干扰。
		b.写锁（互斥锁）： 如果当前写操作没有完毕（买衣服的一系列操作），则无法进行其他的读操作、写操作

	操作范围：
		a.表锁 ：一次性对一张表整体加锁。如MyISAM存储引擎使用表锁，开销小、加锁快；无死锁；但锁的范围大，容易发生锁冲突、并发度低。
		b.行锁 ：一次性对一条数据加锁。如InnoDB存储引擎使用行锁，开销大，加锁慢；容易出现死锁；锁的范围较小，不易发生锁冲突，并发度高（很小概率 发生高并发问题：脏读、幻读、不可重复度、丢失更新等问题）。
		c.页锁		
	
示例：

	（1）表锁 ：  --自增操作 MYSQL/SQLSERVER 支持；oracle需要借助于序列来实现自增
create table tablelock
(
id int primary key auto_increment , 
name varchar(20)
)engine myisam;


insert into tablelock(name) values('a1');
insert into tablelock(name) values('a2');
insert into tablelock(name) values('a3');
insert into tablelock(name) values('a4');
insert into tablelock(name) values('a5');
commit;

	增加锁：
	locak table 表1  read/write  ,表2  read/write   ,...

	查看加锁的表：
	show open tables ;

	会话：session :每一个访问数据的dos命令行、数据库客户端工具  都是一个会话

	===加读锁：
		会话0：
			lock table  tablelock read ;
			select * from tablelock; --读（查），可以
			delete from tablelock where id =1 ; --写（增删改），不可以

			select * from emp ; --读，不可以
			delete from emp where eid = 1; --写，不可以
			结论1：
			--如果某一个会话 对A表加了read锁，则 该会话 可以对A表进行读操作、不能进行写操作； 且 该会话不能对其他表进行读、写操作。
			--即如果给A表加了读锁，则当前会话只能对A表进行读操作。

		会话1（其他会话）：
			select * from tablelock;   --读（查），可以
			delete from tablelock where id =1 ; --写，会“等待”会话0将锁释放


		会话1（其他会话）：
			select * from emp ;  --读（查），可以
			delete from emp where eno = 1; --写，可以
			结论2：
			--总结：
				会话0给A表加了锁；其他会话的操作：a.可以对其他表（A表以外的表）进行读、写操作
								b.对A表：读-可以；  写-需要等待释放锁。
		释放锁: unlock tables ;



	===加写锁：
		会话0：
			lock table tablelock write ;
	
			当前会话（会话0） 可以对加了写锁的表  进行任何操作（增删改查）；但是不能 操作（增删改查）其他表
		其他会话：
			对会话0中加写锁的表 可以进行增删改查的前提是：等待会话0释放写锁

MySQL表级锁的锁模式
MyISAM在执行查询语句（SELECT）前，会自动给涉及的所有表加读锁，
在执行更新操作（DML）前，会自动给涉及的表加写锁。
所以对MyISAM表进行操作，会有以下情况：
a、对MyISAM表的读操作（加读锁），不会阻塞其他进程（会话）对同一表的读请求，
但会阻塞对同一表的写请求。只有当读锁释放后，才会执行其它进程的写操作。
b、对MyISAM表的写操作（加写锁），会阻塞其他进程（会话）对同一表的读和写操作，
只有当写锁释放后，才会执行其它进程的读写操作。



分析表锁定：
	查看哪些表加了锁：   show open tables ;  1代表被加了锁
	分析表锁定的严重程度： show status like 'table%' ;
			Table_locks_immediate :即可能获取到的锁数
			Table_locks_waited：需要等待的表锁数(如果该值越大，说明存在越大的锁竞争)
	一般建议：
		Table_locks_immediate/Table_locks_waited > 5000， 建议采用InnoDB引擎，否则MyISAM引擎

	

（2）行表（InnoDB）
create table linelock(
id int(5) primary key auto_increment,
name varchar(20)
)engine=innodb ;
insert into linelock(name) values('1')  ;
insert into linelock(name) values('2')  ;
insert into linelock(name) values('3')  ;
insert into linelock(name) values('4')  ;
insert into linelock(name) values('5')  ;


--mysql默认自动commit;	oracle默认不会自动commit ;

为了研究行锁，暂时将自动commit关闭;  set autocommit =0 ; 以后需要通过commit


	会话0： 写操作
		insert into linelock values(	'a6') ;
	   
	会话1： 写操作 同样的数据
		update linelock set name='ax' where id = 6;

	对行锁情况：
		1.如果会话x对某条数据a进行 DML操作（研究时：关闭了自动commit的情况下），则其他会话必须等待会话x结束事务(commit/rollback)后  才能对数据a进行操作。
		2.表锁 是通过unlock tables，也可以通过事务解锁 ; 行锁 是通过事务解锁。

		

	行锁，操作不同数据：
	
	会话0： 写操作
	
		insert into linelock values(8,'a8') ;
	会话1： 写操作， 不同的数据
		update linelock set name='ax' where id = 5;
		行锁，一次锁一行数据；因此 如果操作的是不同数据，则不干扰。


	行锁的注意事项：
	a.如果没有索引，则行锁会转为表锁
	show index from linelock ;
	alter table linelock add index idx_linelock_name(name);

	
	会话0： 写操作
		update linelock set name = 'ai' where name = '3' ;
		
	会话1： 写操作， 不同的数据
		update linelock set name = 'aiX' where name = '4' ;
	

	
	会话0： 写操作
		update linelock set name = 'ai' where name = 3 ;
		
	会话1： 写操作， 不同的数据
		update linelock set name = 'aiX' where name = 4 ;
		
	--可以发现，数据被阻塞了（加锁）
	-- 原因：如果索引类 发生了类型转换，则索引失效。 因此 此次操作，会从行锁 转为表锁。

	b.行锁的一种特殊情况：间隙锁：值在范围内，但却不存在
	 --此时linelock表中 没有id=7的数据
	 update linelock set name ='x' where id >1 and id<9 ;   --即在此where范围中，没有id=7的数据，则id=7的数据成为间隙。
	间隙：Mysql会自动给 间隙 加索 ->间隙锁。即 本题 会自动给id=7的数据加 间隙锁（行锁）。
	行锁：如果有where，则实际加索的范围 就是where后面的范围（不是实际的值）

	
	如何仅仅是查询数据，能否加锁？ 可以   for update 
	研究学习时，将自动提交关闭：
		set autocommit =0 ;
		start transaction ;
		begin ;
	 select * from linelock where id =2 for update ;

	通过for update对query语句进行加锁。

	行锁：
	InnoDB默认采用行锁；
	缺点： 比表锁性能损耗大。
	优点：并发能力强，效率高。
	因此建议，高并发用InnoDB，否则用MyISAM。

	行锁分析：
	  show status like '%innodb_row_lock%' ;
		 Innodb_row_lock_current_waits :当前正在等待锁的数量  
		  Innodb_row_lock_time：等待总时长。从系统启到现在 一共等待的时间
		 Innodb_row_lock_time_avg  ：平均等待时长。从系统启到现在平均等待的时间
		 Innodb_row_lock_time_max  ：最大等待时长。从系统启到现在最大一次等待的时间
		 Innodb_row_lock_waits ：	等待次数。从系统启到现在一共等待的次数
12.主从复制  （集群在数据库的一种实现）
	windows:mysql 主
	linux:mysql从

	安装windows版mysql:
		如果之前计算机中安装过Mysql，要重新再安装  则需要：先卸载 再安装
		先卸载：
			通过电脑自带卸载工具卸载Mysql (电脑管家也可以)
			删除一个mysql缓存文件C:\ProgramData\MySQL
			删除注册表regedit中所有mysql相关配置
			--重启计算机
			
		安装MYSQL：
			安装时，如果出现未响应：  则重新打开D:\MySQL\MySQL Server 5.5\bin\MySQLInstanceConfig.exe

		图形化客户端： SQLyog, Navicat


		如果要远程连接数据库，则需要授权远程访问。 
		授权远程访问 :(A->B,则再B计算机的Mysql中执行以下命令)
		GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
		FLUSH PRIVILEGES;
	
		如果仍然报错：可能是防火墙没关闭 ：  在B关闭防火墙  service iptables stop 



	实现主从同步（主从复制）：图
		1.master将改变的数 记录在本地的 二进制日志中（binary log） ；该过程 称之为：二进制日志件事
		2.slave将master的binary log拷贝到自己的 relay log（中继日志文件）中
		3.中继日志事件，将数据读取到自己的数据库之中
	MYSQL主从复制 是异步的，串行化的， 有延迟
	
	master:slave = 1:n

	配置： 
	windows(mysql: my.ini)
	  linux(mysql: my.cnf)

	配置前，为了无误，先将权限(远程访问)、防火墙等处理：
		关闭windows/linux防火墙： windows：右键“网络”   ,linux: service iptables stop
		Mysql允许远程连接(windowos/linux)：
			GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
			FLUSH PRIVILEGES;
	

主机（以下代码和操作 全部在主机windows中操作）：
my.ini
[mysqld]
#id
server-id=1
#二进制日志文件（注意是/  不是\）
log-bin="D:/MySQL/MySQL Server 5.5/data/mysql-bin"
#错误记录文件
log-error="D:/MySQL/MySQL Server 5.5/data/mysql-error"
#主从同步时 忽略的数据库
binlog-ignore-db=mysql
#(可选)指定主从同步时，同步哪些数据库
binlog-do-db=test	

windows中的数据库 授权哪台计算机中的数据库 是自己的从数据库：	
 GRANT REPLICATION slave,reload,super ON *.* TO 'root'@'192.168.2.%' IDENTIFIED BY 'root';
 flush privileges ; 

	查看主数据库的状态（每次在左主从同步前，需要观察 主机状态的最新值）
		show master status;  （mysql-bin.000001、 107）

从机（以下代码和操作 全部在从机linux中操作）：


my.cnf
[mysqld]
server-id=2
log-bin=mysql-bin
replicate-do-db=test

linux中的数据 授权哪台计算机中的数控 是自己的主计算机
CHANGE MASTER TO 
MASTER_HOST = '192.168.2.2', 
MASTER_USER = 'root', 
MASTER_PASSWORD = 'root', 
MASTER_PORT = 3306,
master_log_file='mysql-bin.000001',
master_log_pos=107;
	如果报错：This operation cannot be performed with a running slave; run STOP SLAVE first
	解决：STOP SLAVE ;再次执行上条授权语句


开启主从同步：
	从机linux:
	start slave ;
	检验  show slave status \G	主要观察： Slave_IO_Running和 Slave_SQL_Running，确保二者都是yes；如果不都是yes，则看下方的 Last_IO_Error。
本次 通过 Last_IO_Error发现错误的原因是 主从使用了相同的server-id， 检查:在主从中分别查看serverid:  show variables like 'server_id' ;
	可以发现，在Linux中的my.cnf中设置了server-id=2，但实际执行时 确实server-id=1，原因：可能是 linux版Mysql的一个bug，也可能是 windows和Linux版本不一致造成的兼容性问题。
	解决改bug： set global server_id =2 ;

	stop slave ;
	 set global server_id =2 ;
	start slave ;
	 show slave status \G

	演示：
	主windows =>从

	windows:
	将表，插入数据  
	观察从数据库中该表的数据


数据库+后端

	spring boot（企业级框架,目前使用较多）  
		
	

		
	







