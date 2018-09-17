#Thrift源码分析--IDL和代码生成分析

##1.1 类的组成部分
* 接口类型：默认名称都是Iface，会被客户端和服务器同时使用。服务器端使用它来作为顶层接口，编写实现类。客户端使用它作为生成代理的服务接口。
自动生成的接口有两个，一个是同步调用的Iface，一个是一比吊用的AsynIface。
```java
public interface Iface {
    public int demoMethod(String param1, Parameter param2, Map<String,String> param3) throws org.apache.thrift.TException;
}

public interface AsyncIface {
    public void demoMethod(String param1, Parameter param2, Map<String,String> param3, org.apache.thrift.async.AsyncMethodCallback<AsyncClient.demoMethod_call> resultHandler) throws org.apache.thrift.TException;
}
```
* 客户端类型：同步调用的客户端Client，异步调用的客户端AsyncClient。
* Processor:支持方法调用，每个服务的实现类都要在Processor注册，这样最后服务器端调用接口的实现时能定位到具体的实现类。
* 方法参数的封装类：以“方法名_args”命名
* 方法返回值得封装类：以“方法名_result”命名

##1.2 同步调用客户端Client
* 提供工厂方法创建Client对象
* 接口方法的客户端的代理做了两件事：
    * 发送方法调用请求:
        * 创建方法参数对象，封装方法参数
        * 调用父类的sendBase方法发送消息。发送时先通过writeMessageBegin发送消息头，在参数对象的write（TPtotocol）方法发送消息体，最后结束发送。
    * 接受返回值
        * 创建方法返回值对象，封装方法返回值
        * 调用父类的receiveBase方法接收方法返回值。先通过receiveMessage接收消息体，处理异常，然后调用方法参数对象的read(TProtocol)方法来接受消息体，最后结束接收。
