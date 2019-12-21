# 极客时间--MySQL实战45讲--第38讲：都说InnoDB好，那还要不要使用Memory引擎？

### 内存表的数据结构

    create table t1(id int primary key, c int) engine=Memory;
    create table t2(id int primary key, c int) engine=innodb;
    insert into t1 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
    insert into t2 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
然后执行查询，结果为：
[两种引擎select的结果](../images/mysql实战45讲/两种引擎select的结果.png)

两者的返回结果不一样，因为两者的索引结构不一样：
1. InnoDB索引采用B+树，select * 时，就从左到右扫描，所以是有序的
[InnoDB引擎索引结构](../images/mysql实战45讲/InnoDB引擎索引结构.png)
2. 而memory引擎的数据和索引是分开的
[memory引擎索引](../images/mysql实战45讲/memory引擎索引.png)
内存表的数据部分以数组的形式单独存放，而逐渐id在索引里，存放的是每个数据的位置。主键id是hash索引，可以看到索引上的key并不是有序的，select *的时候，全表扫描按顺序扫描这个数组，因此0最后被读到

比较：
1. InnoDB表的数据总是有序存放的，而内存表的数据顺序就是按照写入顺序存放的
2. 当数据文件有空洞的时候，InnoDB表在插入新数据的时候，为了保证数据有序性，只能在固定位置写入新值，而内存表找到空位置就可以插入新值
3. 当数据位置发生变化的时候，InnoDB表主需要修主键索引，而内存表需要修改所有索引
4. InnoDB引擎走主键索引会走一次索引查找，走普通索引会走两次索引查找。而内存表没有这个区别，所有的索引地位都是一样的
5. InnoDB支持变长数据类型，不同数据行的长度可能不同，而内存表不支持Blob和Text字段，即使定义了varchar，实际也会被当做char，也就是使用固定长度字符串来存储，因此每行的数据长度是一样的

对内存表，每个数据行删除之后，空出来的位置都可以被接下来要插入的数据复用

### hash索引和B-Tree索引
内存表也可以支持B-Tree索引

    alter table t1 add index a_btree_index using btree (id);

[内存表的B-Tree索引](../images/mysql实战45讲/内存表的B-Tree索引.png)

[使用B-Tree索引和hash索引结果对比](../images/mysql实战45讲/使用B-Tree索引和hash索引结果对比.png)

执行select * from t1 where id < 5的时候，优化器会选择B-Tree索引，索引返回是有序的。使用force index强行使用主键id这个索引，id=0这一行就在结果集的最末尾了

### 生产环境不建议使用内存表
1. 内存表支持持行锁，并发度较低
2. 数据持久性问题
