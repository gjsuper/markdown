##深入理解Java虚拟机-性能监控和故障处理工具

### jps:虚拟机进程状况工具
* -m:输出虚拟机进程启动时传递给主类main的参数
* -l:输出主类全名，如果是jar包，输出jar包路径
* -v:输出虚拟机进程启动时的JVmware参数

### jstat:虚拟机统计信息监视工具
&emsp;&emsp;监视虚拟机各种运行状态信息的命令行工具，显示虚拟机中进程中的类加载、内存、垃圾收集、JIT编译等运行参数。
&emsp;&emsp;jstat格式：jstat [option vmid [interval[s|ms] [count]]]
&emsp;&emsp;option代表用户需要查询的信息，包括：类加载、垃圾收集、运行期编译情况
&emsp;&emsp;eg: jstat -gc 2764 500 10:每500ms查询一次进程2764,共查询10次。

###jinfo:java配置信息工具
&emsp;&emsp;实时查看和调整虚拟机各项参数。
&emsp;&emsp;eg：jinfo -flag CMSInitiationOccupancyFraction 1444
&emsp;&emsp;&emsp;&emsp;查询进程1444的CMSInitiationOccupancyFraction参数
&emsp;&emsp;&emsp;&emsp;输出结果：-XX：CMSInitiationOccupancyFraction=85

###jmap：Java内存映像工具
&emsp;&emsp;生成堆转存快照
&emsp;&emsp;eg: jmap -dump:format=b,file=eclipse.bin 3500

###jhat:虚拟机堆内存转存快照分析工具
&emsp;&emsp;与jmap搭配使用，一般不直接使用
&emsp;&emsp;eg: jhat:eclipse.bin

###jstack:Java堆栈跟踪工具
&emsp;&emsp;生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，其主要目的是定位线程长时间停顿的原因，如线程死锁，死循环，请求外部资源的长时间等待等都是导致线程长时间停顿的原因。
* -F:当正常输出的请求无法响应时，强制输出线程堆栈
* -l:出堆栈外，显示关于锁的附加信息
* -m:如果调用本地方法的话，可以显示C/C++的堆栈

###HSDIS:JIT生成代码反汇编

###class文件
&emsp;&emsp;常量池：主要存放两大类常量：字面量和符号引用。class文件不会保存各个方法和字段的最终内存布局信息，虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建或运行时解析、翻译到具体的内存地址中。
&emsp;&emsp;字面量：如文本字符串、声明为final的常量值等等。
&emsp;&emsp;符号引用：类和接口的全限定名、字段的名称和符号描述、方法的名称和描述。