```java
public static class Client extends org.apache.thrift.TServiceClient implements Iface {
    public static class Factory implements org.apache.thrift.TServiceClientFactory<Client> {
      public Factory() {}
      public Client getClient(org.apache.thrift.protocol.TProtocol prot) {
        return new Client(prot);
      }
      public Client getClient(org.apache.thrift.protocol.TProtocol iprot, org.apache.thrift.protocol.TProtocol oprot) {
        return new Client(iprot, oprot);
      }
    }

　 public int demoMethod(String param1, Parameter param2, Map<String,String> param3) throws org.apache.thrift.TException
    {
      send_demoMethod(param1, param2, param3);
      return recv_demoMethod();
    }

    public void send_demoMethod(String param1, Parameter param2, Map<String,String> param3) throws org.apache.thrift.TException
    {
      demoMethod_args args = new demoMethod_args();
      args.setParam1(param1);
      args.setParam2(param2);
      args.setParam3(param3);
      sendBase("demoMethod", args);
    }

    public int recv_demoMethod() throws org.apache.thrift.TException
    {
      demoMethod_result result = new demoMethod_result();
      receiveBase(result, "demoMethod");
      if (result.isSetSuccess()) {
        return result.success;
      }
      throw new org.apache.thrift.TApplicationException(org.apache.thrift.TApplicationException.MISSING_RESULT, "demoMethod failed: unknown result");
    }

  }


//org.apache.thrift.TServiceClient.sendBase，客户端的父类方法
 protected void sendBase(String methodName, TBase args) throws TException {
　　// 发送消息头
    oprot_.writeMessageBegin(new TMessage(methodName, TMessageType.CALL, ++seqid_));
   //　发送消息体，由方法参数对象自己处理编解码
    args.write(oprot_);
    oprot_.writeMessageEnd();
    oprot_.getTransport().flush();
  }

  protected void receiveBase(TBase result, String methodName) throws TException {
    // 接收消息头
    TMessage msg = iprot_.readMessageBegin();
    if (msg.type == TMessageType.EXCEPTION) {
      TApplicationException x = TApplicationException.read(iprot_);
      iprot_.readMessageEnd();
      throw x;
    }
    if (msg.seqid != seqid_) {
      throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID, methodName + " failed: out of sequence response");
    }
    //由返回值对象自己处理编解码
    result.read(iprot_);
    iprot_.readMessageEnd();
}
```
## 1.3 方法参数对象
&emsp;&emsp;他实现了TBase接口，TBase接口定义了对象在某种协议下的编解码接口。
```java
public interface TBase<T extends TBase<?,?>, F extends TFieldIdEnum> extends Comparable<T>,  Serializable {
  public void read(TProtocol iprot) throws TException;

  public void write(TProtocol oprot) throws TException;
｝
```
&emsp;&emsp;方法参数对象做了两件事：
* 创建每个参数的元数据（参数类型，序列号）。参数序号用来识别参数的位置，在编解码的时候用。
* 实现自己的编解码方法，read(TProtocol)，write(TProtocol)。具体的编解码功能在XXXScheme（StandardScheme和TU盘了Scheme）类里实现。
```java
public static class demoMethod_args implements org.apache.thrift.TBase<demoMethod_args, demoMethod_args._Fields>, java.io.Serializable, Cloneable   {
    private static final org.apache.thrift.protocol.TStruct STRUCT_DESC = new org.apache.thrift.protocol.TStruct("demoMethod_args");

    private static final org.apache.thrift.protocol.TField PARAM1_FIELD_DESC = new org.apache.thrift.protocol.TField("param1", org.apache.thrift.protocol.TType.STRING, (short)1);
    private static final org.apache.thrift.protocol.TField PARAM2_FIELD_DESC = new org.apache.thrift.protocol.TField("param2", org.apache.thrift.protocol.TType.STRUCT, (short)2);
    private static final org.apache.thrift.protocol.TField PARAM3_FIELD_DESC = new org.apache.thrift.protocol.TField("param3", org.apache.thrift.protocol.TType.MAP, (short)3);

    private static final Map<Class<? extends IScheme>, SchemeFactory> schemes = new HashMap<Class<? extends IScheme>, SchemeFactory>();
    static {
        schemes.put(StandardScheme.class, new demoMethod_argsStandardSchemeFactory());
        schemes.put(TupleScheme.class, new demoMethod_argsTupleSchemeFactory());
    }

    public String param1; // required
    public Parameter param2; // required
    public Map<String,String> param3; // required

    /** The set of fields this struct contains, along with convenience methods for finding and manipulating them. */
    public enum _Fields implements org.apache.thrift.TFieldIdEnum {
        PARAM1((short)1, "param1"),
        PARAM2((short)2, "param2"),
        PARAM3((short)3, "param3");

    private static final Map<String, _Fields> byName = new HashMap<String, _Fields>();

    static {
        for (_Fields field : EnumSet.allOf(_Fields.class)) {
            byName.put(field.getFieldName(), field);
        }
    }

　　 // 对象自己负责解码
    public void read(org.apache.thrift.protocol.TProtocol iprot) throws org.apache.thrift.TException {
        schemes.get(iprot.getScheme()).getScheme().read(iprot, this);
    }

    public void write(org.apache.thrift.protocol.TProtocol oprot) throws org.apache.thrift.TException {
        schemes.get(oprot.getScheme()).getScheme().write(oprot, this);
    }
}
```

