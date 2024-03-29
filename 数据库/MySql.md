# MySql 

存储引擎：Innodb和myisam（都是B+树）

| Innodb                       | myisam                           |
| ---------------------------- | -------------------------------- |
| 聚簇索引（叶子结点就是数据） | 非聚簇索引（叶子结点是数据地址） |
| 支持事务                     | 不支持事务                       |
| 支持外键                     | 不支持外键                       |
| 支持行级锁                   | 支持表级锁                       |
| 支持MVCC                     | 不支持MVCC                       |

事务：一条SQL语句，一组SQL语句或整个程序。是恢复和[并发控制](https://baike.baidu.com/item/并发控制)的基本单位。

事务的ACID属性：原子性，一致性，隔离性，持续性

原子：每个事务不可分割，要么做，要么不做

一致性：事务是使数据库从一个一致性状态变到另一个一致性

持续性：事务一旦提交，对数据库修改是永久性的

隔离性：事务之间不能互相干扰 

## 四个隔离级别

未提交读：RU（脏读+不可重复读+幻读），数据有写锁，无读锁

已提交读：RC（不可重复读+幻读），每个数据加入排他锁和共享锁，不能共存，但可有多个共享锁，并读完就释放

可重复读：RR，MySql默认级别（幻读，MVCC可以解决），共享锁修改，事务准备提交才释放

可串行化：S（无），事务只能一件件完成

附：

脏读：A事务运行一个查询，B事务在查询前修改，查询后回滚，B事务完成，此时数据错误

不可重复读：A事务运行两次查询，B事务在查询中修改，A第一次查询错误

幻读：A事务修改某行后查询，B事务中途插入同一数据，A事务看起来像没有修改

## explain优化

信息很多，主要看type

system(一行)>const(一次索引)>eq_ref(唯一索引扫描)>ref(非唯一索引扫描)>range(特定行)>index(索引树)>all(全表)

执行顺序：
from->on->join->where->group by->having->select->distinct->order by->limit
简记：先构造虚拟表，然后select，再对表处理

## B+树性质与使用原因

 http://www.sohu.com/a/280609547_818692 

#### B+树性质

m阶B+树

①每个节点最多包含m个子节点

②如果根节点包含子节点，则至少包含 2 个子节点；除根节点外，每个非叶节点至少包含 m/2 个子节点

③拥有 k 个子节点的非叶节点将包含 k - 1 条记录

④所有叶节点都在同一层中

**⑤只有叶子节点存储真实的数据，非叶节点只存储键**

**⑥叶节点之间通过双向链表链接 **

#### 为什么B+树不是B树或者红黑树或者哈希

红黑树：一种平衡二叉树，但是一个节点数据只存储一个值，相对树高比较高，搜索速度比B+树差

B树：多路平衡查找树，跟B+树相比没有⑤⑥点，但是在多条数据选择上需要中序遍历返回，没有B+树高效

哈希：速度为O(1)，比B+树的O(logn)快，只选一个数据哈希快，但是在多个数据情况下，B+树索引有序且叶子节点有链表相连，查询效率比哈希高

B+树：索引保存在磁盘上，数据量大可能一次无法装入内存，B+树设计可以分批加载，同时树高比较低，多数据查询时叶子节点有双向链表，不用返回遍历，查找效率高。

## MySql锁

 https://www.nowcoder.com/discuss/151430 

大类型：行锁、表锁、行锁

行锁：共享锁S、排他锁X

表锁：意向共享锁IS、意向排他锁IX

#### 兼容情况

如果一个事务请求的锁模式与当前锁兼容，InnoDB就请求的锁授予该事务;
反之，如果两者两者不兼容，该事务就要等待锁释放

| 当前锁/是否兼容/请求锁 | X    | IX   | S    | IS   |
| ---------------------- | ---- | ---- | ---- | ---- |
| X                      | 冲突 | 冲突 | 冲突 | 冲突 |
| IX                     | 冲突 | 兼容 | 冲突 | 兼容 |
| S                      | 冲突 | 冲突 | 兼容 | 兼容 |
| IS                     | 冲突 | 兼容 | 兼容 | 兼容 |

简单理解记录：

共享锁一般用于事务读加锁，其他事务可以再加S锁，但是不能再加X锁，保证可以读，但不能修改

排他锁一般用于事务写加锁，其他事务不能再加锁，保证修改时候没有其他事务读或改

意向锁都是表锁，通常用于在加表行对应锁之前获取表的意向锁，避免全表扫描锁而是只用获取对应的意向锁即可，意向锁之间互相兼容，意向共享锁对共享锁兼容。

意向锁是InnoDB自动加的，不需用户干预
对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及及数据集加排他锁（Ｘ）
对于普通SELECT语句，InnoDB不会加任何锁

共享锁语句主要用在需要数据依存关系时确认某行记录是否存在
并确保没有人对这个记录UPDATE或DELETE
但如果当前事务也需要对该记录进行更新，则很有可能造成死锁
对于锁定行记录后需要进行更新操作的应用，应该使用排他锁语句

#### 行锁

通过给索引上的索引项加锁来实现

行锁基于索引实现，如果不通过索引访问数据，InnoDB会使用表锁

##### 间隙锁（Next-Key锁）

 当我们用范围条件而不是相等条件检索数据,并请求共享或排他锁时,InnoDB会给符合条件的已有数据的索引项加锁；此时对于键值在条件范围内但并不存在的记录，叫做“间隙(GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁（Next-Key锁）

##### 目的

①防止幻读，以满足相关隔离级别的要求

对于上例，若不使用间隙锁,如果其他事务插入 empid 大于 100 的任何记录
那么本事务如果再次执行上述语句，就会发生幻读

②满足其恢复和复制的需要

在使用范围条件检索并锁定记录时；
InnoDB 这种加锁机制会阻塞符合条件范围内键值的并发插入，这往往会造成严重的锁等待；
因此,在实际开发中，尤其是并发插入较多的应用；
我们要尽量优化业务逻辑，尽量使用**相等条件来访问更新数据**,避免使用范围条件

①普通索引

②唯一索引

③主键索引

④联合索引

⑤单列索引

## 面试问题

##### 知道集群式数据库吗？了解相关知识吗？说一下主从复制的原理？回滚情况

主服务器开启binlog日志，所有数据的改变都会记录到binlog中，一般是基于行的记录，从服务器每隔一段时间就会对这个日志进行探测查看是否改变，如果是开启一个IO线程请求读取日志，主服务器则开启dump线程发送日志，并将日志保存到从服务器的relaylog中，从服务器最后开启一个SQL线程读取relaylog日志并解析成SQL语句执行；

回滚的话一般是不会不会写到binlog中的，但是会使日志里的id+1

##### 哈希索引用过吗？有什么使用场景和限制？

https://blog.csdn.net/sunjin9418/article/details/80334142

HASH索引只有精确匹配索引所有列的查询才有效。

因为索引自身只需要存储对应的哈希值，所以索引的结构十分紧凑，这也让哈希索引查找的速度非常快，然而，哈希索引也有限制，如下：

哈希索引只包含哈希值和行指针，而不存储字段值，所以不能使用索引中的值来避免读取行（即不能使用哈希索引来做覆盖索引扫描），不过，访问内存中的行的速度很快（因为memory引擎的数据都保存在内存里），所以大部分情况下这一点对性能的影响并不明显。
哈希索引数据并不是按照索引列的值顺序存储的，所以也就无法用于排序
哈希索引也不支持部分索引列匹配查找，因为哈希索引始终是使用索引的全部列值内容来计算哈希值的。如：数据列（a,b）上建立哈希索引，如果只查询数据列a，则无法使用该索引。
哈希索引只支持等值比较查询，如：=,in(),<=>(注意，<>和<=>是不同的操作)，不支持任何范围查询（必须给定具体的where条件值来计算hash值，所以不支持范围查询）。
访问哈希索引的数据非常快，除非有很多哈希冲突，当出现哈希冲突的时候，存储引擎必须遍历链表中所有的行指针，逐行进行比较，直到找到所有符合条件的行。
如果哈希冲突很多的话，一些索引维护操作的代价也很高，如：如果在某个选择性很低的列上建立哈希索引（即很多重复值的列），那么当从表中删除一行时，存储引擎需要遍历对应哈希值的链表中的每一行，找到并删除对应的引用，冲突越多，代价越大。

#####  MySQL用过吗，讲一下索引类型，innodb中索引的数据结构，为什么用b+树，sql语句执行很慢怎么办（首先用explain分析语句，查看执行计划），然后给了个场景： 

```
有一个查询, select * from table where a = ``1` `and b = ``2``，问这个会用到索引吗，如果会的话会用到a还是b，还是说a和b都会用到
```

and用第一个索引，or的话才是两个都用到

##### 为什么用叶子节点保存数据

因为可以通过索引放到内存中，最后通过索引去硬盘中去找对应的数据，由于B+树高度不高，所以占用内存空间不多，同时，由于叶子节点保存的是数据而不是地址，减少了磁盘IO，同时如果叶子节点保存的是地址而不是数据的话，不便于范围遍历的查找，无法充分利用B+树的特点与优势。