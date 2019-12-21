# Redis设计与实现--事务（Redis设计与实现第十九章）

* Redis通过MULTI、EXEC、WATCH等命令来实现事务功能

* 事务的实现
    - 事务开始：MULTI命令标志着事务的开始
    - 命令入队：[服务器判断命令是否是入队还是立即执行](/images/redis/判断命令是入队还是立即执行.png)
    - 事务队列：

```C
 typedef struct redisClient {
     //...

     //事务状态：MULTI/EXEC
     multiState mState;

     //...
 } redisClient;

 typedef struct multiState {
     //事务队列，FIFO顺序
     multiCmd *commands;

     //已入队命令计数
     int count;
 } multiState;

 typedef struct multiCmd {
     //参数
     robj **argv;

     //参数数量
     int argc;

     //命令指针
     struct redisCommond *cmd;
 }multiCmd;
```
    - 当处于事务状态的客户端向服务器发送EXEC命令时，这个EXEC命令会被立即执行
- WATCH命令的实现：WATCH命令是一个乐观锁，他可以在EXEC命令执行之前，监视任意数量得数据库键，并在EXEC命令执行时，检查被监视的键是否被修改过了，如果是，服务器将拒绝执行事务，并向客户端返回事务执行失败的空回复。
    - 使用WATCH命令监视数据库，每个redis数据库都保存着一个watched_keys字典，这个字典的键是某个被WATCH命令监视的数据库键，值是一个链表，记录着监视相应数据库键的客户端
```C
typedef redisDb {
    //...

    //正在被WATCH命令监视的键
    dict *watched_keys;

    //...
} redisDb;
```
    - 监视机制的触发：所有对数据库修改的命令（set、sadd、del...）在执行后会查看是否有客户端在监视这些键，如果有，会将客户端的REDIS_DIRTY_CAS标志打开，表示客户端的事务安全性已经被破坏了
    - 判断事务是否安全：当服务器收到一个客户端发来的EXEC命令时，服务器会根据REDIS_DIRTY_CAS标志来决定是否只能执行事务，如果打开了，标志至少有一个键被修改了，那么提交的事务会被拒绝执行
- 事务的ACID性质
    - 原子性：与关系型数据库的区别是，Redis不支持事务回滚，即使事务队列中某个命令出错，整个事务也会继续执行下去，直到整个事务执行完
    - 一致性：数据符合数据库本身的定义和要求，没有包含非法或者无效的错误数据
    - 隔离性：多个事务兵法执行，各个事务之间不会相互影响，Redis是单线程的，因此事务总是隔离的
    - 耐久性：事务执行完时，所得的结果会被保存到永久性存储介质，即使服务器宕机，结果也不会丢失
