# 极客时间--MySQL实战45讲--第34讲：到底可不可以使用join？

有两个数据表t1和t2，表结构一样

        CREATE TBALE `t2` （id int(11),
        `a` int(11),
        `b` int(11),
        PRIMARY KEY (`id`),
        KEY `a` (`a`)）;

        //在t2里插入1000条数据

        ctrate table t1 like t2；
        insert into t1（select * from t2 where id <= 100）

做join查询

        select * from t1 straight_join t2 on (t1.a = t2.a)

流程：
1. 从表1中读一行数据R
2. 从数据行R中，取出a字段到表t2里去查询
3. 取出表t2中满足条件的行，跟R组成一行，作为结果集的一部分
4. 重复步骤1到3，直到表t1的末尾循环结束

## 被驱动表有索引
这个过程类似循环嵌套，并且可以用上被驱动表的索引，所以称之为Index Nested-Loop Join。整个过程扫描的行数是200行。假设驱动表的函数是N，对于每一行，都要到被驱动表上匹配一次，整个执行过程中，近似复杂度是N+N*2logM。

## 被驱动表无索引：Simple Nested-Loop Join和Block Nested-Loop Join

将sql改为：

        select * from t1 straight_join t2 on (t1.a = t2.b);
由于t2的字段b上没有索引，因此每次到t2匹配的时候，就要做一次全表扫描。这样这个SQL请求要扫描 100 * 1000 = 10万次。所以MySQL没有使用这个Simple Nested-Loop Join 算法，而采用了Block Nested-Loop Join

他的流程：
1. 把表t1的数据读入线程内存join_buffer中，由于我们这个语句中写的是select * ，因此是把整个表t1放入了内存
2. 扫描表t2，把表t2中的每一行取出来，跟join_buffer中的数据对比，满足join条件的，作为结果集的一部分返回

[BlockNested-LoopJoin流程](../images/mysql实战45讲/BlockNested-LoopJoin流程.jpg)

这个过程对表t1和t2都做了全表扫描，总共扫描了1100次，比较了10 * 1000 = 10万次，Simple Nested-Loop 扫秒行数是10万次，两者的时间复杂度是一样的，但Block Nested-Loop Join在内存中比较，速度快很多

**如果内存中放不下表t1的所有数据的话，就分段放**，过程就变成了：
1. 扫描表t1，顺序读取表t1的数据放入内存join_buffer中，放了一部分数据join_buffer就满了，继续第二步
2. 扫描表t2，把表t2中的每一行取出来，跟join_buffer中的数据对比，满足join条件的，作为结果集的一部分返回
3. 情况join_buffer
4. 继续扫描表t1，继续第二步

第一个问题：能不能使用join？
1. 如果使用Index Nested-Loop Join，也就是可以用上被驱动表上的索引，没问题
2. 如果使用Block Nested-Loop Join，扫描行数就会过多。尤其在大表上做join操作，可能会扫描被驱动表很多次，会占用大量系统资源。所以这时join操作尽量避免。

第二个问题：如果使用join，应该选择大表还是小表作为驱动表？

1. 如果使用Index Nested-Loop Join，应该选择小表
2. 如果使用Block Nested-Loop Join：
    1. join_buffer足够大时，是一样的
    2. join_buffer不够大时，应该选择小表
所以，结论是：**总是使用小表，在决定哪个表作为驱动表时，应该是两个表按照各自的条件过滤，过滤完成后，计算参与join的各字段总数据量，数据量小的那个表就是小表**

举例：

        select t1.b,t2.* from t1 straight_join t2 on (t1.b=t2.b) where t2.id <= 100
        select t1.b,t2.* from t2 straight_join t2 on (t1.b=t2.b) where t2.id <= 100
表t1和t2都只是100行数据参加join，但是：
* 表t1只查询字段b，因此如果把t1放到join_buffer中，则join_buffer中只需要放入b这一个字段的值
* 表t2要查询所有的字段，因此如果把t2放到join_buffer中，就需要放入三个字段id、a和b。因此应选择t1作为驱动表。
