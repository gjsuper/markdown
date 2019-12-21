# 极客时间--MySQL实战45讲--第37讲：什么时候使用内部临时表？

### union执行流程
有如下的语句：

    create table t1(id int primary key, a int, b int, index(a));
    delimiter ;;
    create procedure idata()
    begin
    declare i int;

    set i=1;
    while(i<=1000)do
    insert into t1 values(i, i, i);
    set i=i+1;
    end while;
    end;;
    delimiter ;
    call idata();
然后查询：

    (select 1000 as f) union (select id from t1 order by id desc limit 2);

结果为:
[union的explain结果](../imaged/mysql实战45讲/union的explain结果.png)
可知：
* 第二行的key=PROMARY表示用到了索引id
* 第三行的Extra字段，表示在对子查询的结果集做union的时候，使用了临时表（using temporary）。

这个语句的执行过程：
1. 创建一个内存临时表，这个临时表只有一个整形子弹f，并且f是主键字段
2. 执行第一个子查询语句，得到1000这个值，并放入临时表
3. 执行第二个子查询：
    1. 试图放入第一行id=1000。但由于1000这个值已存在，违反了主键约束，插入失败，继续执行
    2. 渠道第二行id = 999，插入临时表成功。
4. 从临时表取出数据，返回结果，并删除临时表，结果里包含1000和999

如果把union改成union all就没有去重的功能了，这样执行的时候，就一依次执行子查询，得到的结果直接作为结果集的一部分发给客户端，也就**不需要临时表了，因此在union里，临时表的功能就是用于去重**

### group by执行流程
    select id%10 as m, count(*) as c from t1 group by m;

[groupBy的explain结果](../images/mysql实战45讲/groupBy的explain结果.png)

从Extra字段可看出：
1. Using index：使用了覆盖索引a
2. Using temporary，使用了临时表
3. Using filesort，需要排序

这个语句的执行流程：
1. 创建内存临时表，表里有两个字段，主键m和字段c
2. 扫描表t1的索引a，依次取出叶子节点上的id值，计算id % 10的结果，记为x
    1. 如果临时表中没有主键为x的行，就插入一个记录（x,1）
    2. 如果有，就将x这一行的c值加一
3. 遍历完成后，再根据字段m做排序，返回客户端

[groupBy的结果](../images/mysql实战45讲/groupBy的结果.png)

如果不要求对结果进行排序，可使用order by null：

    select id%10 as m, count(*) as c from t1 group by m order by null;

[groupby+orderbynull的结果](../images/mysql实战45讲/groupby+orderbynull的结果.png)

由于表t1中的id是从1开始的，因此第一行是id=1，扫描到id=10时才插入m=0这行，因此最后一行是m=0。

这里所有数据都可以让放在内存里，因此全程只使用了内存临时表。但是这个内存临时表的大小是由tmp_table_size来控制的，默认是16M，改一下：

    set tmp_table_size=1024;
    select id%100 as m, count(*) as c from t1 group by m order by null limit 10;
这时内存里放不下这一百行数据，就会把内存临时表转成磁盘临时表，由于磁盘临时表默认使用的引擎是InnoDB，主键索引时有序的，因此，返回的结果也是有序的，如下图：

[groupby+orberbynull+磁盘临时表](../images/mysql实战45讲/groupby+orberbynull+磁盘临时表.png)

如果这个表很大的话，那么这个查询可能需要的临时表会占用大量的磁盘空间

### group by优化方法--索引
不管是内存临时表还是磁盘临时表，group by都会构造一个具有唯一索引的表，代价比较高，如果表的数据量比较大的话，语句就会比较慢。

思考一下：为什么group by语句需要临时表

group by的逻辑语义，是统计不同值出现的个数，但每一行的id%100是无序的，所以需要一个临时表，来记录并统计结果。如果扫描过程中的数据是有序的，是不是就简单了？

[groupby优化-有序输入](../images/mysql实战45讲/groupby优化-有序输入.jpg)

这样group by的时候，就只需要从左到右，顺序扫描，依次累加，这样扫描结束的时候，就可以拿到group by的结果，不需要临时表，而InnoDB的索引就可以满足输入有序的条件。

MySQL5.7支持generate column机制，用来实现列数据的关联更新，因此创建一个新列z，并加索引（5.7之前可以通过建立普通列和索引来解决这个问题）

    alter table t1 add column z int generated always as(id % 100), add index(z);
这样索引z上的数据就是有序的了。上面的group by语句可改为：

    select z, count(*) as c from t1 group by z;

[groupby优化--关联更新](../images/mysql实战45讲/groupby优化--关联更新.png)

从extra字段看到，这个语句不再需要临时表了，也不需要进行排序了。

### group by优化方法--直接排序

如果不适合创建索引，还是只有老老实实的排序，那么如何优化？

如果明明知道，一个group by语句的数据量太大，却还是要按照”先放到内存临时表，插入一部分数据后，发现内存临时表不够用了再转换磁盘临时表”就有点傻。那么有没有直接走磁盘临时表的方法？

在group by语句中加入SQL_BIG_RESULT就可以告诉优化器：这个语句的数据量较大，请直接使用磁盘临时表。而磁盘临时表示B+树，存储效率不如数组高，既然优化器知道了数据量大，从磁盘考虑，就直接使用数组来存储

    select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
这个语句的流程：
1. 初始化sort_buffer，确定放入一个整形字段，记为m；
2. 扫描表t1的索引a，依次取出id值，将id%100放入sort_buffer中
3. 扫描完后，对sort_buffer的字段m排序（如果内存不够，就采用磁盘临时文件辅助排序）
4. 排序完成后，就得到一个临时数组。

[SQL_BIG_RESULT执行流程](../images/mysql实战45讲/SQL_BIG_RESULT执行流程.png)

[SQL_BIG_RESULT的explain结果](../images/mysql实战45讲/SQL_BIG_RESULT的explain结果.png)
可以看到，这个语句没有使用内存临时表，而是直接使用了排序算法

问：MySQL为什么使用内部临时表？
1. 如果语句执行过程可以一边读数据，一边直接得到结果，是不需要额外内存的，否则就需要额外的内存，来保存中间结果
2. join_buffer是无序数组,sort_buffer是有序数组，临时表是二维表结构
3. 如果执行逻辑需要用到二维表特性，就会优先考虑使用临时表，例如union需要用到唯一索引约束，group by还需要用到拎一个字段来存累计字段
