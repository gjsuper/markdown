#Thrift源码解析-TServer
&emsp;&emsp;TServer是Thrift服务器的抽象,提供多种类型服务器的实现。</br>
&emsp;&emsp;TServerTransport是服务器的Acceptor抽象，用来监听端口和创建客户端Socket连接。

##1.1 TserverTransport
* TNonblockingServerReansport和TNonblockingServerSocket作为非阻塞IO的Accept偶然，封装了ServerSocketChannel。
* TServerSocket作为阻塞同步IO的Acceptor，封装了ServerSocket</br>
[TServer服务器](/Users/jingjie/Documents/markdown/images/thrift/TServer.png)

##1.2 TNonblockingServerSocket
```java
public class TNonblockingServerSocket extends TNonblockingServerTransport {
  private ServerSocketChannel serverSocketChannel = null;
}
 protected TNonblockingSocket acceptImpl() throws TTransportException {
    if (serverSocket_ == null) {
      throw new TTransportException(TTransportException.NOT_OPEN, "No underlying server socket.");
    }
    try {
      SocketChannel socketChannel = serverSocketChannel.accept();
      if (socketChannel == null) {
        return null;
      }

      TNonblockingSocket tsocket = new TNonblockingSocket(socketChannel);
      tsocket.setTimeout(clientTimeout_);
      return tsocket;
    } catch (IOException iox) {
      throw new TTransportException(iox);
    }
  }

  public void registerSelector(Selector selector) {
    try {
      // Register the server socket channel, indicating an interest in
      // accepting new connections
      serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    } catch (ClosedChannelException e) {
      // this shouldn't happen, ideally...
      // TODO: decide what to do with this.
    }
  }
public class TServerSocket extends TServerTransport {
  private ServerSocket serverSocket_ = null;
}
 protected TSocket acceptImpl() throws TTransportException {
    if (serverSocket_ == null) {
      throw new TTransportException(TTransportException.NOT_OPEN, "No underlying server socket.");
    }
    try {
      Socket result = serverSocket_.accept();
      TSocket result2 = new TSocket(result);
      result2.setTimeout(clientTimeout_);
      return result2;
    } catch (IOException iox) {
      throw new TTransportException(iox);
    }
}
```

TServer主要有两类：非阻塞IO和同步IO

非阻塞IO：
1. TNoblockingServer是单线程的，只有一个SelectAcceptThread线程来轮询IO就绪事件，调用就绪的Accept、read、write事件，并且还是使用这个线程来同步调用时方法的实现。
2. THsHaServer：半同步半异步的服务器。
    * 半同步：使用一个SelectAcceptThread线程来轮询IO事件，调用就绪的Accept、read、write事件。
    * 半异步：方法的调用封装成一个Runnable交给线程池来执行，交给线程池就立即返回，不等方法执行结束。
1. TThreadSelectorServer，他是多线程Reactor的一种实现。
    * 它采用AcceptorThread来专门监听端口，处理Accept事件，然后创建SocketChannel。创建完后交给一个线程来处理后续动作，将SocketChannel放到SelectorThread的阻塞队列acceptedQueue中。
    * 多个SelectorThread来处理创建好的SocketChannel。每个SelectorThread绑定一个Selector，这样将SocketChannel分给多个Selector。同时SelectorThread由卫华了一个阻塞队列acceptedQueue，从acceptedQueue中拿新创建好的SocketChannel。来注册读事件。

同步的Tserver有TThreadPoolServer，关联一个TServerSocket，采用同步IO来Accept，然后交给一个线程池来处理后续动作。

[Tserver类结构](/Users/jingjie/Documents/markdown/images/thrift/TServer.png)
