##Thrift源码解析--FrameBuffer
&emsp;&emsp;FrameBuffer是Thrift NIO服务器的核心组件，他一方面承担了NIO缓冲区的功能，一方面负责调用RPC方法。

###1.1 相关组件
[FrameBuffer相关类](/Users/jingjie/Documents/markdown/images/thrift/FrameBuffer类.png)

###1.2 FrameBuffer状态
&emsp;&emsp;FrameBufferState定义了FrameBuffer的七种状态。
```java

private enum FrameBufferState {
    // in the midst of reading the frame size off the wire
    // 读Frame消息头，实际是4字节表示Frame长度
    READING_FRAME_SIZE,
    // reading the actual frame data now, but not all the way done yet
    // 读Frame消息体
    READING_FRAME,
    // completely read the frame, so an invocation can now happen
    // 读满包
    READ_FRAME_COMPLETE,
    // waiting to get switched to listening for write events
    // 等待注册写
    AWAITING_REGISTER_WRITE,
    // started writing response data, not fully complete yet
    // 写半包
    WRITING,
    // another thread wants this framebuffer to go back to reading
    // 等待注册读
    AWAITING_REGISTER_READ,
    // we want our transport and selection key invalidated in the selector
    // thread
    // 等待关闭
    AWAITING_CLOSE
  }
```
##1.3 FrameBuffer读顺序
1. 先读4字节的Frame消息头
2. 将FrameBufferState状态从READING_FRAME_SIZE更改为READING_FRAME，并根据读到的Frame长度修改
3. 读Frame消息体，如果读完就修改状态到READ_FRAME_COMPLETE,否则还是把FrameBuffer绑定到SelectionKey，下次继续读。
```java
public boolean read() {
      if (state_ == FrameBufferState.READING_FRAME_SIZE) {
        // try to read the frame size completely
        if (!internalRead()) {
          return false;
        }

        // if the frame size has been read completely, then prepare to read the
        // actual frame.
        if (buffer_.remaining() == 0) {
          // pull out the frame size as an integer.
          int frameSize = buffer_.getInt(0);
          if (frameSize <= 0) {
            LOGGER.error("Read an invalid frame size of " + frameSize
                + ". Are you using TFramedTransport on the client side?");
            return false;
          }

          // if this frame will always be too large for this server, log the
          // error and close the connection.
          if (frameSize > MAX_READ_BUFFER_BYTES) {
            LOGGER.error("Read a frame size of " + frameSize
                + ", which is bigger than the maximum allowable buffer size for ALL connections.");
            return false;
          }

          // if this frame will push us over the memory limit, then return.
          // with luck, more memory will free up the next time around.
          if (readBufferBytesAllocated.get() + frameSize > MAX_READ_BUFFER_BYTES) {
            return true;
          }

          // increment the amount of memory allocated to read buffers
          readBufferBytesAllocated.addAndGet(frameSize);

          // reallocate the readbuffer as a frame-sized buffer
          buffer_ = ByteBuffer.allocate(frameSize);

          state_ = FrameBufferState.READING_FRAME;
        } else {
          // this skips the check of READING_FRAME state below, since we can't
          // possibly go on to that state if there's data left to be read at
          // this one.
          return true;
        }
      }

      // it is possible to fall through from the READING_FRAME_SIZE section
      // to READING_FRAME if there's already some frame data available once
      // READING_FRAME_SIZE is complete.

      if (state_ == FrameBufferState.READING_FRAME) {
        if (!internalRead()) {
          return false;
        }

        // since we're already in the select loop here for sure, we can just
        // modify our selection key directly.
        if (buffer_.remaining() == 0) {
          // get rid of the read select interests
          selectionKey_.interestOps(0);
          state_ = FrameBufferState.READ_FRAME_COMPLETE;
        }

        return true;
      }

      // if we fall through to this point, then the state must be invalid.
      LOGGER.error("Read was called but state is invalid (" + state_ + ")");
      return false;
}
```
&emsp;&emsp;internalRead方法调用SocketChannel来读数据。
```java
private boolean internalRead() {
     try {
        if (trans_.read(buffer_) < 0) {
          return false;
        }
        return true;
      } catch (IOException e) {
        LOGGER.warn("Got an IOException in internalRead!", e);
        return false;
      }
}
```
##1.4  写缓冲
1. 写之前把FrameBuffer的状态改成WRITING
1. 如果没写任何数据就返回false
2. 写完了后，thrift把Select提欧尼Key注册时间改成读，而常用的做法是把写事件取消

```java
public boolean write() {
      if (state_ == FrameBufferState.WRITING) {
        try {
          if (trans_.write(buffer_) < 0) {
            return false;
          }
        } catch (IOException e) {
          LOGGER.warn("Got an IOException during write!", e);
          return false;
        }

        // we're done writing. now we need to switch back to reading.
        if (buffer_.remaining() == 0) {
          prepareRead();
        }
        return true;
      }

      LOGGER.error("Write was called, but state is invalid (" + state_ + ")");
      return false;
}
```
&emsp;&emsp;FrameBuffer可以根据SelectKey的状态来切换自身状态，也可以根据子自身状态来选择注册的Channel事件。
```java
public void changeSelectInterests() {
      if (state_ ==                 FrameBufferState.AWAITING_REGISTER_WRITE) {
        // set the OP_WRITE interest
        selectionKey_.interestOps(SelectionKey.OP_WRITE);
        state_ = FrameBufferState.WRITING;
      } else if (state_ == FrameBufferState.AWAITING_REGISTER_READ) {
        prepareRead();
      } else if (state_ == FrameBufferState.AWAITING_CLOSE) {
        close();
        selectionKey_.cancel();
      } else {
        LOGGER.error("changeSelectInterest was called, but state is invalid (" + state_ + ")");
      }
}
```
##1.4 方法调用
&emsp;&emsp;FrameBuffer提供了invoker方法，从消息头拿到要调用的方法，然后通过它管理的Processor来完成实际方法的调用。然后切换到写模式来写消息体。
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

 public void responseReady() {
      // the read buffer is definitely no longer in use, so we will decrement
      // our read buffer count. we do this here as well as in close because
      // we'd like to free this read memory up as quickly as possible for other
      // clients.
      readBufferBytesAllocated.addAndGet(-buffer_.array().length);

      if (response_.len() == 0) {
        // go straight to reading again. this was probably an oneway method
        state_ = FrameBufferState.AWAITING_REGISTER_READ;
        buffer_ = null;
      } else {
        buffer_ = ByteBuffer.wrap(response_.get(), 0, response_.len());

        // set state that we're waiting to be switched to write. we do this
        // asynchronously through requestSelectInterestChange() because there is
        // a possibility that we're not in the main thread, and thus currently
        // blocked in select(). (this functionality is in place for the sake of
        // the HsHa server.)
        state_ = FrameBufferState.AWAITING_REGISTER_WRITE;
      }
      requestSelectInterestChange();
}
```

&emsp;&emsp;写消息体responseReady（）：
1. 创建ByteBuffer
2. 修改状态到AWAITING_REGISTER_WRITE
3. 调用requestSelectInteresetChange（）方法来注册Channel的写事件。
4. 当Selector根据isWriteable状态来调用要写的Channel时，会调用FrameBuffer的write方法。
