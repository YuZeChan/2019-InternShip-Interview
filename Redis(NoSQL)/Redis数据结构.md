

## 简单动态字符串：

​	C字符串：以空字符‘\0’结尾的字符数组

​	Redis没有直接使用C字符串，而是自己构建了一种抽象类型，称为简单动态字符串（Simple Dynamic String, SDS），用作默认字符串表示。

- Redis只有在 **字面量** （无需修改时）才使用C字符串，比如打印日志。
- 作为对象时，都是用SDS作为底层实现。



​	除了用来保存数据库的字符串之外，SDS还被用作缓冲区：在持久化部分的AOF缓冲区，以及客户端状态中的输入缓冲区，都是由SDS实现的。



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

​	链表提供了高效的节点重排能力，顺序性节点访问方式，并且增删灵活，便于调整链表长度。

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

​	Redis字典的底层实现是**哈希表**。一个哈希表里面有多个哈希表结点，而每个哈希表结点中保存了字典中的一个键值对。



##### 哈希表和哈希表结点：

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

ht属性是一个包含两个项的数组，每项都是一个dictht哈希表。

一般情况下只用到ht[0]，只有在重哈希（一般是扩张）才用到ht[1]。



***

#### 哈希算法：

```python
#使用预先设置的哈希函数计算哈希值
hash = dict->type->hashFunction(key);

#使用sizemask属性和哈希值，计算出索引
#ht[x] 中x可能取0或1
index = hash & dict->ht[x].sizemask;
#按位与，其实相当于求余。
```

**Redis默认使用的hashFunction是MurmurHash2算法。**



***

#### 解决键冲突：

Redis使用的是链地址法（对应的开放地址法）来解决键冲突。

数据插入时使用头插法O(1)



***

#### Rehash：

当不断修改哈希表时，键值对会逐渐增多或减少，为了让哈希表的 **负载因子（load factor）** 维持在一个合理范围内，要对哈希表大小进行相应扩展或收缩。（**均以2^n为粒度**）

- 扩展操作：第一个大于等于2*used的2^n（当然此时已经超过规定负载因子了 **load_factor = used / size** ），比如used为3， 那么size扩大后为 2 * 3 = 6 <= 2^3 = 8，所以扩展后size为8；
- 收缩操作：第一个大于等于2*used的2^n。



##### 扩展与收缩的条件：

- 服务器目前没有执行BGSAVE或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
- 服务器目前正在执行BGSAVE或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5
- 当哈希表的负载因子小于0.1时，会自动进行收缩操作

why？（ **总的来说就是减少不必要的内存写操作** ）

- 当上述两个命令执行时，Redis需要创建当前服务器进程的子进程；
- 因为大多数操作系统采用写时复制的方式来优化子进程效率；
- 所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，尽可能避免在子进程存在期间进行哈希表扩展操作。



***

#### 渐进式rehash：

采用分而治之的方式，将rehash键值对所需的计算工作，分摊到对字典的每个添加、删除、查找和更新操作上。（避免集中操作的庞大计算量）

![rehash1](/resources/rehash1.jpg)![rehash2](/resources/rehash2.jpg)![rehash4](/resources/rehash4.jpg)![rehash3](/resources/rehash3.jpg)

**因为渐进式的rehash操作，所以在此期间的删除、查找、更新等操作会在两个哈希表上执行，先找ht[0]再找ht[1]，以防错漏。**





***



## 跳跃表：

​	跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

- 平均查找效率O(logN)，媲美平衡二叉树，最坏O(N)
- 实现简单，常用来替代平衡树



​	**Redis只在两个地方用到了跳表：实现有序集合键；在集群节点中用作内部数据结构。**



***

#### 跳跃表的实现：

![跳跃表](/resources/跳跃表.png)

- 其中要注意的是level属性，表示层；
- 每一层都有前进指针和跨度，跨度表示前进指针所指向的结点和当前节点的距离；
- 跳表是排序的，首先按分值排序，分值相同按obj排序。（ **成员对象obj是唯一的** ）因为是排序的，也即可以二分查找！！！O(logN)



***

#### 跳表的节点：

```c
typedef struct zskiplistNode{
    
    //层，是结构体数组 
    struct zskiplistLevel{
        
        //前进指针
        struct zskiplistNode* forward;
        
        //跨度；用于计算排位
        unsigned int span;
        
    }level[];
    
    //后退指针
    struct zskiplistNode* backward;
    
    //分值
    double score;
    
    //成员对象
    robj* obj;
    
}zskiplistNode;
```

- 每次一个新的节点生成时，程序会根据幂次定律（power law，越大的数出现概率越小）随机生成一个介于1到32的值（ **层高最高为32层** ），作为level组数的大小。低层次的节点level数组可能大于高层的节点。
- 指向NULL的所有前进指针的跨度为0。
- 遍历时经过的所有跨度之和，为结点在跳表中的排位~~
- 后退指针每次只能后退至前一个结点。

