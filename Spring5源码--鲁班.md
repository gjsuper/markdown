
- context.refresh()
    - prepareRefrash
    - prepaeBeanFactory(配置其标准的特征，比如上下文加载器ClassLoader和post-processor回调)


- ConfigurationClassProcessor
- BeanFactoryPostProcessor:在初始化类之前修改类的scope、是否单例等等
- ImportBeanDefinitionRegistrar 自定义注册BeanDefination
