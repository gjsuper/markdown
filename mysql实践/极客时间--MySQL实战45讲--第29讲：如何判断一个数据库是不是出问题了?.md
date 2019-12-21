# 极客时间--MySQL实战45讲--第29讲：如何判断一个数据库是不是出问题了?

* select 1 判断：
select 1 成功返回，只能说明这个库的进程还在，并不能说明主库没有问题。

    [select1](../images/mysql实战45讲/select1.png)

 从上面可以看到，select 1是能执行成功的，但是查询表t的语句会被堵住，也就是说，select 1是检测不出问题的

 在innodb中，innodb_thread_concurrency这个参数默认值0，表示不限制并发数量，建议把这个设置为64-128之间的值。这个参数指定是并发查询数。

 **并发连接和并发查询**</br>
这俩并不是一个概念，在show process list看到的是并发连接数，达到几千个影响并不大。而“当前正在执行”的语句，才是我们所说的并发查询。

**在线程进入所等待后，并发线程的计数会减一（等待行锁、间隙锁的线程不计算在这个参数内）**
这么做是因为进入锁等待的线程已经不吃CPU了，而且也是为了避免死锁，比如，假设占用并发线程数的话：
1. 线程1执行begin；update set c = c+ 1where id = 1；启动了事务，然后保持这个状态。这个时候，线程处于空闲状态，不算在并发线程里面
2. 线程2~129都执行update set c = c+ 1 where id = 1；进入锁等待，这样就有128个线程进入锁等待
3. 如果处于锁等待的计数器不减一，Innodb就会认为线程数用满了，会阻止其他语句进入引擎，这样线程1不能提交事务。而另外128个线程又处于锁等待，整个系统就堵住了

在这个例子中，同时执行的语句超过了innodb_tread_concurrency的值，这时候系统其实已经不行了，但通过了select1检测，所以需要改一下select 1的逻辑

* 查表判断
可以通过建一个表，命名为health_check，里面只放一行数据，然后定期执行：

        mysql> select * from mysql.health_check;
这虽然检测出由于并发线程过多导致的数据库不借用，但存在一个问题：空间满了以后，更新事物要写binlog，一旦binlog所在空间占用达到100%，则所有的更新语句和事务提交的commit就会被堵住，但是，是可以正常读数据的

* 更新判断
基于上面的问题。采用更新字段来判断，放一个有意义的字段，常见的做法是放一个TIMESTAMP，用来表示最后一次最后一次执行检测的时间：

        mysql> update mysql.health_check set t_modified=now();
节点检测的范围应包括主库和备库，一般主库he备库是双M结构，所以在备库执行的检测命令，也会发回到主库。如果主备库同时更新，可能出现行冲突，会导致主备同步停止。所有检测表中不能只有一行数据：

    mysql> CREATE TABLE `health_check` (
      `id` int(11) NOT NULL,
      `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB;

    /* 检测命令 */
    insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
由于mysql规定了主库和备库的server_id必须不同（否则创建主备关系时会报错），这样就可以保证主备库各自的检测命令不会发生冲突

* 内部统计
针对磁盘利用率的问题，可以用内部每次IO请求的时间来判断数据库是否出问题了。

MySQL5.6之后提供了performance_schema库，就在file_summary_by_event_name表里统计了每次IO请求的时间。
[performance_schema.file_summary的一行](../images/mysql实战45讲/performance_schema.file_summary的一行.png)
