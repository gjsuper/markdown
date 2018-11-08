#Guava源码--优雅的代码

* MoreObjects:
    * toString:
```java
@Override
   public String toString() {
     boolean omitNullValuesSnapshot = omitNullValues;
     String nextSeparator = "";//如何处理第一个变量
     StringBuilder builder = new StringBuilder(32).append(className).append('{');
     for (ValueHolder valueHolder = holderHead.next;
         valueHolder != null;
         valueHolder = valueHolder.next) {
       Object value = valueHolder.value;
       if (!omitNullValuesSnapshot || value != null) {
         builder.append(nextSeparator);
         nextSeparator = ", ";//如何处理第一个变量

         if (valueHolder.name != null) {
           builder.append(valueHolder.name).append('=');
         }
         if (value != null && value.getClass().isArray()) {
           Object[] objectArray = {value};
           String arrayString = Arrays.deepToString(objectArray);//数组转String，eg:"[a,b,c]"
           builder.append(arrayString, 1, arrayString.length() - 1);
         } else {
           builder.append(value);
         }
       }
     }
     return builder.append('}').toString();
   }
```

* ComparisonChain:比较链,如果值相等，就一直往下比较，一直比较出结果为止。

用例：

```java
Ordering<Person> personOrdering = new Ordering<Person>() {
      @Override
      public int compare(Person p1, Person p2) {
         return ComparisonChain.start().compare(p1.getAge(), p2.getAge())
               .compare(p1.getName(), p2.getName())
               .compare(p1.getSex(), p2.getSex()).result();
      }
   }.nullsFirst();
   Collections.sort(persons, personOrdering);
```
源码：
```java
public ComparisonChain compare(Comparable left, Comparable right) {
  return classify(left.compareTo(right));
}

//LESS GREATER ACTIVE都是ComparisonChain
ComparisonChain classify(int result) {
          return (result < 0) ? LESS : (result > 0) ? GREATER : ACTIVE;
        }
```

* equals：当需要进行null检查时，贼好用

```java
Objects.equal(null, "a");//returns false
```

* Sets：牛逼的集合处理API，提供交集、并集、差集的处理接口
* Maps：两个map的交集、左差集、右差集。

```java
Map<String, Integer> left = ImmutableMap.of("a", 1, "b", 2, "c", 3, "e", 4);
Map<String, Integer> right = ImmutableMap.of("a", 2, "b", 2, "c", 4, "d", 5);
MapDifference<String, Integer> diff = Maps.difference(left, right);
System.out.println(diff.entriesInCommon()); // {"b" => 2},key和value都相同
System.out.println(diff.entriesOnlyOnLeft()); // {"e" => 4}，key在右侧不存在
System.out.println(diff.entriesOnlyOnRight()); // {"d" => 5}，key在左侧不存在
System.out.println(diff.entriesDiffering());// {a=(1, 2), c=(3, 4)}
,key一样，值不相同
Map entriesDiffering = diff.entriesDiffering();
Set<Map.Entry<String, MapDifference.ValueDifference<Integer>>> rs = entriesDiffering.entrySet();

for(Map.Entry<String, MapDifference.ValueDifference<Integer>> e : rs) {
    System.out.println(e.getValue().leftValue());
}
```

* 使用和避免null：使用Optional除了赋予null语义，增加了可读性，最大的优点在于它是一种傻瓜式的防护。Optional迫使你积极思考引用缺失的情况，因为你必须显式地从Optional获取引用。直接使用null很容易让人忘掉某些情形
    - Optional<T>表示可能为null的T类型引用。一个Optional实例可能包含非null的引用（我们称之为引用存在），也可能什么也不包括（称之为引用缺失）。它从不说包含的是null值，而是用存在或缺失来表示。但Optional从不会包含null值引用。
* Ordering:流畅风格的比较器
* Multiset:可以以两种方式看待
    - 没有元素顺序限制的ArrayList<E>
        - add(E)添加单个给定元素
        - iterator()返回一个迭代器，包含Multiset的所有元素（包括重复的元素）
        - size()返回所有元素的总个数（包括重复的元素）
    - 当把Multiset看作Map<E, Integer>时，它也提供了符合性能期望的查询操作:
        - count(Object)返回给定元素的计数
        - HashMultiset.count的复杂度为O(1)，TreeMultiset.count的复杂度为O(log n)。
        - entrySet()返回Set<Multiset.Entry<E>>，和Map的entrySet类似。
        - elementSet()返回所有不重复元素的Set<E>，和Map的keySet()类似。
        - 所有Multiset实现的内存消耗随着不重复元素的个数线性增长。
* 使用MultiMap代替Map<E, List<T>>
    - 修改Multimap：对值视图集合进行的修改最终都会反映到底层的Multimap
    - MultiMap 相关工具：
        - invertFrom:翻转MultiMap


```java
ArrayListMultimap<String, Integer> multimap = ArrayListMultimap.create();
multimap.putAll("b", Ints.asList(2, 4, 6));
multimap.putAll("a", Ints.asList(4, 2, 1));
multimap.putAll("c", Ints.asList(2, 5, 3));

TreeMultimap<Integer, String> inverse = Multimaps.invertFrom(multimap, TreeMultimap.<Integer, String>create());

//注意我们选择的实现，因为选了TreeMultimap，得到的反转结果是有序的
/*
* inverse maps:
*  1 => {"a"}
*  2 => {"a", "b", "c"}
*  3 => {"c"}
*  4 => {"a", "b"}
*  5 => {"c"}
*  6 => {"b"}
*/

```

```java
Set<Person> aliceChildren = childrenMultimap.get(alice);

aliceChildren.clear();
aliceChildren.add(bob);
aliceChildren.add(carol);
```

* 当需要多个键做索引的时候，类似于Map<R, Map<C, V>>，使用table更好。

 * HashBasedTable：本质上用HashMap<R, HashMap<C, V>>实现；
 * TreeBasedTable：本质上用TreeMap<R, TreeMap<C,V>>实现

```java
Table<Vertex, Vertex, Double> weightedGraph = HashBasedTable.create();
weightedGraph.put(v1, v2, 4);
weightedGraph.put(v1, v3, 20);
weightedGraph.put(v2, v3, 5);

weightedGraph.row(v1); // returns a Map mapping v2 to 4, v3 to 20
weightedGraph.column(v3); // returns a Map mapping v1 to 20, v2 to 5
```
