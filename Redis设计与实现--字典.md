# Redis设计与实现--字典

* 在字典中，一个key可以和一个value进行关联，形成键值对
    - 例如 SET msg "hello world"就是在数据库中创建了一个建为msg，值为hello world的键值对，这个键值对就是保存在代表数据库的字典里面的。
    -  哈希 、string都是通过字典实现的
* 哈希表

```C
typedef struct dictht {
    //哈希表数组,数组中每一个元素都是一个指向dictEntry结构的指针
    dicEntry **table;

    //哈希表大小
    unsigned long size;

    //哈希表大小掩码，用于计算索引、总是等于size-1
    unsigned long sizemask;

    //该哈希表已有节点的数量
    unsigned long used;
} dictht;

typedef struct dictEntry {
    //键
    void *key;

    //值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    //指向下一个哈希表节点，形成链表,用于解决哈希冲突
    struct dictEntry *next;
}

typedef struct dict {
    //类型特定函数
    dictType *type;

    //私有函数
    void *privdata;

    //哈希表
    dictht ht[2];

    //rehash索引，当rehash不再进行时，值为-1
    int trehashidx;
} dict;
```

* 解决哈希冲突：通过dictEntry的next指针将冲突的键值对连在一起，dictEntry节点没有指向链表尾部的指针，所以为了速度考虑，新来的dictEntry放在链表头。
* rehash： rehash是渐进式的、分而治之的。为ht[1]分配空间，在字典中维持一个索引器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。热rehash期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作外，还会顺带将ht[0]的哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash完成，rehashidx的值加一。随着字典操作的不断进行，所有的数据都会迁移完成，rehashidx的值置为-1。在rehash期间，字典的删除、查找、更新等操作会在两个表上进行（查数据时，现在ht[0]上找，找不到再到ht[1]上找），添加操作只会在ht[1]上进行，保证ht[0]的数据只会减少，不会增加，并随着rehash操作的进行而变成空表。
    - 为ht[1]分配空间
        - 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used * 2的2^n;
        - 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的2^n;
    - 将ht[0]的键值对重新计算哈希值和索引值，迁移到ht[1]
    - 释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新建一个空白哈希表，为下一次rehash做准备
* 哈希表的扩展与收缩操作
    - 当满足以下条件之一时，开始扩展操作(负载因子 load_fator=ht[0].used / ht[0].size)：
        - 服务器目前没有在执行BGSAVE或者BGREWRITEAOF，并且负载因子大于等于1
        - 服务器目前在执行BGSAVE或者BGREWRITEAOF，并且负载因子大于等于5
    - 当负载因子小于0.1时，开始哈希表收缩操作。
