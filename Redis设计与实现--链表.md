# Redis设计与实现--链表

* 由于C语言没有链表这种结构，redis构建了自己的链表实现，链表用在列表键、发布订阅、慢查询、监视器等功能上。
*  链表数据结构：

```c
typedef struct listNode {
    //前置节点
    struct listNode *prev;

    //后置节点
    struct listNode *next;

    //节点的值
    void *value;
}

typedef struct list {
    //表头节点
    listNode *node;

    //表尾节点
    listNode *tail;

    //链表锁包含的节点数量
    unsigned long len;

    //节点值复制函数
    void *(*dup)(void *ptr);

    //节点值释放函数
    void (*free)(void *ptr)

    //节点值对比函数
    int (*match)(void *ptr, void *key);
}
```
