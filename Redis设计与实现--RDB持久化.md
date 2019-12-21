# Redis设计与实现--RDB持久化（Redis设计与实现第十章）

* RDB文件的创建于载入
    - SAVE:创建RDB文件，阻塞进程
    - GBSAVE:创建RDB文件，派生出一个子进程来负责创建RDB文件
* BGSAVE:
    - BGSAVE执行期间，客户端发送的SAVE和BGSAVE命令会被拒绝，防止同时执行rdbSave，防止产生条件竞争
    - 如果BGSAVE正在执行，客户端发送的BGREWRITEAOF命令会被延迟到BGSAVE执行完以后
    - 如果BGREWRITEAOF正在执行，那么客户端发送的BGSAVE命令会被服务器拒绝，这俩命令都由子进程执行，并没有冲突，但从性能上考虑，两个子进程同时进行大量的磁盘写入操作，对性能有影响
- RDB文件载入期间，服务器会一直处于阻塞状态
- 自动间隔性保存
    - 由于BGSAVE不阻塞进程，redis可以通过配置save选项，让服务器每隔一段时间自动执行BGSAVE命令
    - 配置如下：save 900 1  : 在900s之内，对数据库进行了至少1次修改
      save 300 10
      save 60 10000
    - dirty计数器和lastsave属性：
        - dirty计数器：记录距离从上一次成功执行SAVE和BGSAVE命令之后，服务器对数据库状态进行了多少次修改
        - lastsave是一个时间戳，记录上一次服务器成功执行SAVE或BGSAVE命令的时间
    - 检查保存条件是否满足：redis服务器的serverCron每个100毫秒会执行一次，其中一项工作是检查save选项配置的保存条件是否已经满足，如果满足就执行BGSAVE命令
* RDB文件结构 [RDB文件结构](/images/redis/RDB文件结构.png)

举例：[database为空的RDB文件](/images/redis/database为空的RDB文件.png)
举例：[RDB文件中的数据库结构示例](/images/redis/RDB文件中的数据库结构示例.png)

* 键值对的数据结构
    - [不带过期时间的键值对](/images/redis/不带过期时间的键值对.png)
    - [带有过期时间的键值对](/images/redis/带有过期时间的键值对.png)
