#SpringMVC
* springmvc调用过程![springmvc调用过程](/Users/jingjie/Documents/markdown/images/spring/springmvc)
* 控制器实现：这两种控制器是在springmvc里写死的![两个控制器](/Users/jingjie/Documents/markdown/images/spring/springmvc/springmvc控制器.png)
    - BeanNameUrlHandlerMapping 实现了controller接口和继承AbstractController
    - RequestMappingHandlerMapping 注解@Controller
* 三种控制器适配器：![三种控制器适配器](/Users/jingjie/Documents/markdown/images/spring/springmvc/springmvc控制器适配器.png)

* 执行handler：执行applyPreHandler（拦截器）->handle（业务处理逻辑）->afterHandle
*
