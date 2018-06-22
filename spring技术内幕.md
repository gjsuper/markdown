#Spring技术内幕
##一.BeanFactory和ApplicationContext
&emsp;&emsp;spring的容器主要有两类：一是实现了BeanFactory**接口**的简单容器系列。二是ApplicationContext应用上下文，他作为容器的高级形态存在。
* [springIOC容器继承关系图]()
* [BeanFactory接口]()

####1.1编程式使用IOC容器:

```java
ClassPathResource res = new ClasspathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinition(res);
```

&emsp; 因此使用IOC容器的步骤：
1. 创建IOC配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息。
1. 创建一个BeanFactory，这里使用DefaultListableBeanFactory
2. 创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition，通过一个回调配置给BeanFactory

####1.2 IOC容器启动过程
1. 定位Resource。这里是指BeanDefinition的资源定位，不管是哪种形式的BeanDefinition，ResourceLoader都提供了统一的接口来定位。
2. 载入BeanDefinition。把用户定义好的Bean表示成IOC容器的内部数据结构，这个数据结构就是BeanDefinition。
3. 向IOC容器注册这些BeanDefinition。实际上是把BeanDefinition注入到一个HashMap中去。

####1.3 Resource定位过程
以FileSystemXmlApplicationContext为例：
FileSystemXmlApplicationContext构造函数使用refresh() -> 通过父类AbstractRefreshableApplication中的refreshFactory() -> 通过createBeanFactory()创建了一个IOC容器（DefaultListableBeanFactory），同时启动了loadBeanDefinitions来载入BeanDefinition。具体资源的载入在XmlBeanDefinitionReader读入BeanDefinition时完成，在XmlBeanDefinitionReader的基类AbstractBeanDefinitionReader中可以看到这个载入过程。

####1.4生成Bean
Spring提供了两种方式来实例化Bean，一种是通过BeanUtils,它使用了JVM的反射功能，另一种是通过CGLIB来生成。

####1.5 IOC容器中Bean的声明周期
1. Bean示例的创建
2. 为Bean示例设置属性
3. 调用Bean的初始化方法
4. 通过IOC容器使用Bean
5. 销毁Bean，关闭容器

####1.6 Bean销毁过程
1. 调用postProcessBeforeDestruction
2. 调用Bean的的destory方法
3. 调用Bean自定义的销毁方法

####1.7 Bean的初始化过程
1. bean.setBeanName
2. bean.setBeanClassLoader
3. bean.setBeanFactory
4. 调用BeanPostProcessors的postProcessBeforeInitialization（前置处理器）
5. 调用InitializingBean的afterPropertySet方法
6. 调用BeanPostProcessors的postProcessAfterInitialization（后置处理器）

####1.8 autoring注入顺序
1. AUTORED_BY_NAME
2. AUTORED_BY_TYPE
		autowire_by_name实现：先获取当前bean的属性名作为Bean的名字，然后从IOC容器索取Bean，最后把从容器取得的bean设置到当前bean的属性中去。
        
####1.9 Bean依赖检查
&emsp;&emsp; 在Bean的定义中设置dependency-check属性来指定依赖检查模式，用来检查Bean的所有属性是否被正确设置。检查时先查基础类型，再查应用类型

####1.10 ApplicationContextAware
&emsp;&emsp; 实现该接口，可在Bean中感知ApplicationContext