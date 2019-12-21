# 极客时间--MySQL实战45讲--第44讲:join的写法、group by和distinct

### join的写法
问题:
1. 如果用left join，左边的表一定是驱动表吗？
2. 如果两个表的join包含多个条件的等值匹配，是都要写到on里面，还是只把一个条件写到on里面，其他条件写到where部分？

造表：

    create table a(f1 int, f2 int, index(f1))engine=innodb;
    create table b(f1 int, f2 int)engine=innodb;
    insert into a values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6);
    insert into b values(3,3),(4,4),(5,5),(6,6),(7,7),(8,8);

两种写法：

    select * from a left join b on(a.f1=b.f1) and (a.f2=b.f2); /*Q1*/
    select * from a left join b on(a.f1=b.f1) where (a.f2=b.f2);/*Q2*/

结果：
[两个join的查询结果](../images/mysql/实战45讲/两个join的查询结果.png)

* 语句1的返回结果中，表a即使没有满足匹配条件的记录，查询结果中也会返回一行，并将表b的各个字段填成null
* 从逻辑上可以这么理解，由于表b没有匹配的字段，结果集里面b.f2的值是空，不满足where部分的条件判断，因此不能作为结果集的一部分

[Q1的explain结果](../images/mysql实战45讲/Q1的explain结果.png)

* 表a是驱动表，b是被驱动表
* 由于表b上f1没有索引，所以使用的是Block Nexted Loop Join算法

流程：
1. 把表a的内容读入join_buffer中，因为是select *，所以把f1和f2两种字段都放进去了。
2. 顺序扫描表b，对于每一行数据，判断join条件是否满足，满足条件的记录，作为结果集的一行返回。如果语句中有where子句，需要先判断where部分满足条件后，再返回
3. 表b扫描完以后，对于没有配的表a的行，把剩余字段不上null，再放入结果集。

[Q2的explain结果](../images/mysql实战45讲/Q2的explain结果.png)

这里的驱动表是b，如果一条join语句的Extra字段什么都没写，就表示使用的是Index Nested-Loop Join算法

所以，Q2的执行流程是：顺序扫描表b，每一行用b.f1到表a中去查，匹配到记录后判断a.f2=b.f2是否满足，满足条件的话就作为结果集的一部分返回

一个背景：在MySQL里，NULL跟任何值比较（等值和不等值判断）的结果都是NULL。

因此Q2的语句里where a.f2 = b.f2就表示，查询结果里面不会包含b.f2是NULL的行，这样left join的语义就是“找到这两个表里面，f1和f2对于相同的行。对于表a中存在，而表b中匹配不到的行，就放弃”

因此：
1. 使用left join，左边的表不一定是驱动表
2. 如果需要left join的语义，就不能把被驱动表的子弹放在where条件里做等值判断或不等值判断，必须写在on里

而下面这两条语句：

    select * from a join b on(a.f1=b.f1) and (a.f2=b.f2); /*Q3*/
    select * from a join b on(a.f1=b.f1) where (a.f2=b.f2);/*Q4*/
用show warnings可以看到，他们都会被改写成：

    select * from a join b where (a.f1=b.f1) and (a.f2=b.f2);

### distinct 和 group by的性能

如果只是去重，不需要聚合函数，distinct和group by哪种效率更高？

如果表t没有索引，下面这两种语句的性能谁好：

    select a from t group by a order by null;
    select distinct a from t;
首先，这种group by不是标准SQL的写法，标准的group by应该是加一个聚合函数，比如：

    select a,count(*) from t group by a order by null;
没有count(*)以后，逻辑就变成了：按照字段a做分组，相同的a的值只返回一行，其实这就是distinct的语义，这时group by和distinct的语义和执行流程是一样的：
1. 创建一个临时表，临时表有一个字段a，并且在a上黄建了一个唯一索引
2. 遍历表t，一次取出数据插入临时表中：
    1. 如果发现有冲突，就跳过
    2. 否则插入成功
3. 遍历完后，把结果集返回客户端     


因此，对于重复的a的值，只返回主键id最小的那个

### 备库自增主键的问题
在binlog_format=statement时，语句A先获取id=1，然后语句2再获取id=2，在写入binlog的时候，语句b先提交，如果binlog回放，会不会发生语句b的id=1，语句a的id=2？

不会！！！

因为，在insert语句之前，还有一句SET INSERT_ID = 1。意思是这个线程里下一次需要用到自增值的时候，不论当前的自增值id是多少，固定使用1这个值

[insert的binlog](../images/mysql实战45讲/insert的binlog.png)

    SET INSERT_ID=2;
    语句 B；
    SET INSERT_ID=1;
    语句 A；
