#maven命令 
1. mvn clean:清除target目录，也就是class文件
2. mvn compile:编译代码，将class文件编译生成class文件
3. mvn test:执行单元测试，根目录下的src/test/java下的测试类都会执行
4. mvn package:web project->war包,java project -> jar包,父工程->pom,将项目打包在target目录下
5. mvn install：安装jar包,打包到本地仓库

-----------------------
#maven生命周期
maven存在三套生命周期，每套生命周期都是相互独立的，互不影响。
1. CleanLifeCycle:清理生命周期（mvn clean）
2. defaultLifecycle:默认生命周期(compile,test,package,install,deploy)
siteLifeCycle：站点生命周期(site)

---------------------------
#maven依赖冲突解决原则
1. 第一申明者优先原则
2. 短路径优先原则

-----------------------------
#maven拆分与聚合
1. 创建父工程，只有pom文件，定义项目需要的依赖信息
2. 将创建好的父工程发布到本地仓库
3. 创建子工程
4. 在子工程的pom文件中添加相互之间的依赖