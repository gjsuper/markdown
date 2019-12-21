# 极客时间--MySQL实战45讲--第28讲：读写分离有那些坑?

* 由于存在延迟时间，从从库上独处的数据不一定是最新的
* 解决方式
    1. 强制走主方案：可以将查询请求分为两类
        1. 对于必须要拿到最新结果的请求，强制将其发到主库上
        2. 对于可以读出旧数据的请求，才将其发到从库上
    3. sleep方案：主库更新后，读之前sleep一下，看起来不太友好
        * 变更一下思路：商品发布后，用Ajax直接把客户顿的输入显示在页面上，而不是真正的去查数据库。等买家下一次去查询的时候，实际已经过了一段时间，也起到了sleep的效果
    3. 判断主备无延迟方案
         * 第一种确保主备无延迟的方法：showslave status中的 seconds_behind_master参数的值，但精确度较低，只能精确到秒。
         ![show slave status 结果](../images/mysql实战45讲/show-slave-status结果.png)

        - 第二中：对比位点确保主备无延迟，根据show slave status中的信息：
            - （Master_Log_File和Read_Master_Log_Pos：主库的文件和读到的位置信息），
            - Relay_master_Log_File和Exec_Master_Log_Pos，表示备库的最新位点
            - 如果两种相同，就表示日志已经同步
        - 对比GTID集合确保主备关系：
            - Auto_Position：表示主被使用了GTID协议
            - Retrived_Gtid_Set,备库收到的所有GTID集合
            - Executed_Gtid_Set,备库执行玩的GTID协议
            - 如果这两种集合相同，就表示备库收到的素有集合已经同步完成
&emsp;&emsp;但是这种方式精度也不高，因为我们判断的逻辑是，“备库收到的日志都执行完了”，但还有一部分日志，处于客户端已经收到确认提交，而备库还没有收到日志的状态

        4. 配合semi-syns
            - 要解决这个问题，就要引入半同步复制，也就是semi-sync replication，他是这么设计的
                1. 事务提交的时候，主库先把binlog发给从库
                2. 从库收到binlog以后，发挥给主库一个ack，表示收到了
                3. 主库收到了，才能给客户端返回“事务完成”的确认
            - 这样，semi-sync配合前面关于位点的判断，就可以确定在从库上执行的请求，可以避免过期读，但这只能用于一主一备的场景，在多备的情况下，人可能读出过期值，因为主库只要等到一个从库的ACK，就开始给客户端返回确认
        5. 等主库位点方案
            - 一条命令：select master_pos_wait(file, pos[, timeout]);他的逻辑是：
                1. 他是在从库执行的
                2. 参数file和pos指的是主库上的文件名和位置
                3. timeout是可选的，设置为正整数N，表示这个函数最多等待N秒
                4. 这个命令返回的结果是，聪明了执行开始，到应用完file和pos表示的binlog位置，执行了多少个事务。
            - 因此就可以查到正确的结果，逻辑为：
                1. 事务更新完后，马上show master status得到当前主库执行的到的File和Position
                2. 选定一个从库执行查询语句
                3. 在从库上执行，select master_pos_wait(file, pos[, timeout]);
                4. 如果返回值是 >=0 的正整数，则在这个从库执行查询语句
                5. 否则，到主库执行查询语句
            - 风险：如果从库的延时都超过1s，则所有的查询都会跑到主库上

        1. GTID方案

            * 如果数据库开启了GTID模式，对于的也有GTID的方案：select wait_for_executed_gtid_set(gtid_set, 1);他的逻辑是：
                1. 等待，直到这个库执行的事务中包含传入的gtid_set，返回0
                1. 超时返回1
            * 前面的等位点方案中，执行完事务后，还要主动去主库执行show master status。而MySQL5.7.6版本后，允许执行完更新类十五后，直接返回合格事务的GTID，这样等GTID的方案就少了一次查询，流程就编程了：
                1. 事务更新完后，从犯汇报中回去这个事务的GTID，jiuweigtid1；
                1. 选定一个从库执行查询语句
                1. 如果数据库开启了GTID模式，对于的也有GTID的方案：select wait_for_executed_gtid_set(gtid_set, 1);
                1. 如果返回值是0，则在这个从库执行查询语句
                1. 否则，到主库执行查询语句
            * 跟等主库位点的方案一样，等待超时后是否到主库查询，需要rd来做限流考虑
