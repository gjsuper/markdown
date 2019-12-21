# spring官方文档

### 多生命周期机制
* Multiple lifecycle mechanisms configured for the same bean, with different initialization methods, are called as follows:

    1. Methods annotated with @PostConstruct

    1. afterPropertiesSet() as defined by the InitializingBean callback interface

    1. A custom configured init() method

We recommend that you do not use the InitializingBean interface, because it unnecessarily couples the code to Spring. Alternatively, we suggest using the @PostConstruct annotation or specifying a POJO initialization method.**  In the case of XML-based configuration metadata, you can use the init-method attribute to specify the name of the method that has a void no-argument signature. With Java configuration, you can use the initMethod attribute of @Bean

* Destroy methods are called in the same order:

    1. Methods annotated with @PreDestroy

    1. destroy() as defined by the DisposableBean callback interface

    1. A custom configured destroy() method

也不推荐使用DisposableBean，推荐使用PreDestory

* 初始化方法与AOP代理的关系：
The Spring container guarantees that a configured initialization callback is called immediately after a bean is supplied with all dependencies. Thus, the initialization callback is called on the raw bean reference, which means that AOP interceptors and so forth are not yet applied to the bean. A target bean is fully created first and then an AOP proxy (for example) with its interceptor chain is applied. If the target bean and the proxy are defined separately, your code can even interact with the raw target bean, bypassing the proxy. Hence, it would be inconsistent to apply the interceptors to the init method, because doing so would couple the lifecycle of the target bean to its proxy or interceptors and leave strange semantics when your code interacts directly with the raw target bean.

* 利用BeanPostProcessor个性化定义Bean
BeanPostProcessor instances operate on bean (or object) instances. That is, the Spring IoC container instantiates a bean instance and then BeanPostProcessor instances do their work.

BeanPostProcessor:the post-processor gets a callback from the container both before container initialization methods (such as InitializingBean.afterPropertiesSet() or any declared init method) are called, and after any bean initialization callbacks.

* @Autowired可用于集合（list、map、set）

* You can also use @Autowired for interfaces that are well-known resolvable dependencies: BeanFactory, ApplicationContext, Environment, ResourceLoader, ApplicationEventPublisher, and MessageSource. These interfaces and their extended interfaces, such as ConfigurableApplicationContext or ResourcePatternResolver, are automatically resolved, with no special setup necessary. The following example autowires an ApplicationContext object:

```Java
public class MovieRecommender {

    @Autowired
    private ApplicationContext context;

    public MovieRecommender() {
    }

    // ...
}
```

        The @Autowired, @Inject, @Resource, and @Value annotations are handled by Spring BeanPostProcessor implementations. This means that you cannot apply these annotations within your own BeanPostProcessor or BeanFactoryPostProcessor types (if any). These types must be 'wired up' explicitly by using XML or a Spring @Bean method.


* @Primary 当有多个相同类型的Bean时，使用@Primary，@Autowired可以让该Bean注入使用了有@Primary注解的bean


```Java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}

public class MovieRecommender {

    @Autowired
    private MovieCatalog movieCatalog;

    // ...
}



或者：
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog" primary="true">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

* Qualifiers：匹配特定名称的bean

```Java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;
}

public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main")MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }
}


xml配置：
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>

```
 This implies that qualifiers do not have to be unique. Rather, they constitute filtering criteria. For example, you can define multiple MovieCatalog beans with the same qualifier value “action”, all of which are injected into a Set<MovieCatalog> annotated with @Qualifier("action").

 @Autowired applies to fields, constructors, and multi-argument methods, allowing for narrowing through qualifier annotations at the parameter level. By contrast, @Resource is supported only for fields and bean property setter methods with a single argument.


* 自定义自己的qualifier 注解

[自定义qualifier注解](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-autowired-annotation-qualifiers)

```Java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

    String genre();

    Format format();
}

public enum Format {
    VHS, DVD, BLURAY
}

public class MovieRecommender {

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Action")
    private MovieCatalog actionVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Comedy")
    private MovieCatalog comedyVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.DVD, genre="Action")
    private MovieCatalog actionDvdCatalog;

    @Autowired
    @MovieQualifier(format=Format.BLURAY, genre="Comedy")
    private MovieCatalog comedyBluRayCatalog;

    // ...
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Action"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Comedy"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="DVD"/>
        <meta key="genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="BLURAY"/>
        <meta key="genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

</beans>
```

* 自定义扫描过滤器:The following example shows the configuration ignoring all @Repository annotations and using “stub” repositories instead:

```Java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
//The following listing shows the equivalent XML:

<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>

```

* @Bean和@Component
    - @Bean采用CGLIB代理(CGLIB代理是在@Configuration类中的@Bean方法中调用方法或字段来创建对协作对象的bean元数据引用的方法)。@Bean的方法可以被声明为静态方法,但由于技术限制，对静态@Bean方法的调用永远不会被容器截获，即使在@Configuration类中(如本节前面所述)也是如此:CGLIB子类化只能覆盖非静态方法。因此，直接调用另一个@Bean方法具有标准的Java语义，从而直接从工厂方法本身返回独立的实例。且@Bean的方法不能使private和final的
    - @Component没有采用代理(在普通的@Component类中调用@Bean方法中的方法或字段具有标准的Java语义，不应用特殊的CGLIB处理或其他约束。)
* @Inject 与 @Autowired用法一样，但匹配名字是用的是@Named,而不是@Qualifire

* @Named and @ManagedBean: Standard Equivalents to the @Component Annotation



* 参数优先级：[参数查询优先级](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-property-source-abstraction)

    For a common StandardServletEnvironment, the full hierarchy is as follows, with the highest-precedence entries at the top:

    1. ServletConfig parameters (if applicable — for example, in case of a DispatcherServlet context)

    2. ServletContext parameters (web.xml context-param entries)

    3. JNDI environment variables (java:comp/env/ entries)

    5. JVM system properties (-D command-line arguments)

    6. JVM system environment (operating system environment variables)

    自定义参数：

    Most importantly, the entire mechanism is configurable. Perhaps you have a custom source of properties that you want to integrate into this search. To do so, implement and instantiate your own PropertySource and add it to the set of PropertySources for the current Environment. The following example shows how to do so:
```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```
In the preceding code, MyPropertySource has been added with highest precedence in the search. If it contains a my-property property, the property is detected and returned, in favor of any my-property property in any other PropertySource. The MutablePropertySources API exposes a number of methods that allow for precise manipulation of the set of property sources.
