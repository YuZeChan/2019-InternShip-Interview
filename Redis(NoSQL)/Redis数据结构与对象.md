## 简单动态字符串：

C字符串：以空字符‘\0’结尾的字符数组

Redis没有直接使用C字符串，而是自己构建了一种抽象类型，称为简单动态字符串（Simple Dynamic String, SDS），用作默认字符串表示。

- Redis只有在 **字面量** （无需修改时）才使用C字符串，比如打印日志。
- 作为对象时，都是用SDS作为底层实现。

除了用来保存数据库的字符串之外，SDS还被用作缓冲区：在持久化部分的AOF缓冲区，以及客户端状态中的输入缓冲区，都是由SDS实现的。



***

#### SDS的定义：

```c
struct sdshdr{
    
    //记录buf数组中已使用的字节数量
    //等于SDS所保存的字符串长度
    int len;
    
    //记录buf数组中未使用字节的数量
    int free;
    
    //字节数组，用于保存字符串
    char buf[];
}
```

![SDS示例](/resources/SDS示例.png)

遵循C字符串以'\0'即为的惯例，且空字符不计入len属性里面。

**<u>好处是SDS可以重用C字符串的一些库函数。</u>**



***

#### SDS和C字符串之间的关系：

1. **O(1)复杂度获取字符串长度**
2. **API安全，在修改操作时会检查len和free属性，独居缓冲区溢出**
   - 如sdscat，会先检查空间是否满足要求，不满足则会先扩展，再修改。
   - 扩展后满足free == len(更新过)
3. **减少修改字符串是带来的内存重分配次数**
   - 每次分配内存需要进行系统调用，需要进行用户空间向内核空间的切换
     - **空间预分配**（这样使连续增长字符串，由内存必定分配N次，变成内存做多分配N次）
       - len：free = 1:1 （len < 1MB）
       - free = 1MB (len >= 1MB)
     - **惰性空间释放**（字符串收缩操作时，不立即回收多余空间，而是保存在free属性中，记录下来）
4. **二进制安全**
   - C字符串读到'\0'字符串就结束了，而sds则不因'\0'来过滤数据（重写了API）
   - 这也是将SDS中的buf称为 **字节数组** 的原因， **Redis并非用它来保存字符，而是保存一系列二进制数** 。
5. **兼容部分C字符串函数<string.h>**（避免不必要的代码重复）



***



## 链表：

链表提供了高效的节点重排能力，顺序性节点访问方式，并且增删灵活，便于调整链表长度。

- 列表键的底层实现之一。
- 发布与订阅、慢查询、监视器等功能的底层实现用到了链表。
- Redis服务器使用链表来保存多个客户端的状态信息。
- 客户端的输出缓冲区的底层实现。



***

#### 链表和链表结点的实现：

```c
typedef struct listNode{
    
    //前置结点
    struct listNode* prev;
    
    //后置结点
    struct listNode* next;
    
    //节点的值，使用void*来多态保存值~
    void* value;
    
}listNode;

typedef struct list{
    //C中无类的概念，用结构体代替
    //表头节点
    listNode* head;
    
    //表尾节点
    listNode* tail;
    
    //链表包含的节点数量
    unsigned long len;
    
    //结点值复制函数
    void* (*dup)(void* ptr);
    
    //结点值释放函数
    void* (*free)(void* ptr);
    
    //结点值对比函数
    void* (*match)(void* ptr, void* key);
    
}list;
```

![list和listNode组成的链表](/resources/list和listNode组成的链表.png)

Redis链表的特性：

- **双端**
- **无环**
- **带表头指针和表尾指针**
- **带链表长度计数器**
- **多态（void *指针保存value，以及三个void *方法）**



***



## 字典：

字典，又称符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对的抽象数据类型。

- 字典中每个键都独一无二。
- Redis数据库就是用字典作为底层实现的。
- 字典是哈希表的底层实现之一。



***

#### 字典的实现：

Redis字典的底层实现是**哈希表**。一个哈希表里面有多个哈希表结点，而每个哈希表结点中保存了字典中的一个键值对。

哈希表：

```c
typedef struct dictht{
    
    //哈希表数组，是一个动态的指针数组。
    dictEntry **table;
    
    //哈希表大小
    unsigned long size;
    
    //哈希表大小掩码，用于计算索引，总是等于size-1
    unsigned long sizemask;
    
    //该哈希表已有结点的数量
    unsigned long used;
    
}dictht;

typedef struct dictEntry{
    
    //键
    void* key;
    
    //值，有三种类型，全是八个字节长度
    union{
        void* val;
        uint64_t u64;
        int64_t s64;
    }v;
    
    //指向下个哈希表结点，形成链表
    struct dictEntry* next;
    
}dictEnctry;
```

![哈希表](/resources/哈希表.png)

拉链法~~~~



##### 字典：

```c
typedef struct dict{
	
    //类型特定函数
    dictType *type;
    
    //私有数据，为创建多态字典而设置的， 保存了特定函数的可选参数
    void* privdata;
    
    //哈希表
    dictht ht[2];
    
    //rehash索引，当rehash不在进行时，值为-1
    int rehashidx;
    
}dict;

//一簇用于操作特定类型键值对的函数
typedef struct dictType{
    
    //计算哈希值的函数
    unsigned int (*hashFunction)(const void* key);
    
    //复制键的函数
    void* (*keyDup)(void* privdata, const void* key);
    
    //复制值的函数
    void* (*valDup)(void* privdata, const void* obj);
    
    //对比键的函数
    void* (*keyCompare)(void* privdata, const void* key1, const void* key2);
    
    //销毁键的函数
    void* (*keyDestructor)(void* privdata, void* key);
    
    //销毁值的函数
    void* (*valDestructor)(void* privdata, void* obj);
    
}dictType;
```

![普通状态下的字典](/resources/普通状态下的字典.png)

