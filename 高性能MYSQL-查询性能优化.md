#查询性能优化
    分析慢查询：
    1. 确认应用程序是否在检索大量的超过需要的数据。者通常意味着访问了太多的行，有时候也可能是返回问了太多的列。</br>
    2. 确认MySQL服务器层是否在分析大量超过需要的数据行。

[MySQL查询过程](/Users/jingjie/Documents/markdown/images/高性能MYSQL/MySQL查询过程.jpg)

* 列表IN()的比较：MySQL将IN（）列表中的数据先进行排序，然后通过二分查找的方式来确定列表中的值是否满足条件，事件复杂度为O(logn)，等价地转化为OR查询，时间复杂度为O(n)。
* 关联查询（JOIN）：使用循环嵌套的方式 eg:
[内连接JOIN查询过程](/Users/jingjie/Documents/markdown/images/高性能MYSQL/INNER-JOIN查询过程.jpg)</br>
[外连接JOIN查询过程](/Users/jingjie/Documents/markdown/images/高性能MYSQL/OUTTER-JOIN查询过程.jpg)
* 排序优化：
    * 传输单次排序：现读取查询所需要的所有列，然后在根据给定列进行排序，最后返回排序结果。（两次传输排序：先读取行指针和需要排序的字段，对其进行排序，然后再根据排序结果去读所需要的数据行。已被弃用）
    * 关联查询排序
        - 如果ORDER BY的子句中所有的列都来自关联的第一个表，那么MySQL在关联处理第一个表的时候就进行文件排序。在EXPLAIN中可看到EXTRA字段会有“Using filesort”。
        - 除此之外的所有情况，MySQL会先将关联的结果存放到一个临时表中，然后在所有的关联结束后，再进行文件排序。再EXTRA字段中可以看到“Using temporary；Using filesort”。
* 最小值优化：MySQL对MIN()的查询做的并不好，例如：

        select MIN（actor_id）from actor where first_name = 'penelope';
&emsp;&emsp;因为在first_name上没有索引，MySQL会进行全表扫描，如果mysql能进行主键扫描，理论上读到的第一个满足条件的记录 就是最小值，因为主键是严格按照actor_id字段的大小顺序排列的。y优化方法：

    select actor_id from actor use index(PRIMARY) where first_name = 'PENELPE' limit 1;
* COUNT()的优化（针对MyISAM引擎）,没有where条件的COUNT（*）非常快，因为mysql是利用存储引擎的特性直接获得这个值：

        如需找到id>5的城市的数量：select count（*）from city where id > 5;
        这时会扫描几乎全表的数据，如果改为：
        select (select count(*) from city) - count(*) from city where id <= 5;
        这时只会扫描5条数据。（针对 myisam引擎）
* 关联查询优化
    - 确保ON或USING子句的列上有索引。顺序：当表A,B用列C关联时，如果优化器的关联顺序是B,A就只需要在A上加索引。一般来说，只需要在关联顺序的第二个表上创建索引
    - 确保group by和order by中的表达式只涉及到一个表中的列，这样才有可能使用索引。
* union的限制（6.5.2）：有时MySQL无法将限制条件从外层“下推”到内层，这使得原本能够限制部分返回结果的条件无法应用到内层查询的优化上。eg：如果想将两个子查询的查询结果联合起来，然后再取前20条记录，那么mysql会将两个表都存放到同一个临时表中，然后再取出前20条数据:

        (select id from id order by id) union all (select contract_id from contract_party order by id) limit 20
这条查询会取出contract_base_info的359条数据和contract_party中的718条数据放入一个临时表中，然后从临时表中取出前20条数据。
可以改写成:

        (select id from id order by id limit 20) union all (select contract_id from contract_party order by id limit 20) limit 20
这只会从每个表中取出20条数据，从而减少临时表的数据量。
* limit分页优化：在偏移量特别大的时候（limit 10000，20），MYSQL会查询10020条数据，最后只返回
20条数据。这样做的代价非常高，如果所有的页面被访问的频率相同，那么这样的查询平均需要访问半个表的数据量。优化此类查询分页查询的一个最简单的办法就是尽可能的使用覆盖索引扫描，而不是查询所有的列。eg：

        select film_id,description from film order by title limit 50, 5
  在表比较大时，最好将查询改为：

        select film.film_id, film.description from film INNER JOIN (select film_id from film order by title limit 50, 5) as lim using(film_id);
  他让mysql扫描尽可能少的页面，获取需要访问的记录后在根据关联列查询需要的所有列。
* limit 和 offset的问题，其实是offset的问题，他会导致MYSQL扫描大多数不需要的行然后抛弃掉。可以使用标签记录上次取数据的位置，那么下次就可以直接从该书签的位置扫描数据。eg:

        select * from rental order by rental_id desc limit 20;
    如果上次返回的主键为16049到16030的记录，那么下次就可以从16030这个点开始查询。

        select * from rental where rental_id < 16030 order by rental_id desc limit 20;
    该技术的好处是不论翻页到多么后面，性能都会很好。

    将limit查询转化为已知位置的查询，让mysql通过范围扫描获得对应的结果。例如：如果列上有索引，并且预先计算出了边界值：

        select id，contract_no from contract_base_info where id between 50 and 69 order by id;
* 除非确实服务器消除重复的行，否则就一定使用UNION ALL 而不是 UNION，应为UNION会给临时表加上distinct选项，这会导致整个临时表做唯一性检查，代价很高。
* 自定义变量
    - 使用自定义变量 避免查询刚刚更新的数据。如果想要在更新行的同时 获得 改行的数据。eg：

            update t1 set lastupdate = NOW() where id = 1;
            select lastupdate from t1 where id = 1;
        使用变量：

            update t1 set lastupdate = NOW() where id = 1 and @now := NOW();
            select @now;
        虽然看起来是两个查询，需要两次网络来回，但这里的第二个查询无需访问数据表，所以很快。
    - 统计更新和插入的数量，当使用insert ON duplicate key uodate的时候，想知道到底插入了多少行，到底更新了多少行。

            insert into t1(c1, c2) values(1, 2), (2, 3), (3, 4) on duplicate key update c1 = values(c1) + (0 * (@x := @x + 1));
            <!-- select * from @x; -->
        会返回被更改行数的总数，不需要单独去统计这个值。
