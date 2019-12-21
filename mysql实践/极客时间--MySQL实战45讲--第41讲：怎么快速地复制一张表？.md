# 极客时间--MySQL实战45讲--第41讲：怎么快速地复制一张表？

### mysqldumo方法
使用mysqldump命令将数据导出成一组INSERT语句，可以使用下面的命令：

    mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql

### 导出CSV文件
另一种方法是直接将结果导出成.csv文件。MySQL提供了下面的语法，用来将查询结果导出到服务端本地目录：

    select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

### 物理拷贝方法
