#kafka
##一.基本概念
1. kafka集群存储的消息是一topic为类别记录的
2. 每个消息有一个key，valuee和时间戳组成的
##二.四个核心API
1. Producer API发布消息到1个或多个topic
2. COnsumer API订阅一个或多个topic，并处理产生的消息
3. Streams API充当流处理器，从1个或多个topic消费输入流，并产生一个输出流到1个或多个topic，有效的将输入流转换为输出流
4. Connector API允许构建或运行可重复使用的生产者或消费者，将topic连接到现有的应用程序或数据系统。例如一个关系数据库的连接器科捕获每一个变化
见图所示:[api图](images\kafka\coreApi.png)