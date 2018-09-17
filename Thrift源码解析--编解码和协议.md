#一.Thrift源码解析--协议和编解码
##1.1  编解码接口
&emsp;&emsp;thrift把协议和编码整合在了一起。
抽象类TProtocal定义了协议和编解码的顶级接口，TProtocal关联了一个TTransport传输对象。

&emsp;&emsp;TProtocol的工作：
1. 定义了一系列读写消息的编解码接口，包括两类：
   * 针对复杂的数据结构比如readMessageBegin，readMessageEnd,writeMessageBegin,writeMessageEnd.
   * 针对简单的数据结构，比如：read32,write32,wrireString
```java
public abstract class TProtocol {

  /**
   * Transport
   */
  protected TTransport trans_;

　public abstract void writeMessageBegin(TMessage message) throws TException;

  public abstract void writeMessageEnd() throws TException;

  public abstract void writeStructBegin(TStruct struct) throws TException;

  public abstract void writeStructEnd() throws TException;

  public abstract void writeFieldBegin(TField field) throws TException;

  public abstract void writeFieldEnd() throws TException;

  public abstract void writeFieldStop() throws TException;

  public abstract void writeMapBegin(TMap map) throws TException;

  public abstract void writeMapEnd() throws TException;

  public abstract void writeListBegin(TList list) throws TException;

  public abstract void writeListEnd() throws TException;

  public abstract void writeSetBegin(TSet set) throws TException;

  public abstract void writeSetEnd() throws TException;

  public abstract void writeBool(boolean b) throws TException;

  public abstract void writeByte(byte b) throws TException;

  public abstract void writeI16(short i16) throws TException;

  public abstract void writeI32(int i32) throws TException;

  public abstract void writeI64(long i64) throws TException;

  public abstract void writeDouble(double dub) throws TException;

  public abstract void writeString(String str) throws TException;

  public abstract void writeBinary(ByteBuffer buf) throws TException;

  /**
   * Reading methods.
   */

  public abstract TMessage readMessageBegin() throws TException;

  public abstract void readMessageEnd() throws TException;

  public abstract TStruct readStructBegin() throws TException;

  public abstract void readStructEnd() throws TException;

  public abstract TField readFieldBegin() throws TException;

  public abstract void readFieldEnd() throws TException;

  public abstract TMap readMapBegin() throws TException;

  public abstract void readMapEnd() throws TException;

  public abstract TList readListBegin() throws TException;

  public abstract void readListEnd() throws TException;

  public abstract TSet readSetBegin() throws TException;

  public abstract void readSetEnd() throws TException;

  public abstract boolean readBool() throws TException;

  public abstract byte readByte() throws TException;

  public abstract short readI16() throws TException;

  public abstract int readI32() throws TException;

  public abstract long readI64() throws TException;

  public abstract double readDouble() throws TException;

  public abstract String readString() throws TException;

  public abstract ByteBuffer readBinary() throws TException;

  /**
   * Reset any internal state back to a blank slate. This method only needs to
   * be implemented for stateful protocols.
   */
  public void reset() {}

  /**
   * Scheme accessor
   */
  public Class<? extends IScheme> getScheme() {
    return StandardScheme.class;
  }
```
##1.2  传输数据
对于传输的数据内容

调用方：
* 方法名称，包括类名和方法名
* 方法的参数，包括参数类型和参数值
* 附加数据，比如附件、超时事件、自定义的控制信息等等。

服务方：
* 调用的返回码
* 返回值
* 异常信息


##1.3  thrift的约定

