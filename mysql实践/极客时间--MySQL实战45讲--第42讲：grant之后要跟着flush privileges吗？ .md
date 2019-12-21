# 极客时间--MySQL实战45讲--第42讲：grant之后要跟着flush privileges吗？

先说结论：
[权限范围和修改策略](../images/mysql实战45讲/权限范围和修改策略.jpg)

grant语句用来给用户赋权，但grant之后真的要执行flush privileges吗？如果不执行这个flush的话，赋权语句真的不能生效吗？

创建一个用户：

    create user 'ua'@'%' identified by 'pa';
    这里创建一个用户'ua'@'%'，密码是pa，mysql里用户名(user) + dizhi (host)才表示一个用户
这条命令做了两个动作：
1. 磁盘上，往mysql.user表里插入一行，由于没有指定权限，所以这行数据上表示权限的字段值都是N；
2. 内存里，网数组acl_users里插入一个acl_user
对象，这个对象的access字段值是0.

[mysql.user数据行](../images/mysql实战45讲/mysql.user数据行.png)

### 全局权限
全局权限作用于整个MySQL实例，这些信息保存在mysql库的user表里，表示最高权限：

    grant all privileges on *.* to 'ua'@'%' with grant option;
他做了两个动作：
1. 磁盘上，将mysql.user表里，用户'ua'@'%'这一行的所有表示权限的字段都修改为'Y'
2. 内存里，从数组acl_user中找到这个用户的对象，将access值（权限位）修改为二进制的'全1'

作用范围：
1. 创建的新连接会使用新的权限
2. 已存在的连接，他的全局权限不受grant的影响

回收权限：

    revoke all privileges on *.* from 'ua'@'%';

1. 磁盘上，将mysql.user表里，用户'ua'@'%'这一行的所有表示权限的字段都修改为'N'
2. 内存里，从数组acl_user中找到这个用户的对象，讲将access值（权限位）修改为二进制的'全0'

### db权限
让用户拥有db1的所有权限：

    grant all privileges on db1.* to 'ua'@'%' with grant option;
他做了两个动作：
1. 磁盘上，在mysql.db表里插入一行记录，权限的字段都设置为'Y'
2. 内存里，增加一个对象到数组acl_dbs中，这个对象的权限位为'全1'

[mysql.db的数据行](../images/mysql实战45讲/mysql.db的数据行.png)

### 表权限和列权限
表权限放在mysql.tables_priv中，列权限定义放在表mysql.columns_priv中，这两类权限，组合起来放在内存的hash结构column_priv_hash中。

这两类权限的赋权命令如下：

    create table db1.t1(id int, a int);

    grant all privileges on db1.t1 to 'ua'@'%' with grant option;
    GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;
与db权限类似，这两个权限每次grant的时候都会修改数据表，也会同步修改内存中的hash结构。因此对这两类权限的操作，也会马上影响到已经存在的连接

flush privileges命令会清空acl_users数组，然后从mysql.user表中读取数据重新加载，重新构造一个一个acl_users数组。也就是说，以数据表中的数据为准，会将全局权限内存数组重新加载一边。

**因此，正常情况下，grant命令之后，没有必要跟着执行flush privileges命令**

### flush privileges使用场景
显然，当数据表中的权限数据跟内存中的权限数据不一致时，flush privileges语句可以用来重建内存数据，达到一致状态。数据不一致主要是由操作不规范引起的。例如：
[delete用户数据不一致](../images/mysql实战45讲/delete用户数据不一致.png)

[不规范权限操作导致的异常](../images/mysql实战45讲/不规范权限操作导致的异常.png)

可以看到，由于T3的时刻直接删除了数据表的记录，而内存的数据还存在。这就导致了：
1. T4时刻给用户ua赋权失败，因为mysql.user表中已经好不哒这行记录
2. 而T5时刻要重新创建这个用户也不行，因为在做内存判断的时候，会认为这个用户还存在
