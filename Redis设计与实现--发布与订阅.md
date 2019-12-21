# Redis设计与实现--发布与订阅（Redis设计与实现第十八章）
* 频道的订阅与发布：利用SUBSCRIBE订阅某个频道，redis将频道订阅关系保存在pubsub_channels字典里
```C
    struct redisServer {
        //...

        //保存所有频道的订阅关系，键是频道的名字，值是一个链表，里面记录了所有订阅这个频道的客户端
        dict *pubsub_channels;

        //保存所有模式订阅关系
        list *pubsub_patterns;

        //...
    }
```

[pubsub_channels字典实例](/images/redis/pubsub_channels字典实例.png)
[订阅模式链表](/images/redis/订阅模式链表.png)


pubsub_patterns:

```C
typedef struct pubsubPattern {
    //订阅模式的客户端
    redisClient *client;

    //被订阅的模式
    robj *pattern;
}
```
