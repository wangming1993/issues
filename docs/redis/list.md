# 链表 【Redis设计与实现】

## 内部实现

```c
/* Node, List, and Iterator are the only data structures used currently. */

typedef struct listNode {
    // 指向前置节点的指针
    struct listNode *prev;
    // 指向后置节点的指针
    struct listNode *next;
    // 数据， void * 表示可以存储任意类型的指针
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    // 节点值复制函数 
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

<!-- more -->

## 重点

- 链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。
- 每个链表节点由一个listNode结构来表示，每个节点都有一个指向前置节点和后置节点的指针，所以Redis的链表实现是双端链表。
- 每个链表使用一个list结构来表示，这个结构带有表头节点指针、表尾节点指针，以及链表长度等信息。
- 因为链表表头节点的前置节点和表尾节点的后置节点都指向NULL，所以Redis的链表实现是无环链表。
- 通过为链表设置不同的类型特定函数，Redis的链表可以用于保存各种不同类型的值。
