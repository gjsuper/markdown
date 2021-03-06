# 极客时间--MySQL实战45讲--第8讲：事务到底是隔离还是不隔离

* begin/start transaction 命令并不是一个事务的起点，在他之后的第一个操作Innodb表的语句，事务才真正启动，创建视图也是在第一次个语句执行的时候。如果想马上启动一个事务或者立即产生视图，可以使用start transaction with  consistent snapshot命令。

例子：

    mysql> CREATE TABLE `t` (
    `id` int(11) NOT NULL,
    `k` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
    ) ENGINE=InnoDB;
    insert into t(id, k) values(1,1),(2,2);

[事务A、B、C的执行流程](../images/mysql实战45讲/事务A、B、C的执行流程.png)

这里事务A查到的数据时是1，事务B查到的事务是3。事务B的update，如果按照一致性读，结果貌似不对？图中事务B的视图是先生成的，之后事务C才提交，不是应该看不见（1，2）吗，怎么会算出（1，3）来？

如果事务B在更新前先去查一次数据，查出的k值确实是1.

但当他去更新数据的时候，就不能再在历史版本上更新了，否则事务C的更新就丢失了，因此事务B此时的set k=K+1是在（1，2）的基础上进行的。

所以这里就用到了一条规则：**更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”**

因此，在事务B在（1，2）的基础上更新完数据后，变成了（1，3），这个数据版本的row trx_id是事务B的版本，当事务B查询的时候，一看这个数据的版本是自己的版本，就会直接可以用，所以查到的数据k=3。

如果select语句加锁也就当前读，例如 select ... lock in share mode，select .. for update

而这些历史版本的数据什么时候删除呢？当没有事务会再使用它们的时候，就会被删除


* 所有的更新数据都是先读后写的，而这个读是当前读，只能读当前的值（也就是最新的值），称为当前读，当前读必须要加锁。除了update语句外，select语句如果加锁，也是当前读。

* 读提交和可重复读的逻辑类似，为什么读提交不能实现重复读，他们的主要区别是：
    - 在可重复读隔离的级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图
    - 在读提交的隔离级别下，每一个语句执行前都会生成一个新的视图
    - 当前读的情况下,rr是利用record lock+gap lock来实现的,而rc没有gap,所以rc不能可重复读

* 为什么表结构不支持“可重复读”，这是因为表结构没有对应的行数据，也没有row trx_id，因此只能遵循当前读
