# 极客时间--MySQL实战45讲--第43讲：要不要使用分区表？

### 分区表的组织形式

    CREATE TABLE `t` (
      `ftime` datetime NOT NULL,
      `c` int(11) DEFAULT NULL,
      KEY (`ftime`)
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1
    PARTITION BY RANGE (YEAR(ftime))
    (PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
     PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
     PARTITION p_2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
    PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
    insert into t values('2017-4-1',1),('2018-4-1',1);
这个表包含了一个.frm文件和4个.ibd文件，每个分区对应一个.ibd文件。
1. 对引擎层来说，这是4个表
2. 对Server层来说，这是1个表

### 分区表的引擎层行为

[分区表间隙锁实例](../images/mysql实战45讲/分区表间隙锁实例.png)
如果是一个普通表的话。在T1时刻，在表t的ftime索引上，间隙和加锁状态应该是：

[普通表的加锁行为](../images/mysql实战45讲/普通表的加锁行为.png)

但可以看到，session B的第一个语句是可以执行成功的。因为对引擎层来说，p_2018和p_2019是两个不同的表，因此加锁状态应该是：
[分区表t的加锁范围](../images/分区表t的加锁范围.png)


MyISAM的分区表行为：
[MyISAM表锁行为](../immages/mysql实战45讲/MyISAM表锁行为.png)

这时因为MyISAM的表锁是在引擎层实现的，session A加的表锁，其实是在分区p_2018上。因此只会堵住这个分区的查询，落在其他分区的查询不受影响

### 分区表的问题--分区策略
每当第一次访问一个分区表的时候，MySQL需要把所有的分区访问一遍。一个典型的错误情况：如果一个分区表的分区很多，比如超过了1000个而MySQL的open_files_limit参数默认值是1024，那么MySQL启动的时候，由于需要打开所有的文件，导致打开表文件的个数超过了上限而报错。MyISAM引擎会有这个问题，InnoDB不会

### 分区表的server行为
1. MySQL在第一次打开分区表的时候，需要访问所有的分区
2. 从server层看，认为这是同一个表，因此所有分区共用一个MDL锁
3. 在引擎层，认为这是不同的表，因此MDL锁之后的执行过程，会根据分区表规则，只访问必要的分区
