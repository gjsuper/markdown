#Thrift源码解析-Transport
&emsp;&emsp;Thrift使用Transport（抽象类）来封装底层的网络通信。

##1.1 Transport类结构
[Transport类结构](/Users/jingjie/Documents/markdown/images/thrift/Transport类结构.png)
1. TIOStreamTransport和TSocket这两个类对应阻塞同步IO，TSocket封装了Socket接口。
2. TNonblockingTransport和TNonblockingSocket对应非阻塞IO
3. TMemoryInputTransport封装了一个字节数组byte[]来做输入流的封装。
4. TMemoryBuffer使用字节数组输出流ByteArrayOutputStream做输出流的封装。
5. TFramedTransport则封装了TMemoryInputTransport做输入流，封装了TByteArrayOutputStream做输出流，作为内存读写缓冲区的一个封装。TFramedTransport调用flush方法时，会先写4个字节的输出流的长度作为消息头，然后写消息体。和FrameBuffer的读消息对应（FrameBuffer读消息时，先读4字节的消息头，再读消息体）。
6. TFastFramedTransport时是内存利用率最高的一个内存读写缓冲区，它使用自动增长的byte[]（不够长度时才new，而不是每次都new），提高了内存利用率。其他和FramedTransport一样，flush时先写4字节消息头，再写消息体。

**实现方式：Thrift的Transport采用装饰器模式实现包装流。**

Transport类型：
* 节点流：自身采用byte[]来提供IO学习的类
    * TMemoryInputTransport
    * TMemoryBuffer
    - TByteArrayOutputStream
    - 还有俩直接操作网络读写的对象，也认为是节点流：TNonblockingSocket和TSocket
* 包装流：
    - TFastFramedTransport
    - TFramedTransport
##1.2 实例
&emsp;&emsp;Thrift的NIO服务器读消息时使用了FrameBuffer，解码时会先读4字节长度的消息头，因此可知，客户端在发消息时，使用了包装流TFramedXXXTransport来传输数据。
**举例：一个客户端换的对象的构造情况**
```java
TSocket socket = new TSocket(host, port);
socket.setTimeout(timeout);
TTransport transport = new TFramedTransport(socket);
TProtocol protocol = new TCompactProtocol(transport);
transport.open();
```
&emsp;&emsp;Thrift的协议和具体的传输对象绑定的，协议使用具体的Transport传输对象。
