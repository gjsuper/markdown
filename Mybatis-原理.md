#mybatis
## 基于java config 技术

## selectOne
* 调用的是selectList方法，如果返回结果只有一个，就直接返回，如果有多个就跑出异常
* mybatis初始化时，先将namespace + id（方法名）作为key（xml中），SQL作为value放入map，调用时利用namespace + 方法名去取，这也是为什么要求xml中的id要和方法名一样的原因
* 去除SQL后，mybatis的executor执行sql，默认的executor是cachingExecutor，因为mybatis自带一级缓存，但mybatis整合mybatis后，这个自带的一级缓存会失效。因为spring在调用完每次的方法后会关掉session，缓存就失效了，这是spring的aop实现的。如果不整合spring，就需要自己手动关。[mybatis](/Users/jingjie/Documents/markdown/images/mybatis/mybatis.jpg), [spring + mybtais](/Users/jingjie/Documents/markdown/images/mybatis/spring+mybatis.jpg)
* mybatis一级缓存是sqlSession级别，二级缓存是mapper级别
* mybatis加载mapping的方式：
    - resource ：相对路径加载xml
    - package：加载包
    - url：加载xml
    - class：单个类