##1.4 编解码Schema
&emsp;&emsp;XXXScheme类是XXX_args参数的内部类。
&emsp;&emsp;以StandardScheme为例：
* 它的编码方式是从writeStructBegin开始逐个写字段，每个字段写之前会writeFieldBegin，写字段类型和字段的顺序号。如果字段是一个Struct，就调用自己的write（TProtocol）。写完后以writeStructEnd结束。
* 解码方式从readStructBegin开始，然后读字段元数据readFieldBegin，读1个字节的字段类型，2字节的字段顺序，然后根据字段类型来读相应类型长度的数据。读完后用readStructEnd结束。
```java
private static class demoMethod_argsStandardScheme extends StandardScheme<demoMethod_args> {

      public void read(org.apache.thrift.protocol.TProtocol iprot, demoMethod_args struct) throws org.apache.thrift.TException {
        org.apache.thrift.protocol.TField schemeField;
        iprot.readStructBegin();
        while (true)
        {
          schemeField = iprot.readFieldBegin();
          if (schemeField.type == org.apache.thrift.protocol.TType.STOP) {
            break;
          }
          switch (schemeField.id) {
            case 1: // PARAM1
              if (schemeField.type == org.apache.thrift.protocol.TType.STRING) {
                struct.param1 = iprot.readString();
                struct.setParam1IsSet(true);
              } else {
                org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
              }
              break;
            case 2: // PARAM2
              if (schemeField.type == org.apache.thrift.protocol.TType.STRUCT) {
                struct.param2 = new Parameter();
                struct.param2.read(iprot);
                struct.setParam2IsSet(true);
              } else {
                org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
              }
              break;
            case 3: // PARAM3
              if (schemeField.type == org.apache.thrift.protocol.TType.MAP) {
                {
                  org.apache.thrift.protocol.TMap _map0 = iprot.readMapBegin();
                  struct.param3 = new HashMap<String,String>(2*_map0.size);
                  for (int _i1 = 0; _i1 < _map0.size; ++_i1)
                  {
                    String _key2; // required
                    String _val3; // required
                    _key2 = iprot.readString();
                    _val3 = iprot.readString();
                    struct.param3.put(_key2, _val3);
                  }
                  iprot.readMapEnd();
                }
                struct.setParam3IsSet(true);
              } else {
                org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
              }
              break;
            default:
              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
          }
          iprot.readFieldEnd();
        }
        iprot.readStructEnd();

        // check for required fields of primitive type, which can't be checked in the validate method
        struct.validate();
      }

      public void write(org.apache.thrift.protocol.TProtocol oprot, demoMethod_args struct) throws org.apache.thrift.TException {
        struct.validate();

        oprot.writeStructBegin(STRUCT_DESC);
        if (struct.param1 != null) {
          oprot.writeFieldBegin(PARAM1_FIELD_DESC);
          oprot.writeString(struct.param1);
          oprot.writeFieldEnd();
        }
        if (struct.param2 != null) {
          oprot.writeFieldBegin(PARAM2_FIELD_DESC);
          struct.param2.write(oprot);
          oprot.writeFieldEnd();
        }
        if (struct.param3 != null) {
          oprot.writeFieldBegin(PARAM3_FIELD_DESC);
          {
            oprot.writeMapBegin(new org.apache.thrift.protocol.TMap(org.apache.thrift.protocol.TType.STRING, org.apache.thrift.protocol.TType.STRING, struct.param3.size()));
            for (Map.Entry<String, String> _iter4 : struct.param3.entrySet())
            {
              oprot.writeString(_iter4.getKey());
              oprot.writeString(_iter4.getValue());
            }
            oprot.writeMapEnd();
          }
          oprot.writeFieldEnd();
        }
        oprot.writeFieldStop();
        oprot.writeStructEnd();
      }
}
```

TupleScheme的代码逻辑：通过BitSet来区分参数，只写消息体，没有消息头，数据量更小。
```java
private static class demoMethod_argsTupleScheme extends TupleScheme<demoMethod_args> {

      @Override
      public void write(org.apache.thrift.protocol.TProtocol prot, demoMethod_args struct) throws org.apache.thrift.TException {
        TTupleProtocol oprot = (TTupleProtocol) prot;
        BitSet optionals = new BitSet();
        if (struct.isSetParam1()) {
          optionals.set(0);
        }
        if (struct.isSetParam2()) {
          optionals.set(1);
        }
        if (struct.isSetParam3()) {
          optionals.set(2);
        }
        oprot.writeBitSet(optionals, 3);
        if (struct.isSetParam1()) {
          oprot.writeString(struct.param1);
        }
        if (struct.isSetParam2()) {
          struct.param2.write(oprot);
        }
        if (struct.isSetParam3()) {
          {
            oprot.writeI32(struct.param3.size());
            for (Map.Entry<String, String> _iter5 : struct.param3.entrySet())
            {
              oprot.writeString(_iter5.getKey());
              oprot.writeString(_iter5.getValue());
            }
          }
        }
      }

      @Override
      public void read(org.apache.thrift.protocol.TProtocol prot, demoMethod_args struct) throws org.apache.thrift.TException {
        TTupleProtocol iprot = (TTupleProtocol) prot;
        BitSet incoming = iprot.readBitSet(3);
        if (incoming.get(0)) {
          struct.param1 = iprot.readString();
          struct.setParam1IsSet(true);
        }
        if (incoming.get(1)) {
          struct.param2 = new Parameter();
          struct.param2.read(iprot);
          struct.setParam2IsSet(true);
        }
        if (incoming.get(2)) {
          {
            org.apache.thrift.protocol.TMap _map6 = new org.apache.thrift.protocol.TMap(org.apache.thrift.protocol.TType.STRING, org.apache.thrift.protocol.TType.STRING, iprot.readI32());
            struct.param3 = new HashMap<String,String>(2*_map6.size);
            for (int _i7 = 0; _i7 < _map6.size; ++_i7)
            {
              String _key8; // required
              String _val9; // required
              _key8 = iprot.readString();
              _val9 = iprot.readString();
              struct.param3.put(_key8, _val9);
            }
          }
          struct.setParam3IsSet(true);
        }
      }
    }
}
```
