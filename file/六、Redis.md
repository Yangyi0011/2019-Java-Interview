# 一、Redis简介

​	Redis是一个基于单线程的键值对内存数据库，数据库中的键值对由字典保存。每个数据库都有一个对应的字典，这个字典被称之为键空间【key space】。当用户添加一个键值对到数据库时（不论键值对是什么类型）， 程序就将该键值对添加到键空间。

​	字典的键是一个字符串对象。字典的值则可以是包括【字符串（String）、列表（List）、哈希表（Hash）、集合（Set）或有序集（ZSet）】在内的任意一种 Redis 类型对象。

**Redis Object**

```c
/*
 * Redis 对象
 */
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 对齐位
    unsigned notused:2;

    // 编码方式
    unsigned encoding:4;

    // LRU 时间（相对于 server.lruclock）
    unsigned lru:22;

    // 引用计数
    int refcount;

    // 指向对象的值
    void *ptr;

} robj;
```

`type` 、 `encoding` 和 `ptr` 是最重要的三个属性。

`type` 记录了对象所保存的值的类型，它的值可能是以下常量的其中一个（定义位于 `redis.h`）：

```c
/*
 * 对象类型
 */
#define REDIS_STRING 0  // 字符串
#define REDIS_LIST 1    // 列表
#define REDIS_SET 2     // 集合
#define REDIS_ZSET 3    // 有序集
#define REDIS_HASH 4    // 哈希表
```

`encoding` 记录了对象所保存的值的编码，它的值可能是以下常量的其中一个（定义位于 `redis.h`）：

```c
/*
 * 对象编码
 */
#define REDIS_ENCODING_RAW 0            // 编码为字符串
#define REDIS_ENCODING_INT 1            // 编码为整数
#define REDIS_ENCODING_HT 2             // 编码为哈希表
#define REDIS_ENCODING_ZIPMAP 3         // 编码为 zipmap，Redis2.6后不再使用
#define REDIS_ENCODING_LINKEDLIST 4     // 编码为双端链表
#define REDIS_ENCODING_ZIPLIST 5        // 编码为压缩列表
#define REDIS_ENCODING_INTSET 6         // 编码为整数集合
#define REDIS_ENCODING_SKIPLIST 7       // 编码为跳跃表
```

`ptr` 是一个指针，指向实际保存值的数据结构，这个数据结构由 `type` 属性和 `encoding` 属性决定。

![Redis底层](../img/Redis底层.svg)

`REDIS_ENCODING_ZIPMAP` 没有出现在图中， 因为从 Redis 2.6 开始， ZIPMAP不再是任何数据类型的底层结构。

**小结：**

- Redis 使用自己实现的对象机制来实现类型判断、命令多态和基于引用计数的垃圾回收。
- 一种 Redis 类型的键可以有多种底层实现。
- Redis 会预分配一些常用的数据对象，并通过共享这些对象来减少内存占用，和避免频繁地为小对象分配内存。

# 二、Redis数据结构及使用场景

## 1、String

​	`REDIS_STRING` （字符串）是 Redis 使用得最为广泛的数据类型，就是普通的 set 和 get，做简单的 KV 缓存。 

