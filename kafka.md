#kafka
##一.基本概念
1. kafka集群存储的消息是一topic为类别记录的
2. 每个消息有一个key，valuee和时间戳组成的

##二.四个核心API
1. Producer API发布消息到1个或多个topic
2. COnsumer API订阅一个或多个topic，并处理产生的消息
3. Streams API充当流处理器，从1个或多个topic消费输入流，并产生一个输出流到1个或多个topic，有效的将输入流转换为输出流
4. Connector API允许构建或运行可重复使用的生产者或消费者，将topic连接到现有的应用程序或数据系统。例如一个关系数据库的连接器科捕获每一个变化
见图所示:[api图](https://github.com/gjsuper/markdown/blob/master/images/kafka/coreApi.png)

##三.kafka读写高效的原因
1. 一般将数据从文件传到套接字的路径：
 * 操作系统将数据从磁盘读到内核空间的页缓存
 * 应用将数据从内核空间读到用户空间的缓存中
 * 应用将数据从从用户空间写到内存空间的套接字缓存中
 * 操作系统将数据从套接字缓存写到网卡缓存中，以便将数据经网络发出

 总共涉及到四次拷贝，两次系统调用

 kafka采用sendfile（java为FileChannel.transferTo api）两次拷贝可避免：允许操作系统将数据直 接从页缓存发送到网络。优化后，只有最后一步将数据拷贝到网卡缓存是需要的，不用拷贝到用户空间。只涉及到 两次拷贝。

2. kafka基于磁盘的顺序写比随机读写快

生产： 写入 -> pagecached -> 磁盘(一次生产，多次消费，放到pagecached有助于提高读写速度，等满了  或者过期了在写入磁盘)
消费： 磁盘 -> 网络

##四.kafka消息检索原理
1. kafka利用index文件存储消息是第几条，以及该消息的偏移量
[kafka消息检索原理](https://github.com/gjsuper/markdown/blob/master/images/kafka/%E6%B6%88%E6%81%AF%E6%A3%80%E7%B4%A2.PNG)
2. kafka定位某条消息的过程
 - 利用消息的offset定位消息所在的segment file文件（二分查找）
 - 再次利用消息的offset和index文件找到该的消息（二分查找）

##四.参考文献
1. [kafka简介](https://www.cnblogs.com/likehua/p/3999538.html)