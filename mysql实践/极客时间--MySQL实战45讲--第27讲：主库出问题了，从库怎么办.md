# 极客时间--MySQL实战45讲--第27讲：主库出问题了，从库怎么办

* 一主多从结构
[一主多从结构](../images/mysql实战45讲/一主多从结构.png)
其中A和A'互为主备，从库B、C、D指向的都是主库

* 主库发生故障，主备切换
[主备切换](../images/mysql实战45讲/主备切换.png)

* 基于位点的主备切换
&emsp;&emsp;当把节点B设置为A’的从库的时候，需要执行一条change master命令：

        CHANGE MASTER TO
        MASTER_HOST=$host_name
        MASTER_PORT=$port
        MASTER_USER=$user_name
        MASTER_PASSWORD=$password
        MASTER_LOG_FILE=$master_log_name
        MASTER_LOG_POS=$master_log_pos  
而MASTER_LOG_FILE和MASTER_LOG_POS这个位点一般很难精确的获取。由于切换过程中不能丢数据，所以我们找位点的时候，总是找一个“稍微往前”的，然后通过判断跳过那些已经在从库B上执行过的事务。

一种取同步位点的方法：
1. 等待新库A'把中转日志（relay log）全部同步完成
2. 在A'上执行show master status命令，得到当前A'最新的FILE 和position
3. 取原主库A故障的时刻T
4. 用mysqlbinlog工具解析A'的file，得到T时刻的位点

&emsp;&emsp;然而这个值并不精确，因为存在一种情况是，主库A已经执行完一个insert语句插入了数据行R，并且已经将binlog传给A’和B，然后在传完的瞬间主库Ad主机就掉电了。这时在从库B上，由于同步了binlog，R这一行数据已经存在了，在新主库A'上，这一行数据也是存在的，日志卸载123这个位置之后。我们在从库B上执行change master 命令，指向的file文件的123位置，就会把插入R这一行的binlog又同步到从库B上，这时从库B上会报出Duplicated entry ‘id_of_R’ for key ‘PRIMARY’错误，提示出现主键冲突，然后停止同步。
**所以，通常情况下，我们在切换任务的时候，要先跳过这些错误，一种做法是主动跳过一个事务（set global sql_slave_skip_counter = 1），另一种做法是设置slave_skip_errors参数，直接设置跳过指定的错误，因此可以把slave_skip_errors设置为1032(插入数据唯一键冲突)，1062(删除数据时找不到行),这样碰到这两个问题时就直接跳过**

* GTID：global transaction identifier，格式：GTID=server_uuid:gno
&emsp;&emsp;使用前面提到的方式操作复杂且容易出错，所以mysql5.6引入了GTID，彻底解决了这个问题。GTID模式下，GTID有两种生成方式，这取决于session变量gtid_next的值
    - 如果gtid_next = automatic，代表使用默认值。这时MySQL会把server_uuid:gno分配给这个事务
        - 记录binlog的时候，先记录一行SET@@SESSION.GTID_NEXT='server_uuid:gnos'
        - 把这个GTID加入本实例的GTID集合
    - 如果gtid_next拾亿贰指定的GTID值，比如通过set gtid_next='current_gtid'指定为current_gtid，那么就有两种可能：
        - 如果current_gtid已经存在于GTID集合中，接下来执行的这个事务会直接被系统忽略；
        - 如果current_gtid没有存在于实例的GTID的集合中，就将这个current_gtid分配给接下来要执行的事务，也就是说系统不需要给这个事务生成新的GTID，因此gno也不用加1
这样每个mysql都维护有一个GTID集合，用来对于这个实例执行过的所有事务。

* 基于GTID的主备切换
&emsp;&emsp;在GTID模式下，备库B设置为辛苦A'的从库的语法：

        CHANGE MASTER TO
        MASTER_HOST=$host_name
        MASTER_PORT=$port
        MASTER_USER=$user_name
        MASTER_PASSWORD=$password
        master_auto_position=1
其中，master_auto_position=1表示这个主备关系使用GTID协议，而file和file_position参数没有了

在B上执行start slave，取binlog逻辑：
1. 实例B指定主库A，基于主备协议建立连接
2. 实例B把set_b发给主库A'
3. 实例A'算出set_a与set_b的差集（存在于a，不存在于b）的GTID的集合，判断A’的本地是否包含了这个差集需要的所有binlog
    - 如果不包含，表示A'已经把实例B需要的binlog删除了，直接发返错误
    - 如果确认全部包含，A'从自己的binlog文件里，找出第一个不存在与set_b的事务，发给B；
4. 之后就从这个事务开始，往后读文件，按顺序取binlog发给B取去执行

严谨的说，主备切换不是不需要找位点了，而是找位点这个工作由新的主库来完成，是自动的，开发系统的人员很友好。
