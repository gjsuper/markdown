# 极客时间--MySQL实战45讲--第45讲:自增id用完怎么办？

看看MySQL里面的几种自增id，当他们达到上限后会有什么情况？

### 表定义自增id
表定义的自增id达到上限后，再申请一个id时，得到的值保持不变，验证：

    create table t(id int unsigned auto_increment primary key) auto_increment=4294967295;
    insert into t values(null);
    // 成功插入一行 4294967295
    show create table t;
    /* CREATE TABLE `t` (
      `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=4294967295;
    */

    insert into t values(null);
    //Duplicate entry '4294967295' for key 'PRIMARY'

可以看到，第一个数据插入完以后，表的AUTO_INCREMENT没有改变，导致第二个语句插入时获得了相同的自增id，报出主键冲突    
int的上限不是一个很大的数，如果可能，应该将其创建为8个字节的bigint unsigned

### InnoDB系统自增row_id

如果创建的InnoDB表没有指定主键，InnoBDA会给你创建一个不尅案的，长度为6个字节的row_id，InnoDB维护了一个全局的dict_sys.row_id，所有无主键的InnoDB表，每插入一行数据，都要取单花钱的dict_sys.row_id值作为要插入数据的row_id，然后把dict_sys.row_id加1


1. row_id写入表中的值范围，从0 到 2^48-1;
2. 当dict_sys.row_id=2^48时，如果再有插入数据的行来申请row_id，拿到后再取最后一个6字节的话就是0。如果申请到id=N后，表中存在row_id=N的行，新数据就会覆盖旧数据

一般情况下，可靠性优先于可用性，因此还是应该在InnoDB表中主动创建自增主键，否则覆盖数据意味着数据丢失，影响的是数据可靠性。

### Xid
在第15章中，redolog和binlog相配合的时候，提到他们有一个共同的字段叫做Xid，他在MySQL中是用来对应事务的。

MySQL维护了一个纯内存变量：global_query_id,每次重启都会清零，这个值是8个字节，每次达到极限后都会清零。每次执行语句的时候就将他赋值给Query_id,然后这个值加一，如果当前语句是这个事务执行的第一条语句，那么MySQL还会同时把query_id赋值给这个事务的xid。所以同一个数据库中，不同事务的Xid可能相同，但MySQL重启后会生成新的binlog文件，这就保证了，同一个binlog文件里，Xid一定是唯一的。

出现相同Xid的场景：
1. 执行一个事务，假设Xid是A；
2. 接下来执行2^64次查询，让global_query_id回到A
3. 再启动一个事物，这个事务的Xid也是A。不过2^64太大了，这种可能性仅存在与理论中

### InnoDB trx_id
Xid是server层维护的，InnoDB内部使用Xid，就是为了能够在InnoDB事务和server之间做关联，但InnoDB自己的trx_id，是另外维护的。

trx_id就是事务id。innodb内部维护了一个max_trx_id全局变量，每次需要申请一个新的trx_id,就获得max_trx_id的当前值，然后将max_trx_id加1,然鹅，实验的时候发现不止加1,因为：
1. update和delete语句除了事务本身，还涉及到标记删除旧数据，也就是要把数据放到purge队列里等待后续物理操作，这个操作也会把max_trx_id加1，因此一个事务至少是加2
2. InnoDB的后台操作，比如表的索引信息统计类操作，也是会启动内部事务的，因此，trx_id并不是按照加1递增的

只读事务不分配trx_id，只读事务的trx_id是通过当前事务的trx比那里的物理地址转换成整数再加上2^48得到的。只读事务不分配trx_id的好处：
1. 减小事务视图里活跃事务数组的大小。因为当前正在运行的只读事务，是不影响数据可见性判断的，所以在创建事务的一致性视图时，InnoDB就只需要拷贝读写事务的trx_id。
2. 减小trx_id的申请次数，在InnoDB里，即使你只是执行一个普通的select语句，在执行过程中，也是要对应一个只读事务的。所以只读事务优化后，普通的查询语句不需要申请trx_id，就大大减小了并发事务申请trx_id的锁冲突

由于max_trx_id是持久化的，重启也不会清零，当这个值大达到2^24-1后，会出现脏读这个bug

[max_trx_id导致的脏读](../images/mysql实战45讲/max_trx_id导致的脏读.png)

先把系统的max_trx_id设置成2^48-1，所以在session A启动的事务TA的低水位就是2^48-1，第一条语句的事务id就是2^48-1，而第二条update语句的事务id就是0，因此T3时刻，session A执行select语句的时候，判断可见性发现，c=3这个数据的trx_id，小于事务TA的低水位，因此认为这个数据可见。由于低水位值会持续增加，导致这个系统在这个时刻之后的所有数据都会出现脏读


### thread_id
thread_id值：MySQL系统保存了一个全局变量thread_id_counter,每新建一个连接，就将thread_id_counter赋值给这个新连接的线程变量

thread_id_counter定义的大小是4个字节，因此达到2^32-1后，就会重置为0，然后继续增加，但不会看到两个相同的thread_id,因为MySQL设计了一个唯一数组的逻辑：

    do {
      new_id= thread_id_counter++;
    } while (!thread_ids.insert_unique(new_id).second);
