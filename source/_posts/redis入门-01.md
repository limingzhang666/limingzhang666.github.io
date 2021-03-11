---
title: redis入门-01
copyright: true
comments: true
related_posts: true
date: 2021-03-11 21:20:44
tags: redis
categories: redis
---

# 基础概念

## 传统的ACID

关系型数据库遵循ACID规则
事务在英文中是transaction，和现实世界中的交易很类似，它有如下四个特性：

1、A (Atomicity) 原子性
原子性很容易理解，也就是说事务里的所有操作要么全部做完，要么都不做，事务成功的条件是事务里的所有操作都成功，只要有一个操作失败，整个事务就失败，需要回滚。比如银行转账，从A账户转100元至B账户，分为两个步骤：1）从A账户取100元；2）存入100元至B账户。这两步要么一起完成，要么一起不完成，如果只完成第一步，第二步失败，钱会莫名其妙少了100元。

2、C (Consistency) 一致性
一致性也比较容易理解，也就是说数据库要一直处于一致的状态，事务的运行不会改变数据库原本的一致性约束。

3、I (Isolation) 独立性
所谓的独立性是指并发的事务之间不会互相影响，如果一个事务要访问的数据正在被另外一个事务修改，只要另外一个事务未提交，它所访问的数据就不受未提交事务的影响。比如现有有个交易是从A账户转100元至B账户，在这个交易还未完成的情况下，如果此时B查询自己的账户，是看不到新增加的100元的

4、D (Durability) 持久性
持久性是指一旦事务提交后，它所做的修改将会永久的保存在数据库上，即使出现宕机也不会丢失。



## CAP

C:Consistency（强一致性）

A:Availability（可用性）

P:Partition tolerance（分区容错性）



### CAP的三进二

CAP理论就是说在分布式存储系统中，最多只能实现上面的两点。
而由于当前的网络硬件肯定会出现延迟丢包等问题，所以

**分区容忍性是我们必须需要实现的。**

**所以我们只能在一致性和可用性之间进行权衡，没有NoSQL系统能同时保证这三点**。

**C:强一致性 A：高可用性 P：分布式容忍性**

- CA 传统Oracle数据库 

- AP 大多数网站架构的选择

- CP Redis、Mongodb

 注意：分布式架构的时候必须做出取舍。
一致性和可用性之间取一个平衡。多余大多数web应用，其实并不需要强一致性。因此牺牲C换取P，这是目前分布式数据库产品的方向

#### 一致性与可用性的决择

​	对于web2.0网站来说，关系数据库的很多主要特性却往往无用武之地

#### 数据库事务一致性需求 

​	很多web实时系统并不要求严格的数据库事务，对读一致性的要求很低， 有些场合对写一致性要求并不高。允许实现最终一致性。

#### 数据库的写实时性和读实时性需求

​	对关系数据库来说，插入一条数据之后立刻查询，是肯定可以读出来这条数据的，但是对于很多web应用来说，并不要求这么高的实时性，比方说发一条消息之 后，过几秒乃至十几秒之后，我的订阅者才看到这条动态是完全可以接受的。

#### 对复杂的SQL查询，特别是多表关联查询的需求 

　　任何大数据量的web系统，都非常忌讳多个大表的关联查询，以及复杂的数据分析类型的报表查询，特别是SNS类型的网站，从需求以及产品设计角 度，就避免了这种情况的产生。往往更多的只是单表的主键查询，以及单表的简单条件分页查询，SQL的功能被极大的弱化了。

### 经典CAP图

 CAP理论的核心是：一个分布式系统不可能同时很好的满足一致性，可用性和分区容错性这三个需求，
最多只能同时较好的满足两个。
因此，根据 CAP 原理将 NoSQL 数据库分成了满足 CA 原则、满足 CP 原则和满足 AP 原则三 大类：
CA - 单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大。
CP - 满足一致性，分区容忍必的系统，通常性能不是特别高。
AP - 满足可用性，分区容忍性的系统，通常可能对一致性要求低一些。

![](/uploads/redis/Image.bmp)



## BASE

BASE就是为了解决关系数据库强一致性引起的问题而引起的可用性降低而提出的解决方案。

BASE其实是下面三个术语的缩写：
    基本可用（Basically Available）
    软状态（Soft state）
    最终一致（Eventually consistent）

它的思想是通过让系统放松对某一时刻数据一致性的要求来换取系统整体伸缩性和性能上改观。为什么这么说呢，缘由就在于大型系统往往由于地域分布和极高性能的要求，不可能采用分布式事务来完成这些指标，要想获得这些指标，我们必须采用另外一种方式来完成，这里BASE就是解决这个问题的办法





