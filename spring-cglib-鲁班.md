# Spring--cglib

* @Configuration作用 加了生成cglib代理，没加就不生成代理


![Configuration ](/images/spring/Configuration作用.png)

![cglib用法](/images/spring/cglib用法.png)

* 加了@Configuration会进入到下面的分支条件
[Conifguration会进入到的分支](/images/spring/Conifguration会进入到的分支.png)

* spring-cglib实现伪代码 [spring-cglib实现伪代码](/images/spring/spring-cglib实现伪代码.png)

* 判读方法的名字和调用该方法的方法名称和参数是否一致，如果一致就getBean，否则就new Bean
* factoryBean:存放bean的地方，做扩展（mybatis等等）
* beanFactory:生成bean

* 在userService前加上static，会打印两遍，不加只会打印一遍