[Redis_String详情](https://redisbook.readthedocs.io/en/latest/datatype/string.html)

## 2、Hash

`REDIS_HASH` （哈希表）类似 map 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 redis 里，然后每次读写缓存的时候，可以就操作 hash 里的**某个字段**。

```bash
hset person name bingo
hset person age 20
hset person id 1
hget person name

person = {
    "name": "bingo",
    "age": 20,
    "id": 1
}
```

[Redis_Hash详情](https://redisbook.readthedocs.io/en/latest/datatype/hash.html)

## 3、List

list 是有序列表，这个可以玩儿出很多花样。

比如可以通过 list 存储一些**列表型的数据结构**，类似粉丝列表、文章的评论列表之类的东西。

比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

```bash
# 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有。
lrange mylist 0 -1
```

比如可以搞个简单的消息队列，从 list 头怼进去，从 list 尾巴那里弄出来。

```bash
lpush mylist 1
lpush mylist 2
lpush mylist 3 4 5

# 1
rpop mylist
```

​	**[Redis_List详情](https://redisbook.readthedocs.io/en/latest/datatype/list.htm)**

## 4、Set

set 是无序集合，自动去重。

直接基于 set 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 jvm 内存里的 HashSet 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于 redis 进行全局的 set 去重。

可以基于 set 玩儿交集、并集、差集的操作，比如交集吧，可以把两个人的粉丝列表整一个交集，看看俩人的共同好友是谁。

```bash
#-------操作一个set-------
# 添加元素
sadd mySet 1

# 查看全部元素
smembers mySet

# 判断是否包含某个值
sismember mySet 3

# 删除某个/些元素
srem mySet 1
srem mySet 2 4

# 查看元素个数
scard mySet

# 随机删除一个元素
spop mySet

#-------操作多个set-------
# 将一个set的元素移动到另外一个set
smove yourSet mySet 2

# 求两set的交集
sinter yourSet mySet

# 求两set的并集
sunion yourSet mySet

# 求在yourSet中而不在mySet中的元素
sdiff yourSet mySet
```

​	[Redis_Set详情](https://redisbook.readthedocs.io/en/latest/datatype/set.html)

## 5、Sorted Set【ZSet】

sorted set 是排序的 set，去重但可以排序，写进去的时候给一个分数，自动根据分数排序。

```bash
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu

# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
zrevrange board 0 3

# 获取某用户的排名
zrank board zhaoliuCopy to clipboardErrorCopied
```

[Redis_ZSet详情](https://redisbook.readthedocs.io/en/latest/datatype/sorted_set.html)

# 三、Redis集群模式

## 1、主从模式

![redis主从](../img/redis-master-slave.png)

单机的 redis，能够承载的 QPS 大概就在上万到几万不等。对于缓存来说，一般都是用来支撑**读高并发**的。因此架构做成主从(master-slave)架构，一主多从，主负责写，并且将数据复制到其它的 slave 节点，从节点负责读。所有的**读请求全部走从节点**，实现**主从复制，读写分离**，还可以很轻松实现水平扩容，**支撑读高并发**。

主从模式的问题：当从机故障宕机时，redis不能通知客户端哪个节点不可用了，需要手动去更改客户端的配置重新连接，而主机故障宕机时，从机因没有主节点而同步中断，需要人工手动进行故障转移。为了解决这两个问题，在2.8版本之后redis正式提供了sentinel（哨兵）架构。



### 主从复制的核心原理

![](C:\Users\Yang\Desktop\面试准备/img/redis-master-slave-replication.png)

当启动一个 slave node 的时候，它会发送一个 `PSYNC` 命令给 master node。

如果这是 slave node 初次连接到 master node，那么会触发一次 `full resynchronization` **全量复制**。此时 master 会启动一个后台线程，开始生成一份 `RDB` 快照文件，同时还会将从客户端 client 新收到的所有写命令缓存在内存中。`RDB` 文件生成完毕后， master 会将这个 `RDB` 发送给 slave，slave 会先**写入本地磁盘，然后再从本地磁盘加载到内存**中，接着 master 会将内存中缓存的写命令发送到 slave，slave 也会同步这些数据。slave node 如果跟 master node 有网络故障，断开了连接，会自动重连，连接之后 master node 仅会复制给 slave 部分缺少的数据。

### 主从复制的断点续传

从 redis2.8 开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份。【**增量复制**】

master node 会在内存中维护一个 backlog，master 和 slave 都会保存一个 replica offset 还有一个 master run id，offset 就是保存在 backlog 中的。如果 master 和 slave 网络连接断掉了，slave 会让 master 从上次 replica offset 开始继续复制，如果没有找到对应的 offset，那么就会执行一次 `resynchronization`**全量复制**。

### 无磁盘化复制

master 在内存中直接创建 `RDB`，然后发送给 slave，不会在自己本地落地磁盘了。只需要在配置文件中开启 `repl-diskless-sync yes` 即可。

```bash
repl-diskless-sync yes

# 等待 5s 后再开始复制，因为要等更多 slave 重新连接过来
repl-diskless-sync-delay 5Copy to clipboardErrorCopied
```

### 过期 key 处理

slave 不会过期 key，只会等待 master 过期 key。如果 master 过期了一个 key，或者通过 LRU 淘汰了一个 key，那么会模拟一条 del 命令发送给 slave。



**[更多Redis主从模式信息](https://doocs.github.io/advanced-java/#/docs/high-concurrency/redis-master-slave)**



## 2、哨兵模式

![redis哨兵](C:\Users\Yang\Desktop\面试准备/img/redis哨兵.png)

哨兵模式是主从模式的故障自动转移版本，由Sentinel节点定期监控集群节点是否出现了故障，当主节点出现故障时，由Redis Sentinel自动完成故障发现和转移，并通知应用方，实现高可用性。若主机下线了，则会在所有从机中选出一个作为【新主机】，下线的主机再次上线时，会自动转为【新主机】的从机。

**哨兵的功能：**

- 集群监控：负责监控 redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

哨兵用于实现 redis 集群的高可用，本身也是分布式的，作为一个哨兵集群去运行，互相协同工作。

- 故障转移时，判断一个 master node 是否宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题。
- 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的。



[**redis-哨兵模式详细**](https://doocs.github.io/advanced-java/#/docs/high-concurrency/redis-sentinel)



## 3、分布式集群

![redis分布式集群](C:\Users\Yang\Desktop\面试准备/img/redis分布式集群.png)

​	redis主从或哨兵模式的每个实例都是**全量存储所有数据**，浪费内存且有木桶效应。为了最大化利用内存，可以采用分布式集群【Redis-Cluster】，就是分布式存储，集群将数据分片存储，每组节点存储一部分数据，从而达到分布式集群的目的。

​	上图是主从模式与分布式集群模式的区别，redis分布式集群中数据是和槽（slot）挂钩的，其总共定义了**16384个槽**，所有的数据根据一致性哈希算法会被映射到这16384个槽中的某个槽中；另一方面，这16384个槽是按照设置被分配到不同的redis节点上。

​	但分布式集群模式会直接导致访问数据方式的改变，比如客户端向A节点发送GET命令但该数据在B节点，redis会返回重定向错误给客户端让客户端再次发送请求，这也直接导致了必须在相同节点才能执行的一些高级功能（如Lua、事务、Pipeline）无法使用。另外还会引发数据分配的一致性hash问题可以参看[这里](https://github.com/crossoverJie/JCSprout/blob/master/MD/Consistent-Hash.md)。

## 4、集群选择

1. 分布式集群的优势在于高可用，将写操作分开到不同的节点，如果写的操作较多且数据量巨大，且不需要高级功能则可能考虑分布式集群。
2. 哨兵的优势在于高可用，支持高级功能，且能在读的操作较多的场景下工作，所以在绝大多数场景中是适合的，缺点是占用内存较大。
3. 主从的优势在于支持高级功能，且能在读的操作较多的场景下工作，但无法保证高可用，不建议在数据要求严格的场景下使用。

# 四、Redis使用策略

## 1、延迟加载

读：当读请求到来时，**先从缓存读，如果读不到就从数据库读，读完之后同步到缓存且添加过期时间。**
写：当写请求到来时，**只写数据库，如mysql。**

优点：仅对请求的数据进行一段时间的缓存，没有请求过的数据就不会被缓存，节省缓存空间；节点出现故障并不是致命的，因为可以从数据库中得到。
缺点：缓存数据可能不是最新的，存在【缓存击穿】、【缓存失效】问题。

## 2、直写

读：当读请求到来时，**只从缓存读。**
写：当写请求到来时，**先写数据库然后同步到缓存，设置为永不过期。**

优点：缓存数据永不过时且为最新。
缺点：每次写入都需要写缓存导致的性能损失；重启节点或故障会导致缓存数据的丢失直到有写操作同步到缓存；大量数据可能没有被读取到，造成资源浪费。

# 五、Redis过期策略

## 1、内存淘汰机制

redis 内存淘汰机制有以下几个：

- noeviction: 不淘汰。当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了。
- **allkeys-lru**：当内存不足以容纳新写入数据时，在**键空间**中，移除最近最少使用的 key（这个是**最常用**的）。
- allkeys-random：当内存不足以容纳新写入数据时，在**键空间**中，随机移除某个 key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的 key 给干掉啊。
- volatile-lru：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，移除最近最少使用的 key（这个一般不太合适）。
- volatile-random：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，**随机移除**某个 key。
- volatile-ttl：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，有**更早过期时间**的 key 优先移除。即优先移除即将过期的key。

## 2、手写一个 LRU 算法

你可以现场手写最原始的 LRU 算法，那个代码量太大了，似乎不太现实。

不求自己纯手工从底层开始打造出自己的 LRU，但是起码要知道如何利用已有的 JDK 数据结构实现一个 Java 版的 LRU。

```java
import java.util.HashMap;
import java.util.LinkedList;

class LRUCache {
    //保存key和值
    private HashMap<Integer, Integer> cacheMap = new HashMap<>();
    /**
    *在此对key进行最近最少使用淘汰，只保存key
    *新添加的key或新被访问的key放到链表尾部，超过容量时，移除链表第一个key
    */
    private LinkedList<Integer> recentlyList = new LinkedList<>();
    
    //链表的最大容量
    private int capacity;
	
    public LRUCache(int capacity) {
        //需要手动指定容量大小
        this.capacity = capacity;
    }
	
    //访问一个key
    public int get(int key) {
        //若需要访问的key不存在，则返回-1
        if (!cacheMap.containsKey(key)) {
            return -1;
        }
        //若key存在
        //1、先从链表中删除该key，因为不知道该key在链表中的顺序
        recentlyList.remove((Integer) key);
        //2、又将该key添加到链表的尾部
        recentlyList.add(key);
        
        //从map中返回该key的对应的value
        return cacheMap.get(key);
    }

    //添加一个key
    public void put(int key, int value) {
        //若该key已存在，则说明此次操作是修改
        if (cacheMap.containsKey(key)) {
            //移除掉链表中的key，后面再添加，保证最近被访问的key总是在链表的尾部
            recentlyList.remove((Integer) key);
            
        //若容量已满，则移除链表中的第一个key，并移除对应的map元素 
        }else if(cacheMap.size() == capacity){
            cacheMap.remove(recentlyList.removeFirst());
        }
        //将新加入的key放到链表尾部
        recentlyList.add(key);
        //更新或添加key和value
        cacheMap.put(key, value);
    }

    public static void main(String[] args) {
        LRUCache2 cache = new LRUCache2(2);
        cache.put(1, 1);
        cache.put(2, 2);
        System.out.println(cache.get(1)); // returns 1
        cache.put(3, 3); // 驱逐 key 2
        System.out.println(cache.get(2)); // returns -1 (not found)
        cache.put(4, 4); // 驱逐 key 1
        System.out.println(cache.get(1)); // returns -1 (not found)
        System.out.println(cache.get(3)); // returns 3
        System.out.println(cache.get(4)); // returns 4
    }
}
```

# 六、Redis缓存问题

## 1、缓存击穿

​	**缓存击穿：**查询一个数据库中不存在的数据，比如商品详情，查询一个不存在的ID，每次都会访问DB，如果有人恶意破坏，很可能直接对DB造成过大地压力。

​	**解决办法：**当通过某一个key去查询数据的时候，如果对应key在数据库中的数据都不存在，则我们应将此key对应的value设置为一个默认的值，存在缓存中，避免对数据库的重复查询。

## 2、缓存失效

​	**缓存失效：**在高并发的环境下，如果此时key对应的缓存失效【过时】，则多个进程就会去同时去查询DB，然后再去同时设置缓存。这个时候如果这个key是系统中的热点key或者同时失效的数量比较多时，DB访问量会瞬间增大，造成过大的压力。

​	**解决办法：**将系统中key的缓存失效时间均匀地错开。

## 3、缓存雪崩

​	对于系统 A，假设每天高峰期每秒 5000 个请求，本来缓存在高峰期可以扛住每秒 4000 个请求，但是缓存机器意外发生了全盘宕机。缓存挂了，此时 1 秒 5000 个请求全部落数据库，数据库必然扛不住，它会报一下警，然后就挂了。此时，如果没用什么特别的方案来处理这个故障，DBA 很着急，重启数据库，但是数据库立马又被新的流量给打死了。这就是缓存雪崩。

**解决办法：**

- 事前：保证redis 高可用，主从+哨兵或redis cluster集群，避免全盘崩溃。
- 事中：本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。
- 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

[**缓存雪崩及其应对方法**](https://doocs.github.io/advanced-java/#/docs/high-concurrency/redis-caching-avalanche-and-caching-penetration?id=%E7%BC%93%E5%AD%98%E9%9B%AA%E5%B4%A9)

## 4、热点key

​	缓存中的某些Key(可能对应用与某个促销商品)对应的value存储在集群中一台机器，使得所有流量涌向同一机器，造成系统性能瓶颈，该问题的挑战在于它无法通过增加机器容量来解决。

1. 客户端热点key缓存：将热点key对应的value缓存在客户端本地，并且设置一个失效时间。
2. 将热点key分散为多个子key，然后存储到缓存集群的不同机器上，这些子key对应的value都和热点key是一样的。

# 七、Redis持久化

- **RDB：**

  RDB 持久化可以在指定的时间间隔内生成数据集的时间点快照（point-in-time snapshot）。

  生成时间策略：

  - **save 900 1**：1写操作/900s，生成一次快照
  - **save 300 10** ：10写操作/300s，生成一次快照
  - **save 60 10000**​：10000写操作/60s，生成一次快照

- **AOF：**

  ​	AOF 持久化记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集。 AOF 文件中的命令全部以 Redis 协议的格式来保存，新命令会被追加到文件的末尾。 Redis 还可以在后台对 AOF 文件进行重写（rewrite），使得 AOF 文件的体积不会超出保存数据集状态所需的实际大小。

  AOF日志记录策略：

  - **appendfsync no**：等操作系统进行数据缓存同步到磁盘。【快，不安全，数据丢失量大】
  - **appendfsync always**：每次更新数据都同步。【慢，安全，严重损耗性能】
  - **appendfsync everysec**：每秒同步一次。【折中，只会丢失1s的数据】

- **两者同时存在时如何选择**

  ​	Redis 还可以同时使用 AOF 持久化和 RDB 持久化。 在这种情况下， 当 Redis 重启时， 它会优先使用 AOF 文件来还原数据集， 因为 AOF 文件保存的数据集通常比 RDB 文件所保存的数据集更完整。

  ​	但实际上持久化会对Redis的性能造成非常严重的影响，如果一定需要保存数据，那么数据就不应该依靠缓存来保存，建议使用其他方式如mysql数据库。所以Redis的持久化意义不大。



# 八、Redis与memcached的区别

## 1、redis 支持复杂的数据结构

redis 相比 memcached 来说，拥有更多的数据结构，能支持更丰富的数据操作。

redis能支持五大数据结构及其相关操作，而memcached只支持简单的key-value字符串。

## 2、redis 原生支持集群模式

在 redis3.x 版本中，便能支持 cluster 模式，而 memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据。

## 3、性能对比

由于 redis 只使用单核【单线程】，而 memcached 可以使用多核【多线程】，所以平均每一个核上 redis 在存储小数据时比 memcached 性能更高。而在 100k 以上的数据中，memcached 性能要高于 redis，虽然 redis 最近也在存储大数据的性能上进行优化，但是比起 memcached，还是稍有逊色。

# 九、缓存与面试

[**缓存细节及其面试套路**](https://doocs.github.io/advanced-java/#/README?id=%E7%BC%93%E5%AD%98)



