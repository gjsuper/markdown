#java日志体系
##jcl
* log4j可以直接记录日志，不需要依赖第三方的技术
* jcl:jakarta commons-logging,是Apache公司开源的抽象日志通用框架，不直接记录日志，但提供了记录日志的抽象方法即接口（info,debug,error...），依赖第三方技术jul（java.util.logging）
* jcl内置了四种log的名称，调用Logger logger = Logger.getLogger(TestLog4j.class);时 通过内置的四种类名去load一个日志class，如果load成功就直接new出来返回，如果不成功就循环第二个，直到成功为止。

[jcl内置的四种日志类名](/Users/jingjie/Documents/markdown/images/java日志/jcl内置日志.jpg)

##spring日志
* spring5日志技术实现：
    - 使用spring-jcl（spring改了jcl的代码）或者使用jcl来记录日志，spring-jcl设置默认使用JUL，但在静态域中会先去load Log4j 2.x，如果没有load到，再去load SLF4_LAL，如果没有load到，再去load SELF4J 1.7 API,如果都没load到就只能用jul。在获取log时，使用switch case去判断返回那个log。如果提出了spring-jcl，那就要加上jcl的依赖。如果是slf4j，则slf4j会通过绑定器去加载其他的日志工具，如果没有添加绑定器的依赖，则默认没有日志。
    - jcl不能直接日志，采用循环优先的原则来使用相应的log框架。
* spring4中没有spring-jcl，依赖的是原生的jcl，所以他按照jcl的规则来使用日志。
* 已经有了jul、log4j，为什么还要有jcl。因为如果没有jcl，我们自己开发的应用使用的日志记录工具和我们应用依赖的第三方框架采用的日志记录工具可能会不一样，比如引用中采用jul、第三方依赖采用log4j，那么我们就不方便通过统一的入口去控制日志的级别，和日志格式。 有了jcl，第三方框架只需加入jcl的依赖， 然后我们的应用在pom中加入自己想要的日志记录工具的依赖即可，这样我们的应用和第三方框架就采用的是统一的日志记录工具，可以通过jcl来统一控制日志级别和日志格式。
##slf4j
* slf4j：和jcl一样，也是一个门面，不直接记录日志，只提供接口，记录日志通过第三方日志工具来完成。
* 为什么有了jcl，还需要slf4j。因为jcl（它写死了日志类名）只支持自己内置定义的四中日志记录工具，无法使用其他日志工具，比如logback，而slf4j可以支持任意的日志记录工具。
* slf4j提供了绑定器来绑定不同的日志实现，比如对于jul，提供了jul-slf4j，对于log4j，提供了log4j-slf4j。如果市面上出现了新的日志工具，slf4j不需要修改源码，只需开发相应的依赖即可。
