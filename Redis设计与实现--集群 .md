# Redis设计与实现--集群（Redis设计与实现第十七章）

* 集群数据结构：每个节点都会使用clusterNode结构来记录自己的状态，并为其他节点都创建一个clusterNode，以次来记录其他节点的状态
```C
struct clusterNode {
    //创建节点时间
    mstime_t ctime;

    //节点名称,40字节
    char name[REDIS_CLUSTER_NAMELEN];

    //节点标识(主从节点、在线离线)
    int flags;

    //是谁的从节点
    clusterNode *slaveof;

    //节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;

    //节点的IP地址
    char ip[REDIS_IP_STR_LEN];

    //节点的端口号
    int port;

    //保存连接节点的相关信息
    clusterLink *link;

    //slots和numslots记录了属性记录了节点负责处理那些节点，如果对于的二进制位为1，说明节点负责处理该槽，同时会告诉其他节点自己负责哪些槽
    unsigned char slots[16384/8]
    int numslots;

    //这是个主节点，记录正在复制这个主节点的从节点的数量
    int numslave;

    //每个数组指向一个正在复制这个主节点的从节点的clusterNode,数量与numslaves相同
    struct clusterNode **salves;

    //一个链表，记录了所有其他节点对该节点的下线报告
    struct clusterNodeFailReport {
        struct clusterNode *node;//报告目标节点下线的节点
        mstime_t time;//最后一次从节点node收到下线报告的时间，用于检查下线报告是否过期，与当前时间差太多的下线报告会被删除
    }
    list *fail_reports;

}
```

```C
struct clusterLink {
    //连接的创建时间
    mstime_t ctime;

    //TCP套接字描述符
    int fd;

    //输出缓冲区，保存着等待发送给其他节点的信息
    sds sndbuf;

    //输出缓冲区，保存着从其他节点收到的消息
    sds rcvbuf;

    //与这个连接相关的节点
    struct clusterNode *node;
}
```

每个节点保存着一个clusterState节点，这个结构记录了当前节点的视角下，集群所处的状态
```C
typedef struct clusterState {
    //指向当前节点的指针
    clusterNode *mysql;

    //集群当前的配置纪元，用于实现故障转移
    uint64_t current_epoch;

    //集群当前的状态：在线还是离线
    int state;

    //集群中至少处理着一个槽的节点的数量
    int size;

    //集群节点名单（包括myslf）,键为节点名称，值为clusterNode结构
    dict *nodes；

    //记录16384个槽的指派信息,如果slots[i]==myself，那么这个槽就是由自己负责的
    clusterNode *slots[16384];

    //跳跃表，保存槽和键的关系
    zskiplist *slots_to_keys;

    //记录当前节点正在从其他节点导入的槽：
    clusterNode *importing_slots_from[16384];

    //记录当前节点正在迁移至其他节点的槽
    clusterNode *migrating_slots_to[16384];

    //如果这是一个从节点，这个指针指向主节点
    struct clusterNode *slaveof;

    //...
} clusterState;
```

* 槽指派：Redis集群通过分片的方式来保存书酷酷的键值对：集群的整个数据库被分为16384个槽，数据库中的每个键都属于槽中的一个，每个节点可以处理最多16384个槽。16384个槽都有节点处理时，集群处于上线状态，相反，任何一个槽没被处理，就处于下线状态
* 在集群中执行命令：当所有的槽都被分配时，集群就处于上线状态，客户端就可以给服务器发命令了：[判断客户端是否需要转向](/images/redis/判断客户端是否需要转向.png)
    - 计算键属于哪个槽：

```C
    def slot_number(key):
        return CRC16(key) & 16383;
```
    - 节点数据库的实现：与单机基本一致，但是集群只能使用0数据库：[节点7000的数据库](/images/redis/节点7000的数据库.png)
    - slots_to_keys跳跃表每个节点的分值都是一个槽号，而每个节点的成员都是一个数据库键：
        - 每当节点往数据库中添加一个新的键值对时，节点就会将这个键以及键的槽号关联到slots_to_keys跳跃表，删除键值对时，就从跳跃表中移除
        - 利用slots_to_keys,节点可以方便的对属于某个或某些槽的所有数据库键进行批量操作，例如CLUSTER GETKEYSINSLOT <slot> <count>命令最多返回count个属于槽slot的数据库键，这就是通过slots_to_keys来实现的 [slots_to_keys跳跃表](/images/redis/slots_to_keys跳跃表.png)
- 重新分片：将指派给某个节点的槽分配给其他节点
    -迁移键的过程:[迁移键的过程](/images/redis/迁移键的过程.png)
    - 对此槽重新分配的过程 [对槽slot进行重新分配的过程](/images/redis/对槽slot进行重新分配的过程.png)
- ASK错误：在重新分配槽期间,源节点向目标节点迁移一个槽的过程中，客户端可能会向服务器get数据，但这数据可能会被迁移到了新的节点，就会返回ASK错误，客户端根据ASK错误去新的节点get数据 [判断是否发送ASK的过程](/images/redis/判断是否发送ASK的过程.png
)
    - ASK错误具体实现：如果一个节点收到关于key的命令，如果键key所在的槽正好指派给这个节点，那么节点会尝试在自己的数据库查找键，如果找到了的话，节点就执行客户端发送的命令，如果没找到，节点会检查自己的clusterState.migrating_slots_to[i]，看key所在的槽i是否正在进行迁移，如果的确在进行迁移，那么会向客户端发送一个ASK错误，引导客户端到正确的节点去查找key。转到新节点查找前，会先发送ASKING命令（ASKING命令是一次性的），在发送查找命令，如果不发送，会返回MOVED错误。
    - MOVED和ASK错误区别：
        - MOVED：表示槽的负责权已从一个节点转移到了另一个节点，之后的命令可以直接发送到新的节点
        - ASK错误是一次性的，下次还将领了发送到原节点
* 复制和故障转移
    - 设置从节点：如果主节点下线，设置原主节点的属性和其他从节点的属性
    - 故障检测：每个节点会向其他节点发送PING，来检测其他节点是否下线、疑似下线（半数以上的直接点认为它疑似下线，就会被他的主节点标记为下线，bong广播到其他节点），同时各个节点间会胡发消息来交换各个节点的状态信息，记录在fail_reports属性中
    - 故障转移：主节点下线，会从他的从节点中选出新的主节点，新的主节点会接管旧主节点的所有槽，并广播一条PONG，高速球其他节点，自己是新的主节点
- 消息：
    - MEET消息
    - PING消息
    - PING消息
    - FAIL消息：广播某节点下线的消息
    - PUBLISH消息
