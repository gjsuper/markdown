#spring AOP
* JDK动态代理、CGLIB代理：通过源码分析，可以知道是在初始化容器时期织入
* SpringAOP和AspectJ的关系：spring AOP 提供两种编程风格，
    - 利用aspectj实现springAOP @AspectJ
    - Schema-based AOP support ：xml
* SpringAOP如何使用JDK动态代理和cglib：
    - 如果proxyTargetClass == true，永远使用cglib
    - 如果proxyTargetClass == false，没有接口使用cglib，有接口使用jdk动态代理
* IOC容器是一个ConcurrentMap叫singletonObjects，他的key是beanName（例如dao），value是代理类。

* spring初始化：![spring初始化的各种方式](/Users/jingjie/Documents/markdown/images/spring/aop/spring初始化的各种方式.jpg)

* AOP：传统的oop是至上而下的，会产生横切性问题，例如日志记录，权限管理，这些横切性问题。。。。。![什么是aop](/Users/jingjie/Documents/markdown/images/spring/aop/什么是AOP.jpg)

* 为什么jdk动态代理只能代理接口实现类，因为jdk动态代理类继承了proxy，而java只能单继承，所以只能支持接口实现类的动态代理。
