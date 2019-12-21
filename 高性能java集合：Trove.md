# 高性能java集合：Trove

rove是轻量级实现java.util Collections API的第三方开源项目
* trove相比jdk原生的集合类有三个优势:
    1、更高的性能
    2、更底的内存消耗
    3、除了实现原生Collections API并额外提供更强大的功能。
* 例如:支持不同值类型的map
```java
public static void main(String[] args) throws ParseException {
        TIntObjectHashMap map = new TIntObjectHashMap();

        map.put(1, 2);
        map.put(12,"2JJJ");
        map.put(3, new User());

        System.out.println(map.get(12));
    }

    static class User {
        private String name = "jie";
        private int age = 12;

        @Override
        public String toString() {
            return "name=" + name + ", age=" + age;
        }
    }
}
```
