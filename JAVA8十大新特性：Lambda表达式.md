#JAVA8十大新特性：Lambda表达式

##一.Lambda表达式
1. 什么是Lambda表达式
	<pre> a function (or a subroutine) defined, and possibly called, without being bound to an identifier。
    一个不用被绑定到一个标识符上，并且可能被调用的函数
    简单理解就是 一段带有输入参数的可执行语句块
    </pre>
1. Lambda表达式用在何处
	* Lambda表达式用在替换以前广泛使用的匿名内部类，回调，例如时间响应器，传入Thread类的Runnable等等。例如：
    ```java
    Thread oldSchool = new Thread( new Runnable () {
        @Override
        public void run() {
            System.out.println("This is from an anonymous class.");
        }
    } );
    
    Thread gaoDuanDaQiShangDangCi = new Thread( () -> {
        System.out.println("This is from an anonymous method (lambda exp).");
    } );```
    * Lambda表达式与批处理
    集合类（包括List）现在都有一个forEach方法，对元素进行迭代（遍历），所以我们不需要再写for循环了。forEach方法接受一个函数接口Consumer做参数，所以可以使用λ表达式。例如：
   ```java
   for(Object o: list) { // 外部迭代
        System.out.println(o);
    }
    ```
    可以写为：
     ```java
     list.forEach(o -> {System.out.println(o);}); //forEach函数实现内部迭代
    ```
    
    还有可以用于Stream
    
1. Lambda表达式的类型
	* Lambda表达式的目标类型是“函数接口”，“函数接口”的定义是：如果一个接口只有一个显示声明的抽象方法，那么他就是一个函数接口。可以用@FunctionalInterface标注出来（也可以不标）。举例如下：
    ```java
    @FunctionalInterface
    public interface Runnable { void run(); }
    public interface Callable<V> { V call() throws Exception; }
    public interface ActionListener { void actionPerformed(ActionEvent e); }
    ```
    * 
1. Lambda表达式语法
 * Lambda表达式形象化表示如下：
 <pre>Parameters -> expression</pre>
 * 如果Lambda表达式中要执行多个语句块,需要将多个语句块以{}进行包装，如果有返回值，需要显示指定return语句，如下所示:
```java
Parameters -> {expressions;};```
 * 如果Lambda表达式不需要参数，可以使用一个空括号表示，如下示例所示:
 ```java
 () -> {for (int i = 0; i < 1000; i++) doSomething();};
 ```
 * Java是一个强类型的语言，因此参数必须要有类型，如果编译器能够推测出Lambda表达式的参数类型，则不需要我们显示的进行指定，如下所示，在Java中推测Lambda表达式的参数类型与推测泛型类型的方法基本类似:
 ```java
 String []datas = new String[] {"peng","zhao","li"};
Arrays.sort(datas,(String v1, String v2) -> Integer.compare(v1.length(), v2.length()));
```
上述代码中 显示指定了参数类型Stirng，其实不指定，如下代码所示，也是可以的，因为编译器会根据Lambda表达式对应的函数式接口Comparator<String>进行自动推断:
```java
String []datas = new String[] {"peng","zhao","li"};;
Arrays.sort(datas,(v1, v2) -> Integer.compare(v1.length(), v2.length()));
```
 *  如果Lambda表达式只有一个参数，并且参数的类型是可以由编译器推断出来的，则可以如下所示使用Lambda表达式,即可以省略参数的类型及括号:
 ```java
 Stream.of(datas).forEach(param -> {System.out.println(param.length());});
 ```
 * Lambda表达式的返回类型，无需指定，编译器会自行推断，说是自行推断
 * Lambda表达式的参数可以使用修饰符及注解，如*final*、*@NonNull*等
 
##二.函数式接口
1. 什么是函数式接口
	函数式接口具有两个主要特征，是一个接口，这个接口具有唯一的一个抽像方法，我们将满足这两个特性的接口称为函数式接口
1. Lambda表达式不能脱离目标类型存在，这个目录类型就是函数式接口，所下所示是一个样例：
```java
String []datas = new String[] {"peng","zhao","li"};
Comparator<String> comp = (v1,v2) -> Integer.compare(v1.length(), v2.length());
Arrays.sort(datas,comp);
Stream.of(datas).forEach(param -> {System.out.println(param);}); 
```
Lambda表达式被赋值给了comp函数接口变量
1. 函数式接口可以使用 *@FunctionalInterface* 标注，标注的好处是可以使知道这是一个函数式接口，同时在生成java doc时会进行显示标注。
2. 异常，如果Lambda表达式会抛出非运行时异常，则函数式接口也需要抛出异常，还是一句话，函数式接口是Lambda表达式的目标类型
3. 函数式接口可以提供多个抽象方法，但这些方法只能是Object类型里的已有方法

##三.方法引用
任何一个λ表达式都可以代表某个函数接口的唯一方法的匿名描述符。如果我们需要执行的代码在某些类中已经存在了，就不用我们写Lambda表达式，可以直接使用该方法。例如：
```java
String []datas = new String[] {"peng","Zhao","li"};
Arrays.sort(datas,String::compareToIgnoreCase);
Stream.of(datas).forEach(System.out::println);
```
方法引用的分类：
```java
Object:instanceMethod
Class:staticMethod
Class:instanceMethod
```

在java8中接口可以有默认方法，所以在在implements多个接口且这些接口存在相同的方法时，需要显示指定使用哪个接口的方法。例如：
```java
public class Sub implements Base1, Base2 {
	public void hello() {
    	Base1.super.hello(); //使用Base1的实现
    }
}
```


##四.构造方法引用
构造方法引用与方法引用类似，除了一点，就是构造方法引用的方法是new!以下是两个示例.
示例一：
```java
String str = "test";
Stream.of(str).map(String::new).peek(System.out::println).findFirst();
```
示例二：
```java
String []copyDatas = Stream.of(datas).toArray(String[]::new);
Stream.of(copyDatas).forEach(x -> System.out.println(x));
```

总结一下，构造方法引用有两种形式
```java
Class::new
Class[]::new
```

##参考文献:
1. [Lambda表达式](https://www.cnblogs.com/WJ5888/p/4618465.html)
2. [Java8 Lambda表达式教程](http://blog.csdn.net/ioriogami/article/details/12782141/)