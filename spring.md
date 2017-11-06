#spring
##&emsp;一.注解
1. @Autowired按类型注入，依赖spring,如果想按照名称来转配注入，则需要结合@Qualifier一起使用,不需要set方法；
2. @Resource默认是按照名称来装配注入的，只有当找不到与名称匹配的bean才会按照类型来装配注入；@Resource注解是又J2EE提供，而@Autowired是由Spring 提供，故减少系统对spring的依赖建议使用
---
##&emsp;二.xml配置
1. bean中id和name的区别：id中不能有特殊字符，name中可以有

---
##&emsp;三.aop
###&emsp;3.1 动态代理
1. JDK动态代理：针对有接口的情况。创建接口实现类的代理对象
2. cglib动态代理：针对没有接口的情况。创建类的子类的代理对象
---
###&emsp;3.2 术语
1. 连接点：类里面可以被增强的方法
2. 切入点: 实际被增强的方法称为切入点
3. 通知/增强: 实际要扩展出的功能叫通知，例如：日志功能
4. 切面: 本身是一种操作，把增强应用到切入点上的过程就叫切面

###&emsp;3.3 aop操作
1. [aop的xml配置](images/spring/aop/aop_xml.png)
2. [aop运行](images/spring/aop/run_aop.png)