#Java8函数式编程
    判断一个操作是惰性求值还是及早求值：只需要看他的返回值，如果返回值是一个Stream，那么就是惰性求值，如果返回值是另一个值或者为空，那么就是及早求值。

* StreamOf:生成流（Stream.of("a", "b", "c")）
* collect(toList())
* map:将一个流中的值转化成一个新的流（将一种类型的值转化为另一种类型）
* flatMap：将多个Stream连接成一个Stream
* 重构：找出长度大于1min的曲目：[利用stream重构](/Users/jingjie/Documents/markdown/images/java8/Stream重构.jpg)
* reduce:没有初始值的情况下，reduce第一步使用Stream的前两个元素。
* 方法引用：Classname::mathodName。虽然这是一个方法，但不需要再后面加括号，因为这里并不调用该方法，只是提供了和Lambda表达式等价的一种结构，在需要的时候才会调用。
* 拼接字符串并添加前后缀：

```java
String result = artista.stream
    .map(Artist:getName)
    .collect(Collectors.joining(", ", "[", "]"));
```

* 计算list中每个Item的数量并分组：

```java
public Map<Artist, Long> numberOfAlbums(Stream<Album> albums) {
    return albums.collect(groupingBy(album -> album.getMainMusician(), counting()));
```

* mappping允许在收集器的容器上执行类似map的操作。但是需要知名使用什么样的集合来存储结果。eg:使用收集器求每个艺术家的专辑名：
```java
public Map<Artist, List<String>> nameOfAlbums(Stream<Album> albums) {
    return balbums.collect(groupingBy(Album::getMainMusician, mapping(Album::getName, toList)))
}
```

* 统计信息

```java
public static void printTrackLengthStatistics(Album album) {
    IntSummaryStatistics trackLengthStats = album.getTracks()
                        .mapToInt(track -> track.getLength())
                        .summaryStatistics();
         System.out.printf("Max: %d, Min: %d, Ave: %f, Sum: %d",
                           trackLengthStats.getMax(),
                           trackLengthStats.getMin(),
                           trackLengthStats.getAverage(),
                           trackLengthStats.getSum());
}
```

* partitionBy:Map<Boolean, Object>
```java
public Map<Boolean, List<Artist>> bandsAndSoloRef(Stream<Artist> artists) {
    return artists.collect(partitioningBy(Artist::isSolo));
}
```

* reduce和StringCombiner：
```java
    StringCombiner combined = artists.stream.map(Artist::getName).reduce(new StringCombiner(", ", "[", "]"), StringCombiner::add, StringCombiner::merge);

    String res = combined.toString();

    String result =artists.stream()
                   .map(Artist::getName)
                   .collect(Collectors.joining(", ", "[", "]"));
```

* 并行化处理时避免使用无法判断结尾的数据结构（LinkedList，Streams.iterate）和有状态的操作（sored、limit、distinct）
* 数组的并行化操作：
    - parallelPrefix：任意指定一个函数，计算数组的和
    - parallelSetAll：使用lambda表达式更新数组元素
    - parallelSort：并行化对数组元素排序
* stream 中的peak可以让你查看每个值，并且能继续操作流。
* stream 中的allMatch返回一个boolean值，判断流中的所有元素是否都满足某个条件，如果都满足返回true，否则返回false。
*
