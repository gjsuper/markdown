# 极客时间--MySQL实战45讲--第9讲：普通索引和唯一索引

* 查询过程：
    - 对于普通索引，查找到满足条件的第一个记录后，还要继续查找下一个记录，直到碰到第一个不满足条件的记录
    - 对于唯一索引，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止检索。

    这两种不同带来的性能差距有多少呢？微乎其微。因为InnoDB的数据是按数据也单位来读写的。当读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读进内存。在InnoDB中，每个数据也的大小默认为16K。所以找到相应的记录时，他所在的数据也都在内存里，那么对普通索引来说，最多做一次"查找和判断下一条记录"的操作，就只需要一次指针寻找和一次计算。
    如果这条记录恰好是数据页的最后一条，那么读取下一条数据，必须读取下一页，会稍微复杂一点。然而对于整形字段，每一页可以放近千个数据，所以这种可能性很低。
* 更新过程

    当需要更新一个数据时，如果数据在内存中就直接更新，如果不在，InnoDB会将这些更新操作缓存在change buff中，这样就不需要从磁盘读数据了，下次访问这个数据的时候，将数据读入内存，然后执行change buffer中与这个页相关的操作。这样就可以保证逻辑正确性。
    changgebuffer会持久化到磁盘。
    change buffer会在每次访问数据时会触发merge（将change buffer中的操作应用到元原数据页，得到更新结果的过程称为merge），系统有后台线程定期merge，数据库正常关闭的过程中也会执行merge

    * 什么时候可以到使用change buffer，如果要更新的记录在内存中
        - 唯一索引：需要判断表中是否存在这个值，因此需要将数据读入内存，现在已经在内存了，就不用读了，直接找到相应的位置，判断是到无冲突，插入数据
        - 普通索引：找到相应位置，插入值
    * 如果记录不在内存中：
        - 唯一索引：需要将数据页读入内存，判断到无冲突，插入值
        - 普通索引：将更新记录在change buffer中，语句结束
    - 所以change buffer只能用于普通索引，起到加速的作用。因为merge的时候才是真正更新数据的时候，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以一个数据页做merge之前，change buffer记录的变更越多，收益就越大。因此对写多读少的业务，change buffer 性能很好。对于这种更新后立马查询的操作，会先将更新记录在change buffer中，但由于马上访问这个数据页，会立即出发merge，这种随机访问的次数不会减少，反而增加了change buffer的维护代价。
    - 将数据从磁盘读入内存涉及随机IO访问，这时数据库里访问成本最高的操作之一，changgebuffer减少了随机访问磁盘，对性能的提升有很大好处。
- merge的流程
    - 从磁盘读入数据页到内存（老版本的数据页）
    - 从 change buffer 里找出这个数据页的 change buffer 记录 (可能有多个），依次应用，得到新版数据页。
    - 写 redo log。这个 redo log 包含了数据的变更和 change buffer 的变更。

    到这里 merge 过程就结束了。这时候，数据页和内存中change buffer对于的磁盘位置都还没有修改，属于脏页，之后各自刷回自己的物理数据，这就是另外一个过程了。
* redo log主要减少了随机写磁盘的操作，changebuffer减少了随机读读磁盘的操作    
