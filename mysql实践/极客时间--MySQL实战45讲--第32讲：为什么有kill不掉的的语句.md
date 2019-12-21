# 极客时间--MySQL实战45讲--第32讲：为什么有kill不掉的的语句

MySQL有两种kill命令：

     kill query + 线程id：表示终止这个线程正在执行的语句
     kill conection + 线程id：connection是可缺省的，表示断开这个线程的连接，如果线程由于局正在执行，也要先停止正在执行的语句

* 收到kill后，线程做了什么

[kill语句的例子](../images/mysql实战45讲/kill语句的例子.png)

这里的session B不是直接退出的，session B虽然处于blocked状态，但还是持有一个MDL读锁。如果线程被kill的时候直接终止，那这个锁就没机会释放了

**当用户执行 kill query thread_id_B时，mysql里处理kill命令做了两件事**

1. 把session B的运行状态改为THD:KILL_QUERY(将变量killed赋值为THD:KILL_QUERY)
2. 给session B的执行线程发送一个信号，让session B推出等待，来处理这个THD:KILL_QUERY状态，如果不发信号，线程B还会继续等待

[killquery无效的例子](../images/mysql实战45讲/killquery无效的例子.png)

可以看到：
1. session C执行被堵住了
2. 但是session D执行的kill query C命令无效
3. 直到session E执行kill conection命令，才断开了session C的连接
4. 但这时候，如果在session E中执行show processlist，能看到：

[killconnection的结果](../images/mysql实战45讲/killconnection的结果.png)

这时id=12这个线程的command列显示的是killed，也就是说，客户端虽然断开了连接，但实际上服务端上这条语句还在执行过程中。这是因为：
在锁等待时，使用的是pthread_cond_timedwait函数，这个等待状态可以被唤醒。但在这个例子里，12号线程每10秒判断一下是否可以进入InnoDB执行，如果不行，就调用nanosleep函数进入sleep状态，虽然12号线程已经被设置成了KILL_QUERY状态，但是这个等待进入InnoDB的循环过程中，并没有去爬那段线程的状态，因此不会进入终止逻辑。

kill connection逻辑：
1. 把12号线程状态设置为KILL_CONNECTION
2. 关掉12号线程的网络连接。所以会看到session C收到了断开连接的提示

        如果一个线程的状态是 KILL_CONNECTION，就把 Command 列显示成 Killed。所以command列显示的就是killed
