#Java8新特性：Stream
##一.什么是流
Stream 不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算的，它更像一个高级版本的 Iterator。Stream 就如同一个迭代器（Iterator），单向，不可往复，数据只能遍历一次，遍历过一次后即用尽了；而和迭代器又不同的是，Stream 可以并行化操作，迭代器只能命令式地、串行化操作。Stream 的并行操作依赖于 Java7 中引入的 Fork/Join 框架。
Java 的并行 API 演变历程基本如下：
1.0-1.4 中的 java.lang.Thread
5.0 中的 java.util.concurrent
6.0 中的 Phasers 等
7.0 中的 Fork/Join 框架
8.0 中的 Lambda

##二.生成流
* 从 Collection 和数组
	* Collection.stream()
	* Collection.parallelStream()
	* Arrays.stream(T array) or Stream.of()
* 从 BufferedReader
	* java.io.BufferedReader.lines()
* 静态工厂
	* java.util.stream.IntStream.range()
	* java.nio.file.Files.walk()
* 自己构建
	* java.util.Spliterator 
* 其他
	* Random.ints()
	* BitSet.stream()
	* Pattern.splitAsStream(java.lang.CharSequence)
	* JarFile.stream()

##三.构建流的常用方法
```java
// 1. Individual values
Stream stream = Stream.of("a", "b", "c");

// 2. Arrays
String [] strArray = new String[] {"a", "b", "c"};
stream = Stream.of(strArray);
stream = Arrays.stream(strArray);

// 3. Collections
List<String> list = Arrays.asList(strArray);
stream = list.stream();
```

##四.某些API的用法
* flatMap:flatMap 把 input Stream 中的层级结构扁平化，就是将最底层元素抽出来放到一起，最终 output 的新 Stream 里面已经没有 List 了，都是直接的数字。
```java
Stream<List<Integer>> inputStream = Stream.of(
Arrays.asList(1),
Arrays.asList(2, 3),
Arrays.asList(4, 5, 6)
);
Stream<Integer> outputStream = inputStream.
flatMap((childList) -> childList.stream());
```

* foreach和peek:
	forEach 是 terminal 操作，因此它执行后，Stream 的元素就被“消费”掉了
    peek属于intermediate操作
* optional
```java
public static void print(String text) {
 	// Java 8
 	Optional.ofNullable(text).ifPresent(System.out::println);
 	// Pre-Java 8
 	if (text != null) {
 	System.out.println(text);
 	}
 }
public static int getLength(String text) {
 	// Java 8
	return Optional.ofNullable(text).map(String::length).orElse(-1);
 	// Pre-Java 8
	// return if (text != null) ? text.length() : -1;
 };
```
在更复杂的 if (xx != null) 的情况中，使用 Optional 代码的可读性更好，而且它提供的是编译时检查，能极大的降低 NPE 这种 Runtime Exception 对程序的影响，或者迫使程序员更早的在编码阶段处理空值问题，而不是留到运行时再发现和调试。
Stream 中的 findAny、max/min、reduce 等方法等返回 Optional 值。还有IntStream.average() 返回 OptionalDouble 等等。

* reduce
这个方法的主要作用是把 Stream 元素组合起来。它提供一个起始值（种子），然后依照运算规则（BinaryOperator），和前面 Stream 的第一个、第二个、第 n 个元素组合。从这个意义上说，字符串拼接、数值的 sum、min、max、average 都是特殊的 reduce。例如 Stream 的 sum 就相当于
Integer sum = integers.reduce(0, (a, b) -> a+b); 或
Integer sum = integers.reduce(0, Integer::sum);
```java
// 字符串连接，concat = "ABCD"
String concat = Stream.of("A", "B", "C", "D").reduce("", String::concat); 
// 求最小值，minValue = -3.0
double minValue = Stream.of(-1.5, 1.0, -3.0, -2.0).reduce(Double.MAX_VALUE, Double::min); 
// 求和，sumValue = 10, 有起始值
int sumValue = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);
// 求和，sumValue = 10, 无起始值
sumValue = Stream.of(1, 2, 3, 4).reduce(Integer::sum).get();
// 过滤，字符串连接，concat = "ace"
concat = Stream.of("a", "B", "c", "D", "e", "F").
 filter(x -> x.compareTo("Z") > 0).
 reduce("", String::concat);
```
上面代码例如第一个示例的 reduce()，第一个参数（空白字符）即为起始值，第二个参数（String::concat）为 BinaryOperator。这类有起始值的 reduce() 都返回具体的对象。而对于第四个示例没有起始值的 reduce()，由于可能没有足够的元素，返回的是 Optional，请留意这个区别。

* Stream.generate
	在构造海量测试数据的时候了可以使用
	* 生成 10 个随机整数
	```java
    Random seed = new Random();
Supplier<Integer> random = seed::nextInt;
Stream.generate(random).limit(10).forEach(System.out::println);
//Another way
IntStream.generate(() -> (int) (System.nanoTime() % 100)).
limit(10).forEach(System.out::println);
    ```
    * 自实现 Supplier
    
    ```java
    Stream.generate(new PersonSupplier()).limit(10).
	forEach(p -> System.out.println(p.getName() + ", " + p.getAge()));
    
	private class PersonSupplier implements Supplier<Person> {
 		private int index = 0;
 		private Random random = new Random();
 		@Override
 		public Person get() {
 		return new Person(index++, "StormTestUser" + index, random.nextInt(100));
 		}
}
    ```
    * 生成一个等差数列
	```java
    Stream.iterate(0, n -> n + 3).limit(10). forEach(x -> System.out.print(x + " "));.
    output: 0 3 6 9 12 15 18 21 24 27
    ```
* 用 Collectors 来进行 reduction 操作

		**未完待续**

##五.参考文献
* [Java 8 中的 Streams API](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/)