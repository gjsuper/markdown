# 40：高性能队列Disruptor

Disruptor是一个高性能的有界内存队列，性能高是因为：
1. 内存分配更合理，使用RingBUffer数据结构，数组元素在初始化时一次性全部创建，提升缓存命中率；对象循环利用，避免频繁gc
2. 能够避免伪共享，提升缓存利用率
3. 采用无锁算法，避免频繁加锁、解锁的性能消耗
4. 支持批量消费，消费者可用无锁方式消费多个消息

### 基本用法

* Disruptor的消息对象称为event，使用Disruptor必须自定义Event，入下面的LongEvent
* 构建Disruptor对象除了要指定队列大小外，还需传入一个EventFactory，代码里是LongEvent::new;
* 消费Disruptor中的Event需要通过handleEventsWith()注册一个事件处理器，发布Event则需要通过publishEvent()方法

```Java
// 自定义 Event
class LongEvent {
  private long value;
  public void set(long value) {
    this.value = value;
  }
}
// 指定 RingBuffer 大小,
// 必须是 2 的 N 次方
int bufferSize = 1024;

// 构建 Disruptor
Disruptor<LongEvent> disruptor
  = new Disruptor<>(
    LongEvent::new,
    bufferSize,
    DaemonThreadFactory.INSTANCE);

// 注册事件处理器
disruptor.handleEventsWith(
  (event, sequence, endOfBatch) ->
    System.out.println("E: "+event));

// 启动 Disruptor
disruptor.start();

// 获取 RingBuffer
RingBuffer<LongEvent> ringBuffer
  = disruptor.getRingBuffer();
// 生产 Event
ByteBuffer bb = ByteBuffer.allocate(8);
for (long l = 0; true; l++){
  bb.putLong(0, l);
  // 生产者生产消息
  ringBuffer.publishEvent(
    (event, sequence, buffer) ->
      event.set(buffer.getLong(0)), bb);
  Thread.sleep(1000);
}
```

### Disruptor如何提升性能

程序的局部性原理：指的是一段时间内程序的执行会限定在一个局部范围内。包括时间局限性和空间局限性，时间局限性：指的是程序中的某条指令一旦被执行，不久之后这条指令很可能再次被执行；如果某条数据被放完，不久之后这条数据很可能再次被访问。空间局限性指的是某块内存一旦被访问，不久之后这块内存附近的内存也可能被访问。

Dsiruptor被补的RingBuffer使用数据实现的，数组里的元素在初始化时是一次性全部创建的，所以这些元素的地址大概率是连在一起的，这样遵循空间局部性原理，内存预加载的时候，会把相邻的数据也加载进缓存，访问数据的时候就可以直接从缓存里取，可以大大提高效率。

同时一次性创建了所有的元素，新来的数据是通过set操作更新旧的元素（已访问过的）即可，因此避免了频繁地创建和删除对象，减少了gc的负担。

为了避免“伪共享”，Disruptor使用了缓存行填充的方式：

```Java
// 前：填充 56 字节
class LhsPadding{
    long p1, p2, p3, p4, p5, p6, p7;
}
class Value extends LhsPadding{
    volatile long value;
}
// 后：填充 56 字节
class RhsPadding extends Value{
    long p9, p10, p11, p12, p13, p14, p15;
}
class Sequence extends RhsPadding{
  // 省略实现
}
```
### disruptor中的无锁算法
对于入队操作，最关键的是不能覆盖没有消费的元素；对于出队操作，最关键的是不能读取没有写入的元素，所以Disruptor中也一定维护了类似出队和入队的索引变量，其实Disruptor的RingBuffer只维护了入队索引，但没有维护出队索引，因为在Disruptor中多个消费者可以同时消费，每个消费者都会有一个出队索引，所以RingBuffer的出队索引时所有消费者里面最小的那个。

Disruptor的入队核心代码如下：如果没有足够的空余位置，就让出CPU的使用权，然后重新计算；繁殖使用CAS设置入队索引：

```Java
// 生产者获取 n 个写入位置
do {
  //cursor 类似于入队索引，指的是上次生产到这里
  current = cursor.get();
  // 目标是在生产 n 个
  next = current + n;
  // 减掉一个循环
  long wrapPoint = next - bufferSize;
  // 获取上一次的最小消费位置
  long cachedGatingSequence = gatingSequenceCache.get();
  // 没有足够的空余位置
  if (wrapPoint>cachedGatingSequence || cachedGatingSequence>current){
    // 重新计算所有消费者里面的最小值位置
    long gatingSequence = Util.getMinimumSequence(
        gatingSequences, current);
    // 仍然没有足够的空余位置，出让 CPU 使用权，重新执行下一循环
    if (wrapPoint > gatingSequence){
      LockSupport.parkNanos(1);
      continue;
    }
    // 从新设置上一次的最小消费位置
    gatingSequenceCache.set(gatingSequence);
  } else if (cursor.compareAndSet(current, next)){
    // 获取写入位置成功，跳出循环
    break;
  }
} while (true);
```
