##Java虚拟机-Synchronized
&emsp;&emsp;Synchronized可用于修饰方法和代码块。
```Java
public class SynchronizedDemo {
     //同步方法
    public synchronized void doSth(){
        System.out.println("Hello World");
    }

    //同步代码块
    public void doSth1(){
        synchronized (SynchronizedDemo.class){
            System.out.println("Hello World");
        }
    }
}
```

&emsp;&emsp;反编译：
```Java
public synchronized void doSth();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return

public void doSth1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #5                  // class com/hollis/SynchronizedTest
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #3                  // String Hello World
        10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          23
        18: astore_2
        19: aload_1
        20: monitorexit
        21: aload_2
        22: athrow
        23: return
```
&emsp;&emsp;对于同步方法，JVM采用ACC_SYNCHRONIZED标记符来实现同步。方法级的同步是隐式的。同步方法的常量池会有一个ACC_SYNOCHRINIZED标志。当某个线程要访问该方法时，会检查是否有ACC_SYNOCHRONIZED，如果有设置，则需要先获得监视器锁，然后开始执行方法，方法结束后再释放监视器锁。这时如果其他线程请求执行方法，会因为无法获得监视器锁而被阻断住。如果在方法执行过程中发生异常并且在方法内没有出来该异常，那么在异常被抛到方法外之前监视器所会被自动释放。</br>
&emsp;&emsp;对于同步代码块，JVM采用monitorenter、monitorexit两个指令来实现同步。每个对象维护着一个记录着加锁次数的计数器。未被锁定的对象的计数器为0，当一个对象获得锁（monitorenter）后，该计数器自增1，当同一个线程再次获得该对象的锁，计数器再次自增。当同一个线程释放锁（执行monitorexit）的时候，计数器自减。当计数器为0时，锁会被释放。</br>
&emsp;&emsp;**ACC_SYNCHRONIZED、monitorenter、monitorexit都是基于monitor实现的，monitor基于C++实现，由ObjectMonitor实现。ObjectMonitor类中提供了enter、exit、wait、notify、notifyAll等。Synchronized加锁的时候会调用ObjectMonitor的enter方法，解锁的时候回调用exit方法。**

* 原子性
* 可见性：在解锁之前，必须将变量同步到主存。
* 有序性：Synchronized无法禁止指令重排和处理器优化。as-if-serial：不管怎么重排，单线程的执行结果不能被改变。编译器和处理器无论怎么优化，都必须遵循as-if-serial语义。**由于Synchronized保证了单线程执行，因此可以保证有序性。**
