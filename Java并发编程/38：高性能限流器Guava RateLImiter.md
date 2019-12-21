# 38：高性能限流器Guava RateLImiter

我们利用Guava RateLImiter创建了一个流速为2个请求/秒的限流器，实例如下：
```Java
// 限流器流速：2 个请求 / 秒
RateLimiter limiter =
  RateLimiter.create(2.0);
// 执行任务的线程池
ExecutorService es = Executors
  .newFixedThreadPool(1);
// 记录上一次执行时间
prev = System.nanoTime();
// 测试执行 20 次
for (int i=0; i<20; i++){
  // 限流器限流
  limiter.acquire();
  // 提交任务异步执行
  es.execute(()->{
    long cur=System.nanoTime();
    // 打印时间间隔：毫秒
    System.out.println(
      (cur-prev)/1000_000);
    prev = cur;
  });
}

输出结果：
...
500
499
499
500
499
```

### 经典限流算法：令牌桶算法

详细算法如下：
1. 令牌以固定塑料谈驾到令牌桶当中，假设限流的速率是r/秒，则令牌每1/r秒回添加一个
2. 假设令牌桶的容量是b，如果令牌桶已满，则新的令牌会被丢弃
3. 请求能够通过限流器的前提是令牌桶中有令牌

令牌桶的容量b：brust，意义是限流器允许的最大容量，例如b=10，表示限流器允许10个请求同时通过限流器

如何实现：直观的想法是利用生产者-消费者模式，一个生产者已固定速率向阻塞队列添加令牌，而试图通过限流器的线程则作为消费者，只有从阻塞队列获取到令牌，才允许通过限流器。

这里有一个问题，在高并发场景，系统的压力已临近极限，此时定时器的精度无误差会非常大，同时定时器本身会创建调度线程，也会对系统的性能有影响。

### Guava 如何实现令牌桶算法
Guava采用了一个很简单的办法，其关键就是**动态计算下一令牌发放的时间。我们只需要记录下一个令牌产生的时间，并动态更新他，就能轻松完成限流功能**

下面的实现关键是reverse()方法，其逻辑是如果线程请求令牌的时间在下一个令牌产生的时间之后，那么线程就可以立即获得令牌；反之，如果请求时间在下一令牌的产生时间之前，那么就需要计算下一个令牌产生的时间，该线程获取令牌的时间就是下个令牌产生的时间

```Java
class SimpleLimiter {
  // 下一令牌产生时间
  long next = System.nanoTime();
  // 发放令牌间隔：纳秒
  long interval = 1000_000_000;
  // 预占令牌，返回能够获取令牌的时间
  synchronized long reserve(long now){
    // 请求时间在下一令牌产生时间之后
    // 重新计算下一令牌产生时间
    if (now > next){
      // 将下一令牌产生时间重置为当前时间
      next = now;
    }
    // 能够获取令牌的时间
    long at=next;
    // 设置下一令牌产生时间
    next += interval;
    // 返回线程需要等待的时间
    return Math.max(at, 0L);
  }
  // 申请令牌
  void acquire() {
    // 申请令牌时的时间
    long now = System.nanoTime();
    // 预占令牌
    long at=reserve(now);
    long waitTime=max(at-now, 0);
    // 按照条件等待
    if(waitTime > 0) {
      try {
        TimeUnit.NANOSECONDS
          .sleep(waitTime);
      }catch(InterruptedException e){
        e.printStackTrace();
      }
    }
  }
}
```

如果令牌桶的容量大于一，那又如何处理呢，令牌首先才能改令牌桶中出，所以我们需要按需计算令牌桶中的数量，当有线程请求令牌时，先从桶中出。这里增加了resync()方法，这个方法里，如果线程请求的时间在下一个令牌产出时间之后，会重新计算令牌桶中的令牌数，如果令牌是从令牌桶中出，那么next就无需增加interval了。

```Java
class SimpleLimiter {
  // 当前令牌桶中的令牌数量
  long storedPermits = 0;
  // 令牌桶的容量
  long maxPermits = 3;
  // 下一令牌产生时间
  long next = System.nanoTime();
  // 发放令牌间隔：纳秒
  long interval = 1000_000_000;

  // 请求时间在下一令牌产生时间之后, 则
  // 1. 重新计算令牌桶中的令牌数
  // 2. 将下一个令牌发放时间重置为当前时间
  void resync(long now) {
    if (now > next) {
      // 新产生的令牌数
      long newPermits=(now-next)/interval;
      // 新令牌增加到令牌桶
      storedPermits=min(maxPermits,
        storedPermits + newPermits);
      // 将下一个令牌发放时间重置为当前时间
      next = now;
    }
  }
  // 预占令牌，返回能够获取令牌的时间
  synchronized long reserve(long now){
    resync(now);
    // 能够获取令牌的时间
    long at = next;
    // 令牌桶中能提供的令牌
    long fb=min(1, storedPermits);
    // 令牌净需求：首先减掉令牌桶中的令牌
    long nr = 1 - fb;
    // 重新计算下一令牌产生时间
    next = next + nr*interval;
    // 重新计算令牌桶中的令牌
    this.storedPermits -= fb;
    return at;
  }
  // 申请令牌
  void acquire() {
    // 申请令牌时的时间
    long now = System.nanoTime();
    // 预占令牌
    long at=reserve(now);
    long waitTime=max(at-now, 0);
    // 按照条件等待
    if(waitTime > 0) {
      try {
        TimeUnit.NANOSECONDS
          .sleep(waitTime);
      }catch(InterruptedException e){
        e.printStackTrace();
      }
    }
  }
}
```

### 总结
经典的限流算法有两种，一种是令牌桶算法，另一中是漏桶算法。令牌桶是定时向令牌桶发送令牌，请求需要从桶中获取到令牌才能通过限流器，而漏桶算法里，请求就像水一样注入漏桶，漏桶会按照一定的速率自动将水漏掉，只有漏桶里还能注入水的时候，请求才能通过限流器。这俩算法就像是一个硬币的正反面。
