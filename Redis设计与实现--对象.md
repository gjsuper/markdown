# Redis设计与实现--对象（Redis设计与实现第八章）
* 对象类型和编码
    - redis 中每一个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性。

```C
typedef struct redisObject {
    //类型
    unsigned type:4;

    //编码
    unsigned encoding:4;

    //指向底层实现数据结构的指针
    void *ptr;

    //引用计数，用于垃圾回收和内存共享
    int refcount;

    //记录对象最后一次被访问的时间
    unsigned lru:22;
}
```

* 类型：redis数据库的键总是一个字符串对象，而值可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象
* 编码：通过encoding属性来设定对象所使用的编码，而不是为特性类型的对象关联一种固定的编码，极大地提升了Redis的灵活性和效率，因为Redis可以根据不用的场景为一个对象设置不同的编码，从而优化对象在某一个场景下的效率。eg：
    - 在列表对象包含的夙愿比较少时，Redis使用压缩列表作为列表对象的底层实现：
    - 因为压缩列表比起双端列表更节约内存，并且在元素数量较少时，在内存中以连续块方式保存的压缩列表比起双端列表可以更快被载入到缓存中。
    - 随着列表对象包含的元素越来越多，使用压缩列表来保存元素的优势逐渐消失时，对象就会将底层实现从压缩列表转向功能更强、也更适合保存大量元素的双端链表上面。
* 字符串对象：字符串对象的编码可以是int、raw或者embstr
    - 如果一个字符串对象保存的是整数值，并且这个整数值可以用long来表示，那么字符串会将整数值保存在字符串对象结构的ptr属性中（将void* 转换成long），type=REDIS_STRING，encoding =  RREDIS_ENCODING_INT

    [int编码的字符串对象](/images/redis/int编码的字符串对象.png)

    - 如果字符串对象保存的是一个字符串值，并且这个字符串值得长度大于32字节，那么字符串对象将使用SDS来保存这个字符串值，并将对象的编码设置为raw

    [raw编码的字符串对象.png](/images/redis/raw编码的字符串对象.png)

    * 如果字符串对象保存的是一个字符串值,并且这个字符串值的长度小于等于32字节，那么字符串对象将使用embstr编码的方式来保存这个字符串值
    * embstr和raw编码的区别：两者都是用redisObject和sdshdr来保存字符串对象
        - raw编码会调用两次内存分配分别来创建redisObject和sdshdr结构，而embstr编码则通过调用一次内存分配函数来分配一块连续的空间，空间中一次包含redisObject和sdshdr连个结构
        - embstr是只读的，raw可读可写
    * embstr的好处：
        - embstr编码将创建对象所需分配的次数从raw的两次将为一次
        - 释放embstr编码的字符串对象只需要调用一次内存释放函数，erraw需要两次
        - 因为embstr编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比起raw编码的字符串对象能更好的利用缓存带来的优势

        [embstr编码的字符串对象](/images/redis/embstr编码的字符串对象.png)
    - 用long double类型表示的浮点数也是作为字符串值来保存的。

|值|编码|
|:----:|:----:|
|long类型的整数   | int  |
|long double类型的浮点数   |  embstr或者raw |
|字符串值、长度太大而没法用long保存的整数、长度太大没法用long double表示的浮点数   |  embstr或者raw |

    - 编码的转换
        - int编码的字符串对象，如果做了APPEND操作，🙆她不再是一个整数值，而是一个字符串值，那么编码会从int变为raw（不会是embstr，因为embstr是只读的）
        - emnstr编码的字符串对象在执行修改命令之后，会变成raw编码的字符串对象

* 类表对象：ziplist和linkedlist
    - ziplist编码的对象：[ziplist编码的对象](/images/redis/ziplist编码的对象.png)
    - linkedlist编码的对象：[linkedlist编码的对象](images/redis/linkedlist编码的对象.png),其中three是以embstr编码的字符串值
    - 当对象同时满足一下两个条件时，列表对象使用ziplist，不能满足这两个条件的需要使用linkedlist编码，这两个值是可以修改的：
        - 列表中的每个字符串元素长度都小于64字节
        - 列表中的元素数量小于512个
    - 编码转换
        - 加入一个元素导致 该元素长度大于64字节，或者加入钙元素后，列表长度大于512，回事的列表里的元素都会被转移到双端列表里
- 哈希对象：编码方式可以是ziplist或者hashtable
    - 哈希对象的压缩列表底层实现：[哈希对象的压缩列表底层实现](/images/redis/哈希对象的压缩列表底层实现.png)
    - hashtable编码：hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都是用字典键值来保存 [hashtable编码的哈希对象](/images/redis/hashtable编码的哈希对象.png)
    - 编码转换：对于ziplist编码的对象来说，当使用ziplist编码的连个把条件任意一个不满足时，就会转移到字典里面，斌阿妈方式变为hashtable
        - 哈希对象保存的键和值字符串长度都小于64字节
        - 哈希对象保存的键值对的数量小于512个
- 集合对象：编码方式是intset或者hashtable
    - intset编码的集合对象：[intset编码的集合对象](/images/redis/intset编码的集合对象.png)
    - hashtable编码的集合对象：字典的键是一个字符串对象，值为NULL ![hashtable编码的集合对象](/images/redis/hashtable编码的集合对象.png)
    - 编码的转换：当集合对象那个同时满足两个条件时，对象使用intset编码，否则使用hashtable编码，两个值可修改
        - 集合对象保存的所有uansu都是整数值
        - 集合对象保存的元素谁啊不过不超过512个
- 有序集合对象：编码方式可以是ziplist或者skiplist
    - ziplist，压缩列表内的元素按照分值从小到大排序：[ziplist编码的有序集合](/images/redis/ziplist编码的有序集合.png)
    - skiplist编码：该编码方式使用zset结构作为底层实现，一个zset同时包含一个字典和一个跳跃表 [skiplist编码的有序集合对象](/images/redis/skiplist编码的有序集合对象.png)
    - skiplist和dict通过指针来共享相同的成员和分值，因为不会占用杜宇的空间

```C
typedef struct zset {
    zkipist *zsl;
    dict *dict;
} zset;
```
        - dict:保存了元素和分值的键值对，以O(1)来查找分值，但数据时无序的
        - skiplist：支持范围型操作
    - 编码的转换：当满足下面两个条件时，对象使用ziplist，两个值可以修改
        - 有序集合保存的元素小于128个
        - 所有的元素成员长度小于64字节
- 类型检查与命令多态
    - 类型检查的发现：根据redisObject结构的type来判断 数据库的键的值是否符合 输入的命令 所需的类型
    - 多态命令的实现：根据值对象的编码方式，选择正确的命令实现代码来执行命令，例如列表对象有ziplist和linkedlist两种编码方式，当执行LLEN命令时，会根据编码类型来使用对应的函数

- 内存回收：引用计数
- 对象共享：引用计数，目前Redis在初始化服务器时，会创建一万个字符串对象，这些对象包含了从0到9999的所有整数，用于对象共享
- 空转时长：lru属性。如果服务器打开了maxmemory选项，且内存回收算法使用volatile-lru或者allkeys-lru,那么当内存占用超过了maxmemory时，空转时长较高的键会优先被释放