# REDIS 入门

## 功能：

- 内存存储和持久化：redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务
- 取最新N个数据的操作，如：可以将最新的10条评论的ID放在Redis的List集合里面
- 模拟类似于HttpSession这种需要设定过期时间的功能
- 发布、订阅消息系统
- 定时器、计数器

## 基础知识讲解

- 单进程模型来处理客户端的请求。对读写等事件的响应是通过对epoll函数的包装来做到的。Redis的实际处理速度完全依靠主进程的执行效率

- Epoll是Linux内核为处理大批量文件描述符而作了改进的epoll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。
- 默认16个数据库，类似数组下表从零开始，初始默认使用零号库
- Select命令切换数据库
- Dbsize查看当前数据库的key的数量
- Flushdb：清空当前库
- Flushall；通杀全部库
- 统一密码管理，16个库都是同样密码，要么都OK要么一个也连接不上
- Redis索引都是从零开始
- 为什么默认端口是6379



## 数据类型

### String（字符串）

string是redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。

string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。

string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是512M

### Hash（哈希，类似java里的Map）

Hash（哈希）
Redis hash 是一个键值对集合。
Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象

### List（列表）

Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。
它的底层实际是个链表

### Set（集合）

Redis的Set是string类型的无序集合。它是通过HashTable实现实现的，

### Zset(sorted set：有序集合)

zset(sorted set：有序集合)
Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。
不同的是每个元素都会关联一个double类型的分数。
redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复。



## 常用命令：

http://redisdoc.com/

### Redis 键(key)

![](/uploads/redis/redis_key.bmp)

-  keys *
-  exists key的名字，判断某个key是否存在
-  move key db   --->当前库就没有了，被移除了
-  expire key 秒钟：为给定的key设置过期时间
-  ttl key 查看还有多少秒过期，-1表示永不过期，-2表示已过期
-  type key 查看你的key是什么类型

### Redis字符串(String)

![](/uploads/redis/redis-string01.bmp)

![](/uploads/redis/redis-string02.bmp)

- set/get/del/append/strlen

- Incr/decr/incrby/decrby,一定要是数字才能进行加减
- getrange/setrange
-  setex(set with expire)键秒值/setnx(set if not exist)
- mset/mget/msetnx

- getset(先get再set)

### Redis列表(List)

![](/uploads/redis/redis-list01.bmp)

![](/uploads/redis/redis-list02.bmp)

- lpush/rpush/lrange
-  lpop/rpop
-  lindex，按照索引下标获得元素(从上到下)
- llen
-  lrem key 删N个value
-  ltrim key 开始index 结束index，截取指定范围的值后再赋值给key
-  rpoplpush 源列表 目的列表
-  lset key index value
-  linsert key  before/after 值1 值2
- 它是一个字符串链表，left、right都可以插入添加；
  如果键不存在，创建新的链表；
  如果键已存在，新增内容；
  如果值全移除，对应的键也就消失了。
  链表的操作无论是头和尾效率都极高，但假如是对中间元素进行操作，效率就很惨淡了。

### Redis集合(Set)

![](/uploads/redis/redis-set.bmp)

- sadd/smembers/sismember
-  scard，获取集合里面的元素个数
-  srem key value 删除集合中元素
-  srandmember key 某个整数(随机出几个数)
-  spop key 随机出栈
-  smove key1 key2 在key1里某个值      作用是将key1里的某个值赋给key2
- 数学集合类
  - 差集：sdiff
  - 交集：sinter
  - 并集：sunion

### Redis哈希(Hash)

![](/uploads/redis/redis-hash.bmp)

- hset/hget/hmset/hmget/hgetall/hdel
- hlen
-  hexists key 在key里面的某个值的key
- hkeys/hvals
-  hincrby/hincrbyfloat
-  hsetnx

### Redis有序集合Zset(sorted set)

![](/uploads/redis/redis-zset01.bmp)

![](/uploads/redis/redis-zset02.bmp)

- zadd/zrange
-  zrangebyscore key 开始score 结束score
-  zrem key 某score下对应的value值，作用是删除元素
-  zcard/zcount key score区间/zrank key values值，作用是获得下标值/zscore key 对应值,获得分数
-  zrevrank key values值，作用是逆序获得下标值
-  zrevrange
-  zrevrangebyscore  key 结束score 开始score





## 解析配置文件redis.conf

TODO

