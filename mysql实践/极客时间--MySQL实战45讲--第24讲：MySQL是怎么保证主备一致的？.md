# 极客时间--MySQL实战45讲--第24讲：MySQL是怎么报纸准备一致的？

* 主从同步过程：
    1. 在备库上通过change master命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量
    2. 在备库上执行start slave命令，这时备库回启动两个线程，就是图中的io_thread和sql_thread（后来演化成了多线程）。io_thread负责与主库建立连接。
    3. 主库检验完用户名、密码后，开始按照备库传过来的位置，从本地读取binlog，发给备库
    4. 备库拿到binlog后，写到本地文件，称为中转日志
    5. sql_thread读取中转日志，解析出日志里的命令，并执行。
* mixed格式的binlog
    - statement格式的binlog可能会导致主备不一致
    - row格式的binlog很占空间
    - 因此MySQL产生了一个折中的方案：mixed格式的binlog。意思就是，MySQL会自己判断这条SQL语句是否可能会引起主备不一致，如果有可能，就用row格式，否则使用statement格式。也就是说，mixed格式既具备占用空间小的特点，也有避免数据不一致的能力

* 但现在越来越多的场景要求把MySQL的binlog格式设置为row。这么做的其中一个理由是：恢复数据
    - delete语句，row格式的binlog也会把被删掉的行的整行信息保存下来，可以直接把binlog中记录的delete语句转成insert，把错删除的数据恢复出来
    - insert语句，row格式的binlog也会记录所有的数据信息，这些信息可以用来精确定位刚刚被插入的那一行，所以可以直接把insert语句转成delete语句
    - update语句，binlog里面会记录修改前整行的数据和修改后的整行数据。所以只需要把这个event千户的两行信息对调一下，再去数据库里执行，就能恢复这个更新操作了。

    - 对于下面的语句：insert into t values(10, 10, now());采用statement格式的话，记录的是sql语句本身，binlog传给备库的时候，时间延迟会导致主从库的now()值不一样，所以在binlog中记录event的视乎，多记录一条命令：set TIMESTAMP=1546103491。他用这个命令忽而定了接下来的now()函数的返回时间。
* 通过server id
    1. 规定两个库的server id必须不同，如果相同，则他们不能设定为主备关系
    2. 一个备库接到binlog并在重放的过程中，生成与原来binlog的server id相同的新的binlog
    3. 每个库在收到从自己的主库发过来的日之后，先判断server id，如果跟自己的相同，表示这个日志时自己生成的，就直接丢掉这个日志。
