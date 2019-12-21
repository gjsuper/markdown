# 极客时间--MySQL实战45讲--第2讲：日志系统

* redo log：重做日志，保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力被称为crash-safe(用于发生异常时，恢复之前的数据，位于引擎层，是InnoDB引擎特有的)
    - 文件大小固定，write pos记录当前位置，checkpoint是当前要擦除的位置，两者都是往后推进并且循环的，擦除记录前把记录更新到数据文件，write pos和checkpoint之间可以用来记录新的操作
    [relog的两个指针](../images/mysql实战45讲/relog的两个指针.png)
* binlog：归档日志，位于server层，可用于"误删除"恢复
* redo log和binlog区别：
    - redo log是InnoDB特有的，binlog是mysql的server层实现的，所有引擎都可以使用
    - redo log是物理日志，记录的是“在某个数据也上做了什么修改”；binlog时逻辑日志，记录的是这个语句的原始逻辑，比如“给id=2
    这一行的c字段加1"
    - redolog是循环写的，空间固定，binlog是可以追加的，文件写到一定大小会切换到下一个，不会覆盖以前的日志
- 一个update语句的执行过程:[update语句执行过程](../images/mysql实战45讲/update语句执行过程.png)
    - redolog的写入分为了两个步骤：prepare和commit，这既是两阶段提交，这是为了保证redo log和binlog的一致性
