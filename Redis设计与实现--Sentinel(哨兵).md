# Redis设计与实现--Sentinel哨兵（Redis设计与实现第十六章）

* Sentinel是redis的高可用性解决方案：由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个服务器，以及这些服务器属下的所有从服务器，在监视到有主服务器下线时，自动将主服务器下的某个从服务器升级为新的主服务器
* 启动并初始化Sentinel
    1. 初始化服务器：Sentinel本质上是一个运行在特殊模式下的Redis服务器。但他不载入RDB文件或者AOF文件
    2. 使用Sentinel专用代码：将部分普通redis服务器使用的代码替换为Sentinel的专用代码
    3. 初始化Sentinel状态

```C
    struct sentinelState {
        //当前纪元，用于实现故障转移
        uint64_t current_epoch;

        //保存了所有监视的主服务器，字典的键是主服务器的名字，值时一个指向sentinelRedisInstance结构的指针
        dict *masters;

        //是否进入了TILT模式
        int tilt;

        //目前正在执行的脚本的数量
        int running_scripts;

        //进入TILT模式的时间
        mstime_t tilt_start_time;

        //最后一次执行事件处理器的时间
        mstime_t previous_time;

        //一个FIFO队列，包含了所有需要执行的用户脚本
        list *scripts_queue;

    } sentinel;
```
    4. 初始化Sentinel状态的masters属性
    5. 创建连向主服务器的两个网络连接
        - 一个是命令连接，用于向主服务器发送命令，接受命令回复
        - 一个是订阅连接，用于订阅主服务器的_sentinel_:hello频道
- 获取主服务器信息： Sentinel默认会以每10秒一次的频率，通过命令连接向被监视的主服务器发送INFO，并通过分析INFO命令的回复来获取主服务器的当前信息。信息有两方面
    - 关于主服务器的信息，包括run_id记录的服务器运行id，以及role属性记录的服务器角色
    - 服务器属下所有的从服务器记录，每个从服务器都由一个“slave”字符串开头的行记录，记录了从服务器的IP，port，有了这些信息，Sentinel无需用户提供从服务器的地址信息，就可以自动发现从服务器
    [主服务器和他的从服务器](/images/redis/主服务器和他的从服务器.png)
- 获取从服务器信息：Sentinel发现主服务器有新的从服务器出现时，除了实例化数据结构，还会创建连接从服务器的命令连接和订阅连接，之后，Sentinel会以10秒一次的通信频率想从服务器发送INFO命令，以获取相关信息：从服务运行ID，run_id;role;主服务器的ip，port；从服务器偏移量...
- 向主服务器和从服务器发送消息：Sentinel会以两秒一次的频率，向主服务器和从服务器的_sentinel_:hello频道发送命令，命令里包含了Sentinel的ip、port、运行ID，当前配置纪元，主服务器的名称、主服务器ip、端口、当前配置纪元
- 接收主服务器和从服务器的频道信息:Sentinel向_sentinel:hello频道发送的信息会被其他监听这个服务器的Sentinel的服务器收到，并更新信息
    - 更新sentinel信息：当一个sentinel收到其他Sentinel发来的信息时，目标Sentinel会信息中提取出源sentinel的ip、port、运行id、配置纪元。以及源sentinel正在监视的主服务器名称、ip、port、配置纪元。setnienl会根据提取到的sentinel参数查看自己是否保存了该sentinel的信息，如果有就更新，没有就添加
    - 创建连向其他sentinel的命令连接：当sentinel通过频道信息发现一个新的sentinel时，他不仅会在sentinel中创建实例，还会建立一个连向新sentinel的连接，最终多个sentinel会相互连接、请求信息交换
- 检测主观下线状态：sentinel会以每秒一次的频率向所有创建了命令连接的实例发送PING，并通过PING回复来判断实例是否在线。如果一个实例在down_after_millseconds内，连续返回无效回复，那么Sentinel会修改这个示例所对应实例结构中的flags属性，以表示这个实例进入主观下线状态
- 检查客观下线状态：当sentinel判断一个主服务器下线后，为了确认这个主服务器是真的的下线，他会向同样监视这一主服务器的其他sentinel进行询问，看他们是否也认为主服务器已经显现了。
    - 发送is_master_down_by_addr
    - 接收is_master_down_by_addr，返回回复：
        - <down_state>
        - <leader_runid>
        - <leader_epoch>
    - 接收is_master_down_by_addr命令回复：Sentinel收到回复后，会统计其他Sentinel同意主服务器下线的数量，当数量达到下线所需的数量时，会将主服务器实例结构中的SRI_O_DOWN标识打开，表示主服务器已进入客观下线状态
- 选举领头Sentinel
    - 当一个主服务器下线时，监视这个服务器的Sentinel会选举出一个领头Sentinel，由他进行主服务器的故障转移
    - 选举时源Sentinel根据is_master_down_by_addr回复中的leader_epoch和leader_runid来判断自己是否被选中，如果有超过半数的Sentinel选举自己，那么自己就是领头Sentinel，如果没有超过半数Sentinel，则一段时间后继续选举，直到选出来为止
- 故障转移
    - 在从服务器中选举出新的主服务器，选举出后，Sentinel发送slaveof no one，当从服务器的回复中的role油slave变为master时，说明从服务器顺利升级了
    - 修改从服务器的复制目标，向其他从服务器发送 slave of <Ip> <port>
    - 将旧的主服务器（重新上线时，Sentinel向server发送SALVEOF 命令）变为从服务器
