#thrift源码解析--方法调用模型
&emsp;&emsp;thrift的方法调用不涉及反射机制，他是通过方法名称和实际方法实现类的注册完成的。

##1.1  相关组件
* 自动生成的iface接口：远程方法的顶层接口
* 自动生成的processor类相关父类：TProcessor接口，TBaseProcessor抽象类
* ProcessFunction抽象类：抽象了一个具体的方法调用，包含了方法信息，调用方法的抽象过程等。
* TNoblockingServer：NIO服务器的默认实现，通过Args参数类配置Processor等信息。
* FrameBuffer类：服务器NIO的缓冲对象，在服务端收到全包并解码后，会调用Processor去完成实际方法的调用
##1.2  TProcessor接口和相关类
1. TProcessor定义了一个顶层的调用方法，参数是输入流和输出流
```java
public interface TProcessor {
  public boolean process(TProtocol in, TProtocol out)
    throws TException;
}
```
2. 抽象类TBaseProcessor implements TProcessor,提供了TProcessor中process的默认实现，先读消息头，拿到要调用的方法名，然后从维护的Map中取出ProcessFunction对象。ProcessFunction对象是实际方法的抽象，调用，调用他的process方法，调用的是实际的方法。
```java
public abstract class TBaseProcessor<I> implements TProcessor {
  private final I iface;
  private final Map<String,ProcessFunction<I, ? extends TBase>> processMap;

  protected TBaseProcessor(I iface, Map<String, ProcessFunction<I, ? extends TBase>> processFunctionMap) {
    this.iface = iface;
    this.processMap = processFunctionMap;
  }

  @Override
  public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin();
    ProcessFunction fn = processMap.get(msg.name);
    if (fn == null) {
      TProtocolUtil.skip(in, TType.STRUCT);
      in.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
      x.write(out);
      out.writeMessageEnd();
      out.getTransport().flush();
      return true;
    }
    fn.process(msg.seqid, in, out, iface);
    return true;
  }
}
```
3. Processor类继承自TBaseProcessor,他依赖Iface接口，负责把实际的方法实现和犯法的key关联起来，放到Map中维护。
```java
public static class Processor<I extends Iface> extends org.apache.thrift.TBaseProcessor<I> implements org.apache.thrift.TProcessor {
    public Processor(I iface) {
      super(iface, getProcessMap(new HashMap<String, org.apache.thrift.ProcessFunction<I, ? extends org.apache.thrift.TBase>>()));
    }

    protected Processor(I iface, Map<String,  org.apache.thrift.ProcessFunction<I, ? extends  org.apache.thrift.TBase>> processMap) {
      super(iface, getProcessMap(processMap));
    }

    private static <I extends Iface> Map<String,  org.apache.thrift.ProcessFunction<I, ? extends  org.apache.thrift.TBase>> getProcessMap(Map<String,  org.apache.thrift.ProcessFunction<I, ? extends  org.apache.thrift.TBase>> processMap) {
      processMap.put("demoMethod", new demoMethod());
      return processMap;
    }

    private static class demoMethod<I extends Iface> extends org.apache.thrift.ProcessFunction<I, demoMethod_args> {
      public demoMethod() {
        super("demoMethod");
      }

      protected demoMethod_args getEmptyArgsInstance() {
        return new demoMethod_args();
      }

      protected demoMethod_result getResult(I iface, demoMethod_args args) throws org.apache.thrift.TException {
        demoMethod_result result = new demoMethod_result();
        result.success = iface.demoMethod(args.param1, args.param2, args.param3);
        result.setSuccessIsSet(true);
        return result;
      }
    }
}
```

&emsp;&emsp;在上面的代码中，demoMethod类继承自ProcessFunction类，他把方法参数，Iface，方法返回值这些抽象的概念组合在一起。

&esp;&emsp;TNonblockingServer是NIO服务器的实现，他处理的是读事件，用handlead
来进一步处理。TNonblockingServer通过Selector来检查IO就绪状态，进而调用相关的channel。
```java
private void select() {
      try {
        // wait for io events.
        selector.select();

        // process the io events we received
        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
          SelectionKey key = selectedKeys.next();
          selectedKeys.remove();

          // skip if not valid
          if (!key.isValid()) {
            cleanupSelectionKey(key);
            continue;
          }

          // if the key is marked Accept, then it has to be the server
          // transport.
          if (key.isAcceptable()) {
            handleAccept();
          } else if (key.isReadable()) {
            // deal with reads
            handleRead(key);
          } else if (key.isWritable()) {
            // deal with writes
            handleWrite(key);
          } else {
            LOGGER.warn("Unexpected state in select! " + key.interestOps());
          }
        }
      } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
      }
    }

   protected void handleRead(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.read()) {
        cleanupSelectionKey(key);
        return;
      }

      // if the buffer's frame read is complete, invoke the method.
      if(buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
          cleanupSelectionKey(key);
        }
      }
    }

   protected boolean requestInvoke(FrameBuffer frameBuffer) {
    frameBuffer.invoke();
    return true;
}
```
&emsp;&emsp;NIO会使用FrameBuffer来作为缓冲区类存放读写的中间状态，它提供了invoke()来实现对Processor的调用。
```java
public void invoke() {
      TTransport inTrans = getInputTransport();
      TProtocol inProt = inputProtocolFactory_.getProtocol(inTrans);
      TProtocol outProt = outputProtocolFactory_.getProtocol(getOutputTransport());

      try {
        processorFactory_.getProcessor(inTrans).process(inProt, outProt);
        responseReady();
        return;
      } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
      } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
      }
      // This will only be reached when there is a throwable.
      state_ = FrameBufferState.AWAITING_CLOSE;
      requestSelectInterestChange();
}
```

FrameBuffer使用了ProcessorFactory来获得Processor。ProcessorFactory实在创建服务器的时候传递过来的，只是对Processor的简单封装。
```java
protected TServer(AbstractServerArgs args) {
    processorFactory_ = args.processorFactory;
    serverTransport_ = args.serverTransport;
    inputTransportFactory_ = args.inputTransportFactory;
    outputTransportFactory_ = args.outputTransportFactory;
    inputProtocolFactory_ = args.inputProtocolFactory;
    outputProtocolFactory_ = args.outputProtocolFactory;
  }

public class TProcessorFactory {

  private final TProcessor processor_;

  public TProcessorFactory(TProcessor processor) {
    processor_ = processor;
  }

  public TProcessor getProcessor(TTransport trans) {
    return processor_;
  }
}

public T processor(TProcessor processor) {
      this.processorFactory = new TProcessorFactory(processor);
      return (T) this;
}

```

##1.3 实例
&emsp;&emsp;下面把实际的服务提供者通过服务器参数的方式Processor传递给TNoblockingServer,共FrameBuffer使用。
```java
public class DemoServiceImpl implements DemoService.Iface{

	@Override
	public int demoMethod(String param1, Parameter param2,
			Map<String, String> param3) throws TException {

		return 0;
	}

	public static void main(String[] args){
		TNonblockingServerSocket socket;
		try {
			socket = new TNonblockingServerSocket(9090);
			TNonblockingServer.Args options = new TNonblockingServer.Args(socket);
			TProcessor processor = new DemoService.Processor(new DemoServiceImpl());
			options.processor(processor);
			options.protocolFactory(new TCompactProtocol.Factory());
			TServer server = new TNonblockingServer(options);
			server.serve();			
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
}
```