![跳表的遍历](/resources/跳表的遍历.png)



***

#### 跳表：

​	同理，只有结点不行，需要一个zskiplist结构来持有这些结点。

```c
typedef struct zskiplist{
    
    //表头指针和表尾指针
    struct zskiplistNode *head, *tail;
    
    //表中结点的数量
    unsigned long length;
    
    //表中层数最大的节点的层数
    int level;
    
}zskiplist;
```

![带list结构的跳表](/resources/带list结构的跳表.png)



***



## 整数集合：

​	整数集合是集合键的底层实现之一，当有以下特征时，Redis用其实现集合：

- 一个集合只包含整数
- 且集合元素数量不多



***

#### 整数集合的实现：

​	因为是集合，所以保证不出现重复；保存整数的类型int16_t、int32_t或int64_t。

```c
typedef struct intset{
    
    //编码方式：int16_t、int32_t或int64_t
    uint32_t encoding;
    
    //集合包含的元素数量
    uint32_t length;
    
    // typedef signed char int8_t;
    // 保存元素的数组
    int8_t contents[];
    
}intset;
```



***

#### 升级：

​	当向集合中添加新的元素时，并且新元素的类型比当前集合中的所有元素类型都要长时，整数集合需要先进行升级：

1. 根据新元素类型，扩展content数组空间，并为新元素分配空间。
2. 原集合元素升级，并放在正确的数组位置上。
3. 添加新元素。
4. 更新属性编码和元素个数。

![整数集合升级](/resources/整数集合升级.png)

元素移动时，计算位置，先移动后面的元素，这样可以不占额外空间。



​	**升级之后，因为新元素最长，所以只有可能是比当前集合中所有元素都大，或者都小（负数情况），所以插入位置只需考虑数组尾部（length-1位置）和头部（0位置）。**



***

#### 升级的好处：

1. 提升灵活性：不在一个数据结构（content数组）中，存不同类型的值，避免类型错误。
2. 节约内存：尽力使用最小的空间，只有在必要时才升级。



***

#### 不支持降级；



***



## 压缩列表：

​	压缩列表是Redis为了 **节约内存** 而开发的，由一系列特殊编码的 **连续内存块** 组成的顺序（sequential）数据类型。一个压缩列表可包含多个项（entry）， **每个节点可以保存一个字节数组或一个整数值** 。

​	压缩列表（ziplist）是列表键和哈希键的底层实现之一。满足以下条件，列表建使用压缩列表作为底层实现：

- 只包含少量列表项
- 每个列表项要不就是小整数值，要么就是长度较短的字符串



***

#### 压缩列表的构成：

​	![压缩列表](/resources/压缩列表.png)



***

#### 压缩列表结点的构成：

- 字节数组：
  - 长度小于等于63（2^6-1）字节的字节数组
  - 长度小于等于16383（2^14-1）字节的字节数组
  - 长度小于等于4294967295（2^32-1）字节的字节数组
- 整数：
  - 4位长，0-12的无符号整数
  - 1字节长的有符号整数
  - 3字节长的有符号整数
  - int16_t类型的整数
  - int32_t类型的整数
  - int64_t类型的整数

![压缩列表结点](/resources/压缩列表结点.png)

##### previous_entry_length：

- 记录 **前一个结点的长度，类似于偏移量** ，通过当前ptr可以直接找到前一个结点的头。（方便从后向前遍历）
- 属性长度为1字节或5字节



##### encoding：

- 记录结点content属性保存的数据类型和长度（具体规则不记了）
- 字节数组00,01,10_ _ _ _ _ _ _ ...
- 整数编码1100000,11010000,11100000...

![压缩列表结点编码规则](/resources/压缩列表结点编码规则.png)



##### content:

- encoding = 类型 + 长度 （可以对照上表）
- 如00 001011 => “hello world”



***

#### 连锁更新：

- 由于previous_entry_length可用1或5个字节来记录前面结点的长度，当结点长度小于255时，比如254，可用1个字节存储。
- 当一个新的长结点插入头部时，之后节点的previous_entry_length则需用5个字节来储存。
- 然后需要将previous_entry_length属性扩展到5个字节，这个结点的长度也超过了255
- 程序需要不断地对压缩列表之后的节点也进行空间扩展，出现“ **连锁更新** ”。

![压缩列表连锁更新](/resources/压缩列表连锁更新.png)



**最坏更新复杂度为O(N^2)**

但这种情况的实际影响较低：

- 首先长度介于250-253字节的节点，而且连续，的情况并不常见。
- 其次即便连续更新，则只要需更新的节点数不多，就不会造成性能的大幅下降。

