# 字典【Redis 设计与实现】

## 内部实现

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```

结构示意图：

![redis-dict](http://otuvs4s36.bkt.clouddn.com/redis-dict.png)

## 原理讲解

### 字段

#### ht 字段

`ht` 是一个两个项的数组，其中 `ht[0]` 用于存放哈希表，`ht[1]` 在哈希重排时使用

#### rehashidx 字段

当 `rehashidx == -1` 时，表示当前不是在 `rehash`, 用于表示在进行 `rehash` 操作时，进行到了哪一步。

### 如何操作数据

假设现在需要添加一个数据 `<k, v>`, 先需要计算哈希值:
```
hash = dict->type->hashFunction(k);
```
然后根据 `sizemask` 求出索引值:
```
 index = hash & dict->ht[x]->sizemask;   
 // x 表示实际存放哈希表的索引， 一般为 0 ，当在进行 rehash 时为 1
```
这样就可以将 `<k, v>` 存储在 `dict->ht[x]->table[index]` 中， 如果 `table[index]` 中已经有数据， 则新添加的数据放在链表表头. (这是因为 `table[index]` 中的链表时一个单链表，没有指向链表尾部的指针，添加到表头更快)

### rehash 过程

当哈希表保存的数据太多或太少时，需要对哈希表进行相应的扩容或者收缩。
如果进行扩容操作，那么 `ht[1]` 的大小为第一个不小于 `ht[0].used * 2` 的 2^n (n 为正整数)， 如: `used = 5`, `ht[0].used * 2 = 10 < 2^4 = 16`, 所以 `ht[1]` 的大小为：16 · 
然后就可以将 `ht[0]` 的数据哈希到 `ht[1]` 中， 当 `ht[0]` 中的数据全部哈希到 `ht[1]` 后， 释放 `ht[0]`,  将 `ht[1]` 变 `ht[0]`

#### 扩展的触发条件

1. 在没有执行 `BGSAVE` 或 `BGREWRITEAOF`时，哈希表的负载因子 `>= 1`
2. 在执行 `BGSAVE` 或 `BGREWRITEAOF`时，哈希表的负载因子 `>= 5`

负载因子的计算：
```
load_factor = ht[0].used / ht[0].size
```

> 在进行 `BGSAVE` 或 `BGREWRITEAOF`时，提高负载因子是为了避免扩容，避免不必要的内存写入

#### 收缩的触发条件

负载因子 `< 0.1`

### 渐进式哈希

在扩容或者收缩时，需要将 `ht[0]` 全部哈希到 `ht[1]`, 如果一次性哈希， 当数据足够多时，会存在效率问题。因此 `redis` 通过逐步哈希的方式，其具体过程如下：

1. 为 `ht[1]` 分配空间
2. 设置 `rehashidx = 0`
3. 新添加的数据不再加入到 `ht[0]`, 而是加入到 `ht[1]`中，保证 `ht[0]`不会增加数据
4. 将 `ht[0]->table[rehashidx]` 的数据全部哈希到 `ht[1]` 中
5. `rehashidx++`
6. 随着不断的执行，`ht[0]` 的数据哈希到了 `ht[1]`, 这是可以 设置 `rehashidx = 1`， 表明 `rehash` 结束，释放 `ht[0]`,  `ht[1]` 设置为 `ht[0]`

在 `rehash` 的过程中， 删除，查找，更新会同时在 `ht[0]`, `ht[1]` 中进行， 但是添加只会在 `ht[1]` 中。
