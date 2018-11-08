#Stream最佳实践

* 去重，过滤相同对象:根据特定条件过滤对象
```java
List<PO> result = Lists.newArrayList(PO1, PO2, PO3).stream()
            .collect(Collectors.collectingAndThen(
                    Collectors.toCollection(() -> new TreeSet<>(Comparator.comparing(PO::getContractId))),
                    ArrayList::new
            ));
```
* 统计频次
```java
Map<Integer, Long> map = Lists.newArrayList("1", "1", "2", "3").stream().collect(Collectors.toMap(w -> w, w -> 1, Integer::sum));

Map<Integer, Long> map = list.stream().collect(Collectors.groupingBy(p -> p,Collectors.counting()));
```
* 分类:值为list（key <--> List)
```java
List<String> NON_WORDS = new ArrayList<String>() {{
                add("the"); add("and"); add("of"); add("to"); add("a");
                add("i"); add("it"); add("in"); add("or"); add("is");
                add("d"); add("s"); add("as"); add("so"); add("but");
                add("be"); }};

Map<Integer, List<String>> map = NON_WORDS.stream().collect(Collectors.groupingBy(String::length, Collectors.mapping(w -> w, Collectors.toList())));

```

* 分组
```java
Map<Integer, List<Apple>> groupBy = appleList.stream().collect(Collectors.groupingBy(Apple::getId));
```

* reduce:
```java
从一个作为累加器的初始值开始，利用 BinaryOperator 与流 中的元素逐个结合，从而将流归约为单个值累加
int totalCalories = menuStream.collect(reducing(0, Dish::getCalories, Integer::sum));
```

* list转map

```java

* toMap 如果集合对象有重复的key，会报错Duplicate key .... *  
* apple1,apple12的id都为1。 *  
* 可以用 (k1,k2)->k1 来设置，如果有重复的key,则保留key1,舍弃key2 *
Map<Integer, Apple> appleMap = appleList.stream().collect(Collectors.toMap(Apple::getId, a -> a,(k1,k2)->k1));

```
