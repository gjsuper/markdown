# 39：高性能网络应用框架Netty

### 网络编程性能的瓶颈

之前32节里写的echo程序采用的是阻塞式I/O，所有read()和write()操作都会阻塞当前操作，所以使用BIO模型，一般会为每个socket分配一个独立的线程，这样就不会因为线程阻塞在一个socket上而影响对其他socket的读写。

但是实际场景是，互联网中往往需要服务器能够支撑上十万、百万的链接，这样的话用上述的BIO显然是不适合的，而且，在大多数的场景中，虽然连接虽多，但每个连接的请求并不频繁，所以线程大部分时间都在等待I/O就绪，这完全就是浪费，如果能解决这个问题，就不需要那么多线程了。

Java提供了NIO模式，利用非阻塞式API就能够实现一个线程处理多个连接了。现在普遍采用Reactor模式来实现。

### Reactor模式
[Reacter模型](../images/Java并发/Reacter模型.png)
这里，Handle指的是I/O句柄，他的本质是一个网络连接。Event handler就是事件处理器，其中handle_event()方法处理I/O事件，也就是每个Event Handle处理一个I/O Handle；get_handle()返回这个I/O的Handle。Synchronous Event Demutiplexer可以理解为操作系统提供的I/O多路复用API，利用POSIX标准里的select()以及linux里的epoll()。

Reacter模式的核心就是Reactor这个类，其中register_handle()和remove_handle()这两个方法可以注册和删除一个事件处理器；handle_events()方法是核心，他的逻辑是：首先通过同步事件多路选择器提供的select()方法监听网络事件，当有网络事件就绪后，就遍历处理器来处理该网络事件，由于网络事件是源源不断的，所以在主程序中启动Reacter模式，需要以while(true)的方式调用handle_events()方法：

```Java
void Reactor::handle_events(){
  // 通过同步事件多路选择器提供的
  //select() 方法监听网络事件
  select(handlers);
  // 处理网络事件
  for(h in handlers){
    h.handle_event();
  }
}
// 在主程序中启动事件循环
while (true) {
  handle_events();
```

### Netty中的线程模型
Netty中最核心的概念是事件循环（EventLoop），其实就是Reacter模式中的Reacter，负责监听网络事件并调用事件处理器进行处理，在4.x的版本中，网络连接和EventLoop是多对一的关系，而EventLoop和线程是1对1的关系，也就是说1个网络连接只会对应一个线程。

好处就是对于一个网络连接的事件处理时单线程的，这样就可以避免各种并发问题。
[Netty中的线程模型](../images/Java并发/Netty中的线程模型.png)

Netty的另一个核心概念就是EventLoopGroup，她就是由一组EventLoop组成的。实际使用时会创建两个EventLoopGroup，一个是称为bossGroup，用于处理连接请求，另一个是workerGroup，用于处理读写请求的。这是由socket的网络请求机制决定的，因为socket处理TCP连接请求和读写请求是通过两个不同的socket完成的（监听是一个socket，建立连接后会创建一个新的socket）

bossGroup处理完连接请求后，会把这个连接交给workerGroup来处理，workerGroup里有多个EventLoop，这个新的连接是通过轮询算法来决定由哪一个EventLoop来处理。

### 用Netty实现Echo程序服务端

1. 如果NettyGroup只监听一个端口，那bossGroup只需要一个EventLoop就可以了
2. 默认情况下，Netty会创建“2*CPU核数”个EventLoop，由于网络连接与EventLoop有稳定关系，所以事件处理器在处理网络事件的时候是不能有阻塞操作的，否则很容易导致请求大面积超时。如果无法避免使用阻塞操作，那可以通过线程池来异步操作：

```Java
// 事件处理器
final EchoServerHandler serverHandler
  = new EchoServerHandler();
//boss 线程组  
EventLoopGroup bossGroup
  = new NioEventLoopGroup(1);
//worker 线程组  
EventLoopGroup workerGroup
  = new NioEventLoopGroup();
try {
  ServerBootstrap b = new ServerBootstrap();
  b.group(bossGroup, workerGroup)
   .channel(NioServerSocketChannel.class)
   .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch){
       ch.pipeline().addLast(serverHandler);
     }
    });
  //bind 服务端端口  
  ChannelFuture f = b.bind(9090).sync();
  f.channel().closeFuture().sync();
} finally {
  // 终止工作线程组
  workerGroup.shutdownGracefully();
  // 终止 boss 线程组
  bossGroup.shutdownGracefully();
}

//socket 连接处理器
class EchoServerHandler extends
    ChannelInboundHandlerAdapter {
  // 处理读事件  
  @Override
  public void channelRead(
    ChannelHandlerContext ctx, Object msg){
      ctx.write(msg);
  }
  // 处理读完成事件
  @Override
  public void channelReadComplete(
    ChannelHandlerContext ctx){
      ctx.flush();
  }
  // 处理异常事件
  @Override
  public void exceptionCaught(
    ChannelHandlerContext ctx,  Throwable cause) {
      cause.printStackTrace();
      ctx.close();
  }
}
```
