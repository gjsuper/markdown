# 41：高性能数据库连接池HiKariCP

### HiKariCP的用法
下面的代码中，ds.getConnection()获取数据库连接时，实际是向数据库连接池申请一个数据库连接，而不会是创建一个新的数据连接。同样，conn.close()释放一个数据库连接时，也不是直接将连接关闭，而是将连接归还给数据库连接池。
```Java
// 数据库连接池配置
HikariConfig config = new HikariConfig();
config.setMinimumIdle(1);
config.setMaximumPoolSize(2);
config.setConnectionTestQuery("SELECT 1");
config.setDataSourceClassName("org.h2.jdbcx.JdbcDataSource");
config.addDataSourceProperty("url", "jdbc:h2:mem:test");
// 创建数据源
DataSource ds = new HikariDataSource(config);
Connection conn = null;
Statement stmt = null;
ResultSet rs = null;
try {
  // 获取数据库连接
  conn = ds.getConnection();
  // 创建 Statement
  stmt = conn.createStatement();
  // 执行 SQL
  rs = stmt.executeQuery("select * from abc");
  // 获取结果
  while (rs.next()) {
    int id = rs.getInt(1);
    ......
  }
} catch(Exception e) {
   e.printStackTrace();
} finally {
  // 关闭 ResultSet
  close(rs);
  // 关闭 Statement
  close(stmt);
  // 关闭 Connection
  close(conn);
}
// 关闭资源
void close(AutoCloseable rs) {
  if (rs != null) {
    try {
      rs.close();
    } catch (SQLException e) {
      e.printStackTrace();
    }
  }
}
```

从宏观上看，HiKariCP性能高，主要是体现在两个数据结构上，一个是FastList，另一个是ConcurrentBag。

### FastList
按照规范步骤，执行数据库操作之后，需要依次关闭ResuSet、Statement、Connection，但有时可能会只关闭Conection，而忘了关闭ResultSet和Statement。为了解决这个问题，最好的方式是关闭Connection时，能够自动关闭Statement，为了达到这个目的，Connection就需要跟踪创建的Statement，最简单的方法就是将创建的Statement保存在数组ArrayList里，这样当关闭Connection的时候，就可以一次将数组里所有的Statement关闭。

HiKariCP觉得ArrayList太慢了，因为conn.createStatement()时，调用ArrayList的add()方法是没问题的，但通过stmt.close()关闭Statement的时候，需要调用ArrayList的remove()方法是有优化的余地的。

例如Connection创建了6个Statement，分别是s1,s2....,s6，关闭是他们是逆序关闭的，而ArrayList的remove()方法是顺序查找的，这样删除就太慢了，因此需要改成逆序查找。

同时FastList还有一个优点就是get(int index)方法没有对index进行越界检查，HiKariCP能保证不越界。


### ConcurrentBag
让自己实现一个数据库连接池的话，最简单的方法就是用两个阻塞队列来实现：

```Java
// 忙碌队列，获取连接池从idel移到busy
BlockingQueue<Connection> busy;
// 空闲队列，关闭连接时从busy移到idle
BlockingQueue<Connection> idle;
```

然而HiKariCP并没有使用java中的阻塞队里，而是实现了一个叫ConcurrentBag的东西，他的设计最初来源于C#,hexin使用ThreadLocal避免部分并发问题。

ConcurrentBagzui 关键的属性有四个：
```java
// 用于存储所有的数据库连接的共享队列
CopyOnWriteArrayList<T> sharedList;
// 线程本地存储中的数据库连接
ThreadLocal<List<Object>> threadList;
// 等待数据库连接的线程数
AtomicInteger waiters;
// 分配数据库连接的工具，主要用于线程间传递数据
SynchronousQueue<T> handoffQueue;
```

当线程池创建了一个数据库连接时，通过调用ConcurrentBag的add()方法，逻辑很简单，就是讲这个连接加入到共享队列sharedList里，如果此时有线程在等待数据库连接，那么就通过handoffQueue将这个连接分配给等待的线程。
```Java
// 将空闲连接添加到队列
void add(final T bagEntry){
  // 加入共享队列
  sharedList.add(bagEntry);
  // 如果有等待连接的线程，
  // 则通过 handoffQueue 直接分配给等待的线程
  while (waiters.get() > 0
    && bagEntry.getState() == STATE_NOT_IN_USE
    && !handoffQueue.offer(bagEntry)) {
      yield();
  }
}
```
通过ConcurrentBag提供的borrow()方法，可以获取一个空闲的数据库连接，borrow()的主要逻辑是：
1. 首先检查线程本地存储是否有空闲连接，如果有，则返回一个空闲的连接
2. 如果线程本地存储中无空闲连接，则从共享队列获取
3. 如果共享队列中也没有空闲连接，则请求线程需要等待

这里线程本地存储中的连接是可以被其他线程窃取的，所以需要用CAS方法防止重复分配。

```Java
T borrow(long timeout, final TimeUnit timeUnit){
  // 先查看线程本地存储是否有空闲连接
  final List<Object> list = threadList.get();
  for (int i = list.size() - 1; i >= 0; i--) {
    final Object entry = list.remove(i);
    final T bagEntry = weakThreadLocals
      ? ((WeakReference<T>) entry).get()
      : (T) entry;
    // 线程本地存储中的连接也可以被窃取，
    // 所以需要用 CAS 方法防止重复分配
    if (bagEntry != null
      && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
      return bagEntry;
    }
  }

  // 线程本地存储中无空闲连接，则从共享队列中获取
  final int waiting = waiters.incrementAndGet();
  try {
    for (T bagEntry : sharedList) {
      // 如果共享队列中有空闲连接，则返回
      if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
        return bagEntry;
      }
    }
    // 共享队列中没有连接，则需要等待
    timeout = timeUnit.toNanos(timeout);
    do {
      final long start = currentTime();
      final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
      if (bagEntry == null
        || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
          return bagEntry;
      }
      // 重新计算等待时间
      timeout -= elapsedNanos(start);
    } while (timeout > 10_000);
    // 超时没有获取到连接，返回 null
    return null;
  } finally {
    waiters.decrementAndGet();
  }
}
```
释放连接需要调用ConcurrentBag提供的required()方法，该方法逻辑很简单，首先将数据库连接状态更改为STATE_NOT_IN_USE，之后查看是否有在等待的线程，如果有，则分配给等待线程；如果没有，则将该数据库连接保存在线程本地存储：

```Java
// 释放连接
void requite(final T bagEntry){
  // 更新连接状态
  bagEntry.setState(STATE_NOT_IN_USE);
  // 如果有等待的线程，则直接分配给线程，无需进入任何队列
  for (int i = 0; waiters.get() > 0; i++) {
    if (bagEntry.getState() != STATE_NOT_IN_USE
      || handoffQueue.offer(bagEntry)) {
        return;
    } else if ((i & 0xff) == 0xff) {
      parkNanos(MICROSECONDS.toNanos(10));
    } else {
      yield();
    }
  }
  // 如果没有等待的线程，则进入线程本地存储
  final List<Object> threadLocalList = threadList.get();
  if (threadLocalList.size() < 50) {
    threadLocalList.add(weakThreadLocals
      ? new WeakReference<>(bagEntry)
      : bagEntry);
  }
}
```
