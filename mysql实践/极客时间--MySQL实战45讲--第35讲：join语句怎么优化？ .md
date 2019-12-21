# 极客时间--MySQL实战45讲--第35讲：join语句怎么优化？

## Multi-Range Read优化（MRR）：尽量使用顺序读盘
回表时，由于二级索引的顺序和主键的顺序可能不一样，会导致回表时id的值变成随机的，那么会出现随机访问，性能会相对较差。虽然回表时是“按行查”这个机制不能，但通过调整查询顺序还是能够加速的。

**因为大多数的数据都是按照主键递增的顺序插入的，所以可以认为，如果按照主键顺序递增的顺序查询，对磁盘的读会接近顺序读，能都提升性能**，所以MRR优化的思路：
1. 根据二级索引，查到id，将id值放入read_rnd_buffer
2. 将read_rnd_buffer中的id进行递增排序
3. 排序后的id数组，依次到主键id索引查询返回结果

如果read_rnd_buffer放满了，就会先执行2和3，然后清空read_rnd_buffer，之后继续找二级索引的下个记录。

MRR能够提升性能的核心在于，这条查询语句在索引a上做的是一个范围查询，可以得到足够多的主键id。这样通过排序以后，再去主键索引查询数据，才能体现出“顺序性”的优势。

## Batched Key Access（BKA）：对NLJ的优化
对于NLJ：从驱动表t1，一行一行取出a的值，再到被驱动表t2去做join，对于表t2来说，每次都是匹配一个值。这时，MRR的优势就用不上。优化的方法就是从表t1里一次性多拿些行出来，一起传给t2。所以把表t1的数据取出来，先放到一个临时内存（join_buffer）

## NBL算法性能问题
1. 可能会扫描多次被驱动表，占用磁盘资源
2. 判断join条件需要执行M * N次对比，如果是大表会占用很多的CPU资源
3. 可能会导致Buffer Pool的热数据被淘汰，影响内存命中率

## BNL转BKA
一些情况下，可以直接在被驱动表建索引，直接转换成BKA算法。
如果是冷表或者数据量较小，建索引不划算，但不建索引的话比较次数又太多，可以建立一个临时表，给临时表加索引，让表t1和临时表做join操作，转成BKA。大致思路是：

1. 把表t2中满足条件的数据临时放在临时表tmp_t中
2. 为了让join使用BKA算法，给临时表tmp_t的字段b加上索引
3. 让表t1和表tmp_t做join操作

        create tmporary table tmp_t(id int primary key, a int, b int, index(b)) engine = innodb;

        insert into tmp_t select * from t2 where b >= 1 and b <= 2000;
        select * from t1 join tmp_t on (t1.b = tmp_t.b);
## 扩展 -hash join

如果join_buffer里维护的不是一个无需数组，而是一个hash表的话，那么就不是10亿次判断，而是100万次hash查找了，这样速度就快多了。但MySQL的优化器和执行器不支持hash join。我们可以自己在业务端实现：

1. select * from t1；取得表t1的所有数据1000行数据，存入业务端的hash表里
2. select * from t2 where b >= 1 and b <= 2000；获取表t2的数据满足条件的2000行数据
3. 把这2000行数据，一行一行的取出来，到hash表里查找匹配的数据。

理论上，这种方式比临时表要快。