###1.3.1 写数据
1. 先writeMessageBegin表示开始传输消息了，先写消息头。Message里面定义了方法名，调用的类版本号，消息seqid。
2. 接下来写方法的参数，实际就是写消息体。如果参数是一个类，就writeStructBegin
3. 接下来写字段,writeFieldBegin,这个方法会写接下来的字段的数据类型和顺序号。这个顺序号是thrift对要传的字段的一个编码，从1开始
4. 如果是一个集合就writeListBegin/writeMapBegin,如果是基本数据类型，拨入int，就writeI32
5. 每个复杂的数据类型写完，都调用writeXXXEnd,之道writeMessageEnd结束
6. 读消息时根据数据类型读取相应的长度
---
1. writeMessgeBegin方法写了消息头，包括4字节的版本号和类型信息，字符串类型的方法名，４字节的序列号seqId
2. writeFieldBegin，写了１个字节的字段数据类型，和2个字节字段的顺序号
3. writeI32，写了４个字节的字节数组
4. writeString,先写４字节消息头表示字符串长度，再写字符串字节
5. writeBinary,先写４字节消息头表示字节数组长度，再写字节数组内容
6.readMessageBegin时，先读４字节版本和类型信息，再读字符串，再读４字节序列号
7.readFieldBegin，先读1个字节的字段数据类型，再读2个字节的字段顺序号
8. readString时，先读４字节字符串长度，再读字符串内容。字符串统一采用UTF-8编码
```java
public void writeMessageBegin(TMessage message) throws TException {
    if (strictWrite_) {
      int version = VERSION_1 | message.type;
      writeI32(version);
      writeString(message.name);
      writeI32(message.seqid);
    } else {
      writeString(message.name);
      writeByte(message.type);
      writeI32(message.seqid);
    }
  }

public void writeFieldBegin(TField field) throws TException {
    writeByte(field.type);
    writeI16(field.id);
  }

private byte[] i32out = new byte[4];
  public void writeI32(int i32) throws TException {
    i32out[0] = (byte)(0xff & (i32 >> 24));
    i32out[1] = (byte)(0xff & (i32 >> 16));
    i32out[2] = (byte)(0xff & (i32 >> 8));
    i32out[3] = (byte)(0xff & (i32));
    trans_.write(i32out, 0, 4);
  }

public void writeString(String str) throws TException {
    try {
      byte[] dat = str.getBytes("UTF-8");
      writeI32(dat.length);
      trans_.write(dat, 0, dat.length);
    } catch (UnsupportedEncodingException uex) {
      throw new TException("JVM DOES NOT SUPPORT UTF-8");
    }
  }

public void writeBinary(ByteBuffer bin) throws TException {
    int length = bin.limit() - bin.position();
    writeI32(length);
    trans_.write(bin.array(), bin.position() + bin.arrayOffset(), length);
  }

public TMessage readMessageBegin() throws TException {
    int size = readI32();
    if (size < 0) {
      int version = size & VERSION_MASK;
      if (version != VERSION_1) {
        throw new TProtocolException(TProtocolException.BAD_VERSION, "Bad version in readMessageBegin");
      }
      return new TMessage(readString(), (byte)(size & 0x000000ff), readI32());
    } else {
      if (strictRead_) {
        throw new TProtocolException(TProtocolException.BAD_VERSION, "Missing version in readMessageBegin, old client?");
      }
      return new TMessage(readStringBody(size), readByte(), readI32());
    }
  }

public TField readFieldBegin() throws TException {
    byte type = readByte();
    short id = type == TType.STOP ? 0 : readI16();
    return new TField("", type, id);
  }

public String readString() throws TException {
    int size = readI32();

    if (trans_.getBytesRemainingInBuffer() >= size) {
      try {
        String s = new String(trans_.getBuffer(), trans_.getBufferPosition(), size, "UTF-8");
        trans_.consumeBuffer(size);
        return s;
      } catch (UnsupportedEncodingException e) {
        throw new TException("JVM DOES NOT SUPPORT UTF-8");
      }
    }

    return readStringBody(size);
}
```
##1.3生成的代码简介
见thrift自动生成的代码
1. 方法的调用从writeMessageBegin开始，发送了消息头
2. 写方法的参数。方法的参数有统一的接口TBase来描述，提供了read和write的统一接口
3. 写完收工
