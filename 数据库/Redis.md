# Redis

Redis是NoSQL（泛指非关系型的数据库）

最大连接数：info clients查看

## 常用数据结构

数组string

列表list

哈希hash

集合set

有序集合zset

## 为什么快

①内存数据库

②单线程模式

③非阻塞IO

## 持久化

RDB（默认）：  当前进程数据生成快照保存到硬盘的过程。

fork一个进程，遍历hashtable，利用copy on write，把整个db dump保存下来。
save, shutdown, slave 命令会触发这个操作。粒度比较大，如果save, shutdown, slave 之前crash了，则中间的操作没办法恢复

AOF（主流）：  以独立日志的方式记录每次写命令，重启时再重新执行AOF文件 

把写操作指令，持续的写到一个类似日志文件里。（类似于从postgresql等数据库导出sql一样，只记录写操作）粒度较小，crash之后，只有crash之前没有来得及做日志的操作没办法恢复。 

 https://www.cnblogs.com/yhq-qhh/p/10140586.html 

## 类型对应底层结构

##### string（string是五种类型中唯一一种会被其他四种对象嵌套的类型）

int：存储long类型的整数
embstr：字符串长度小于等于39字节，或者double浮点数
raw： 字符串长度大于39字节，或者long和double无法存储的数字

##### list

ziplist：元素数量小于512，且每个字符串长度小于64

linkedlist：不满足上述条件
**在新版的redis中，实现方式是quicklist** 

##### hash

ziplist：K-V数量小于512，且K和V的字符串长度小于64
hashtable：不满足上述条件

##### set

intset：集合中元素数量小于512，且所有元素都是整数
hashtable：不满足上述条件

##### zset

ziplist：有序集合元素数量小于128，且所有元素的长度小于64
skiplist & hashtable：不满足上述条件

## 底层结构解析

 https://www.cnblogs.com/javazhiyin/p/11063944.html 

#### 字符串

sds动态字符串

```
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串
     char buf[];
}
```

优点：

开发者不用担心字符串变更造成的内存溢出问题。

常数时间复杂度获取字符串长度`len字段`。

空间预分配free字段，会默认留够一定的空间防止多次重分配内存。

#### 链表

```
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;

typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode
```

特点：

可以直接获得头、尾节点。

常数时间复杂度得到链表长度。

是双向链表。

#### 字典

```
typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used;
}dictht//哈希表

typedef struct dictEntry{
     //键
     void *key;
     //值
     union{//可以是指针，uint64_torint64_t整数
          void *val;
          uint64_tu64;
          int64_ts64;
     }v;
 
     //指向下一个哈希表节点，形成链表
     struct dictEntry *next;
}dictEntry//哈希点
```

哈希冲突解决：**链地址法（相同地址kv链地址链接）**

#### 跳跃表

 https://segmentfault.com/a/1190000019690783 

 https://www.jianshu.com/p/61f8cad04177 

一种有序数据结构，它通过在每个节点中维持多个指向其它节点的指针，从而达到快速访问节点的目的 

（可以当成是一种二分查找的结构保存数据，时间复杂度log2n）

 ![img](https://segmentfault.com/img/remote/1460000019690786?w=1720&h=1056) 

特点：

①层结构组成

②每一层都是有序链表，排序由head到后面的nil，至少含两个节点

③底层链表包含了所有元素

④一个元素出现在某一层链表，该层之下的链表也全都出现

⑤每个节点含两个指针，一个指向同一层下个，另一个指向下一层同一个链表

```
typedef struct zskiplist{
     //表头节点和表尾节点
     structz skiplistNode *header, *tail;
     //表中节点的数量
     unsigned long length;
     //表中层数最大的节点的层数
     int level;
}zskiplist;

typedef struct zskiplistNode {
     //层
     struct zskiplistLevel{
           //前进指针
           struct zskiplistNode *forward;
           //跨度
           unsigned int span;
     }level[];
 
     //后退指针
     struct zskiplistNode *backward;
     //分值
     double score;
     //成员对象
     robj *obj;
} zskiplistNode
```

搜索：从最高层的链表节点开始，如果比当前节点要大和比当前层的下一个节点要小，那么则往下找，也就是和当前层的下一层的节点的下一个节点进行比较，以此类推，一直找到最底层的最后一个节点，如果找到则返回，反之则返回空。

插入：首先确定插入的层数，有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层。

删除：在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则删除这一层。

#### 整数集合

```
typedef struct intset{
     //编码方式
     uint32_t encoding;
     //集合包含的元素数量
     uint32_t length;
     //保存元素的数组
     int8_t contents[];
}intset;
```

按照从小到大的顺序排列，并且不包含任何重复项

#### 压缩列表

**压缩列表的原理：压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。** 

 ![img](https://upload-images.jianshu.io/upload_images/9463862-47c3cac032667b12.png?imageMogr2/auto-orient/strip|imageView2/2/w/799/format/webp) 

| 名称    |       | 作用                                                         |
| ------- | ----- | ------------------------------------------------------------ |
| zlbytes | 4字节 | 记录整个压缩列表占用的内存字节数，在对压缩列表进行内存重分配或计算zlend的位置时使用 |
| zltail  | 4字节 | 记录压缩列表尾节点距离压缩列表的起始地址有多少字节，通过这个偏移量，可以直接确定尾节点的位置 |
| zllen   | 2字节 | 记录压缩列表包含的节点数量                                   |
| entryX  | 不定  | 表示各种节点,数量和长度不一定。                              |
| zlend   | 1字节 | 用于标记压缩列表的末端                                       |

 节点组成（entryX）： ![img](https://upload-images.jianshu.io/upload_images/9463862-d83a4dd1504bd27d.png?imageMogr2/auto-orient/strip|imageView2/2/w/406/format/webp)  

| 名称                  | 长度                                                         | 作用               |
| --------------------- | ------------------------------------------------------------ | ------------------ |
| previous_entry_length | 1字节（小于254字节）或者5字节                                | 记录前一个节点长度 |
| encoding              | 1字节、2字节或者5字节（最高位是字节数组编码，长度由后面记录） | 保存数据类型及长度 |
| content               | 一个字节数组或者整数                                         | 保存节点的值       |

#### 快速表

由ziplist组成的双向链表

 ![img](https://upload-images.jianshu.io/upload_images/9463862-565a85166eb3f558.png?imageMogr2/auto-orient/strip|imageView2/2/w/1183/format/webp) 

```
//真正表示quicklist的数据结构
typedef struct quicklist {
    // 指向头节点的指针（最左边）
    quicklistNode *head;
    // 指向尾节点的指针(最右边)
    quicklistNode *tail;
    // 所有ziplist数据项的个数总和
    unsigned long count;       
    // quicklist节点的个数
    unsigned int len;           
    // ziplist大小设置
    int fill : 16;              
    // 节点压缩深度设置
    unsigned int compress : 16;
} quicklist;
typedef struct quicklistNode {
    //前驱节点
    struct quicklistNode *prev;
    //后继节点
    struct quicklistNode *next;
    //数据指针,如果当前节点的数据没有压缩,它就指向一个ziplist结构,否则指向quicklistLZF结构
    unsigned char *zl;
    // 表示zl指向的ziplist的总大小,如果ziplist被压缩了,它的值仍然是压缩前的大小
    unsigned int sz;            
    // 表示ziplist里面包含的数据项个数,这个字段16bit
    unsigned int count : 16;    
    // 表示ziplist是否压缩了,1代表没有压缩  2代表使用LZF压缩
    unsigned int encoding : 2;  
    // 预留字段,固定值2,表示使用ziplist作为数据容器
    unsigned int container : 2; 
    // 此节点之前是否已经压缩过
    unsigned int recompress : 1; 
    // 测试用的,暂时用不上
    unsigned int attempted_compress : 1; 
    // 扩展字段,暂时无用
    unsigned int extra : 10;
} quicklistNode;

// 此结构表示一个被压缩过的ziplist
typedef struct quicklistLZF {
    // 压缩后的ziplist大小
    unsigned int sz;
    // 存放压缩后的ziplist字节数组
    char compressed[];
} quicklistLZF;
```

## 扩容机制

渐进式单线程扩容

https://blog.csdn.net/sanyuesan0000/article/details/89920746

为了让哈希表的负载因子（load factor）维持在一个合理的范围内，会使用rehash（重新散列）操作对哈希表进行相应的扩展或收缩。

2.3.1，哈希表被扩展的条件：

1）Redis服务器目前没有在执行BGSAVE命令或BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。

2）Redis服务器目前在执行BGSAVE命令或BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。

负载因子=哈希表已保存节点数量 / 哈希表大小   load_factor = ht[0].used / ht[0].size

当哈希表的负载因子小于0.1时，对哈希表执行收缩操作。

2.3.2，rehash的操作步骤

1）为字典的ht[1]哈希表分配空间。

如果执行的是扩展操作，那么ht[1] 的大小为第一个大于等于ht[0] .used*2的2的n次幂

如果执行的是收缩操作，那么ht[1] 的大小为第一个大于等于ht[0].used 的2的n次幂

因此这里我们为ht[1] 分配 空间为8

2)将ht[0]中的数据转移到ht[1]中，在转移的过程中，重新计算键的哈希值和索引值，然后将键值对放置到ht[1]的指定位置。

渐近式rehash

在进行拓展或者压缩的时候，可以直接将所有的键值对rehash 到ht[1]中，这是因为数据量比较小。在实际开发过程中，这个rehash 操作并不是一次性、集中式完成的，而是分多次、渐进式地完成的。

渐进式rehash 的详细步骤：

1、为ht[1] 分配空间，让字典同时持有ht[0]和ht[1]两个哈希表

2、在几点钟维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash 开始

3、在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一

4、当ht[0]中所有数据转移到ht[1]中时，将rehashidx 设置成-1，表示rehash 结束

采用渐进式rehash 的好处在于它采取分而治之的方式，避免了集中式rehash 带来的庞大计算量。

##### 高并发处理

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。 