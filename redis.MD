#############################################################
# Redis，Memcached，MongoDB 对比
#############################################################

=============================================================
= 应用场景对比
=============================================================
1.memcached：单一键值对内存缓存的，做对象缓存无可替代的分布式缓存；

2.redis：是算法和数据结构的集合，快速的数据结构操作是他最大的特点，支持数据持久化；

3.mongodb是bson结构、介于rdb和nosql之间的，更松散更灵活的，但是不支持事务，只用作非重要数据存储。


=============================================================
= memcached和redis性能对比
= memcached 多线程，redis单线程
=============================================================
Redis的作者Salvatore Sanfilippo曾经对这两种基于内存的数据存储系统进行过比较：

Redis支持服务器端的数据操作：Redis相比Memcached来说，拥有更多的数据结构和并支持更丰富的数据操作，通常在Memcached里，你需要将数据拿到客户端来进行类似的修改再set回去。这大大增加了网络IO的次数和数据体积。在Redis中，这些复杂的操作通常和一般的GET/SET一样高效。所以，如果需要缓存能够支持更复杂的结构和操作，那么Redis会是不错的选择。

内存使用效率对比：使用简单的key-value存储的话，Memcached的内存利用率更高，
而如果Redis采用hash结构来做key-value存储，由于其组合式的压缩，其内存利用率会高于Memcached。
性能对比：由于Redis只使用单核，而Memcached可以使用多核，所以平均每一个核上Redis在存储小数据时比Memcached性能更高。而在100k以上的数据中，Memcached性能要高于Redis，虽然Redis最近也在存储大数据的性能上进行优化，但是比起Memcached，还是稍有逊色。



=============================================================
= Memcached 和redis区别
=============================================================
1. 支持数据类型不同
Memcached仅支持简单的key-value结构的数据记录不同。
redis支持5中数据类型

2.内存管理机制不同
在Redis中，并不是所有的数据都一直存储在内存中的。这是和Memcached相比一个最大的区别。当物理内存用完时，Redis可以将一些很久没用到的value交换到磁盘。Redis只会缓存所有的key的信息，如果Redis发现内存的使用量超过了某一个阀值，将触发swap的操作，Redis根据“swappability = age*log(size_in_memory)”计算出哪些key对应的value需要swap到磁盘。然后再将这些key对应的value持久化到磁盘中，同时在内存中清除。这种特性使得Redis可以保持超过其机器本身内存大小的数据。当然，机器本身的内存必须要能够保持所有的key，毕竟这些数据是不会进行swap操作的。同时由于Redis将内存中的数据swap到磁盘中的时候，提供服务的主线程和进行swap操作的子线程会共享这部分内存，所以如果更新需要swap的数据，Redis将阻塞这个操作，直到子线程完成swap操作后才可以进行修改。当从Redis中读取数据的时候，如果读取的key对应的value不在内存中，那么Redis就需要从swap文件中加载相应数据，然后再返回给请求方。 这里就存在一个I/O线程池的问题。在默认的情况下，Redis会出现阻塞，即完成所有的swap文件加载后才会相应。这种策略在客户端的数量较小，进行批量操作的时候比较合适。但是如果将Redis应用在一个大型的网站应用程序中，这显然是无法满足大并发的情况的。所以Redis运行我们设置I/O线程池的大小，对需要从swap文件中加载相应数据的读取请求进行并发操作，减少阻塞的时间。

对于像Redis和Memcached这种基于内存的数据库系统来说，内存管理的效率高低是影响系统性能的关键因素。传统C语言中的malloc/free函数是最常用的分配和释放内存的方法，但是这种方法存在着很大的缺陷：首先，对于开发人员来说不匹配的malloc和free容易造成内存泄露；其次频繁调用会造成大量内存碎片无法回收重新利用，降低内存利用率；最后作为系统调用，其系统开销远远大于一般函数调用。所以，为了提高内存的管理效率，高效的内存管理方案都不会直接使用malloc/free调用。Redis和Memcached均使用了自身设计的内存管理机制，但是实现方法存在很大的差异，下面将会对两者的内存管理机制分别进行介绍。

Memcached默认使用Slab Allocation机制管理内存，其主要思想是按照预先规定的大小，将分配的内存分割成特定长度的块以存储相应长度的key-value数据记录，以完全解决内存碎片问题。Slab Allocation机制只为存储外部数据而设计，也就是说所有的key-value数据都存储在Slab Allocation系统里，而Memcached的其它内存请求则通过普通的malloc/free来申请，因为这些请求的数量和频率决定了它们不会对整个系统的性能造成影响Slab Allocation的原理相当简单。 如图所示，它首先从操作系统申请一大块内存，并将其分割成各种尺寸的块Chunk，并把尺寸相同的块分成组Slab Class。其中，Chunk就是用来存储key-value数据的最小单位。每个Slab Class的大小，可以在Memcached启动的时候通过制定Growth Factor来控制。假定图中Growth Factor的取值为1.25，如果第一组Chunk的大小为88个字节，第二组Chunk的大小就为112个字节，依此类推。


3. 数据持久化支持
Redis虽然是基于内存的存储系统，但是它本身是支持内存数据的持久化的，而且提供两种主要的持久化策略：RDB快照和AOF日志。而memcached是不支持数据持久化操作的。


4. 集群管理的不同

Memcached是全内存的数据缓冲系统，Redis虽然支持数据的持久化，但是全内存毕竟才是其高性能的本质。作为基于内存的存储系统来说，机器物理内存的大小就是系统能够容纳的最大数据量。如果需要处理的数据量超过了单台机器的物理内存大小，就需要构建分布式集群来扩展存储能力。

Memcached本身并不支持分布式，因此只能在客户端通过像一致性哈希这样的分布式算法来实现Memcached的分布式存储。下图给出了Memcached的分布式存储实现架构。当客户端向Memcached集群发送数据之前，首先会通过内置的分布式算法计算出该条数据的目标节点，然后数据会直接发送到该节点上存储。但客户端查询数据时，同样要计算出查询数据所在的节点，然后直接向该节点发送查询请求以获取数据。







#############################################################
# Redis环境安装
#############################################################

$sudo apt-get install redis-server
nohup sudo /opt/redis/redis-server /opt/redis/redis.conf &

$redis-cli  # 起动客户端
redis 127.0.0.1:6379>
redis 127.0.0.1:6379> ping
PONG

#############################################################
# Redis数据类型
#############################################################

=============================================================
= 1. Strings - 字符串
=============================================================

Redis的字符串是字节序列。在Redis中字符串是二进制安全的，这意味着他们有一个已知的长度，是没有任何特殊字符终止决定的，
所以可以存储任何东西，最大长度可达512兆。
例子

redis 127.0.0.1:6379> SET name "yiibai"
OK
redis 127.0.0.1:6379> GET name
"yiibai"

在上面的例子使用Redis命令set和get，Redis的名称是yiibai的键存储在Redis的字符串值。

注：字符串值可以存储最大512兆字节的长度。

=============================================================
= 2. Hashes - 哈希值
=============================================================
Redis的哈希键值对的集合。 Redis的哈希值是字符串字段和字符串值之间的映射，所以它们被用来表示对象
例子

redis 127.0.0.1:6379> HMSET user:1 username yiibai password yiibai points 200
OK
redis 127.0.0.1:6379> HGETALL user:1

1) "username"
2) "yiibai"
3) "password"
4) "yiibai"
5) "points"
6) "200"

在上面的例子中的哈希数据类型，用于存储包含用户的基本信息用户的对象。这里HMSET，HGETALL对于Redis 命令 user:1 是键。

每个哈希可存储多达232 - 1个 字段 - 值对(超过4十亿)。


Redis hash是一个string类型的field和value的映射表.它的添加、删除操作都是 O(1) （平均）。
hash特别适合用于存储对象。相较于将对象的每个字段存成单个 string类型。将一个对象存
储在 hash 类型中会占用更少的内存，并且可以更方便的存取整个对象。省内存的原因是新
建一个hash对象时开始是用 zipmap（又称为small hash）来存储的。这个zipmap其实并不
是hash table，但是zipmap 相比正常的hash实现可以节省不少 hash本身需要的一些元数据
存储开销。尽管zipmap的添加，删除，查找都是 O(n)，但是由于一般对象的 field数量都不 
太多。所以使用 zipmap 也是很快的,也就是说添加删除平均还是 O(1)。如果 field 或者 value
的大小超出一定限制后， Redis会在内部自动将zipmap替换成正常的hash实现.  这个限制可
以在配置文件中指定
hash-max-zipmap-entries 64 #配置字段最多64个
hash-max-zipmap-value 512 #配置value 最大为512字节
1 hset
设置hash field为指定值，如果 key 不存在，则先创建。
redis 127.0.0.1:6379> hset myhash field1 Hello
(integer) 1
redis 127.0.0.1:6379>
2 hsetnx
设置hash field为指定值，如果 key 不存在，则先创建。如果 field已经存在，返回0，nx是
not exist的意思。
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 1
redis 127.0.0.1:6379> hsetnx myhash field "Hello"
(integer) 0
redis 127.0.0.1:6379>
第一次执行是成功的，但第二次执行相同的命令失败，原因是field已经存在了。
3 hmset
同时设置hash的多个field。
redis 127.0.0.1:6379> hmset myhash field1 Hello field2 World
OK
redis 127.0.0.1:6379>
4 hget
获取指定的hash field。
redis 127.0.0.1:6379> hget myhash field1
"Hello"
redis 127.0.0.1:6379> hget myhash field2
"World"
redis 127.0.0.1:6379> hget myhash field3
(nil)
redis 127.0.0.1:6379> 
由于数据库没有field3，所以取到的是一个空值nil
5 hmget
获取全部指定的hash filed。
redis 127.0.0.1:6379> hmget myhash field1 field2 field3
1) "Hello"
2) "World"
3) (nil)
redis 127.0.0.1:6379>
由于数据库没有field3，所以取到的是一个空值nil
6 hincrby
指定的hash filed  加上给定值。
redis 127.0.0.1:6379> hset myhash field3 20
(integer) 1
redis 127.0.0.1:6379> hget myhash field3
"20"
redis 127.0.0.1:6379> hincrby myhash field3 -8
(integer) 12
redis 127.0.0.1:6379> hget myhash field3
"12"
redis 127.0.0.1:6379>
在本例中我们将field3 的值从20 降到了12，即做了一个减8 的操作。
7 hexists
测试指定field是否存在。
redis 127.0.0.1:6379> hexists myhash field1
(integer) 1
redis 127.0.0.1:6379> hexists myhash field9
(integer) 0
redis 127.0.0.1:6379>
通过上例可以说明field1存在，但field9是不存在的。
8 hlen
返回指定hash的field数量。
redis 127.0.0.1:6379> hlen myhash
(integer) 4
redis 127.0.0.1:6379>
通过上例可以看到myhash中有4 个field。
9 hdel
删除指定hash的field。
redis 127.0.0.1:6379> hlen myhash
(integer) 4
redis 127.0.0.1:6379> hdel myhash field1
(integer) 1
redis 127.0.0.1:6379> hlen myhash
(integer) 3
redis 127.0.0.1:6379>
10 hkeys
返回hash的所有field。
redis 127.0.0.1:6379> hkeys myhash
1) "field2"
2) "field"
3) "field3"
redis 127.0.0.1:6379>
说明这个hash中有3 个field
11 hvals
返回hash的所有value。
redis 127.0.0.1:6379> hvals myhash
1) "World"
2) "Hello"
3) "12"
redis 127.0.0.1:6379>
说明这个hash中有3 个field
12 hgetall
获取某个hash中全部的 filed及value。
redis 127.0.0.1:6379> hgetall myhash
1) "field2"
2) "World"
3) "field"
4) "Hello"
5) "field3"
6) "12"
redis 127.0.0.1:6379>
可见，一下子将myhash 中所有的field及对应的value都取出来了。


=============================================================
= 3. Lists - 列表
=============================================================
Redis的list类型其实就是一个每个子元素都是string类型的双向链表。链表的最大长度是(2的32次方)。
Redis的列表是简单的字符串列表，排序插入顺序。可以添加元素到Redis列表的头部或尾部。
列表的最大长度为232- 1元素(4294967295，每个列表中的元素超过4十亿)。

实例：

　1、lpush

　　在key对应list的头部添加字符串元素：
redis 127.0.0.1:6379> lpush mylist "world"
(integer) 1
redis 127.0.0.1:6379> lpush mylist "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379>

　　在此处我们先插入了一个world，然后在world的头部插入了一个hello。其中lrange是用于取mylist的内容。

　　2、rpush

　　在key对应list的尾部添加字符串元素：
redis 127.0.0.1:6379> rpush mylist2 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist2 "world"
(integer) 2
redis 127.0.0.1:6379> lrange mylist2 0 -1
1) "hello"
2) "world"
redis 127.0.0.1:6379>

　　在此处我们先插入了一个hello，然后在hello的尾部插入了一个world。

　　3、linsert

　　在key对应list的特定位置之前或之后添加字符串元素：
redis 127.0.0.1:6379> rpush mylist3 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist3 "world"
(integer) 2
redis 127.0.0.1:6379> linsert mylist3 before "world" "there"
(integer) 3
redis 127.0.0.1:6379> lrange mylist3 0 -1
1) "hello"
2) "there"
3) "world"
redis 127.0.0.1:6379>

　　在此处我们先插入了一个hello，然后在hello的尾部插入了一个world，然后又在world的前面插入了there。

　　4、lset

　　设置list中指定下标的元素值(下标从0开始)：
redis 127.0.0.1:6379> rpush mylist4 "one"
(integer) 1
redis 127.0.0.1:6379> rpush mylist4 "two"
(integer) 2
redis 127.0.0.1:6379> rpush mylist4 "three"
(integer) 3
redis 127.0.0.1:6379> lset mylist4 0 "four"
OK
redis 127.0.0.1:6379> lset mylist4 -2 "five"
OK
redis 127.0.0.1:6379> lrange mylist4 0 -1
1) "four"
2) "five"
3) "three"
redis 127.0.0.1:6379>

　　在此处我们依次插入了one,two,three，然后将标是0的值设置为four，再将下标是-2的值设置为five。

　　5、lrem

　　从key对应list中删除count个和value相同的元素。

　　count>0时，按从头到尾的顺序删除，具体如下：
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist5 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist5 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist5 2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist5 0 -1
1) "foo"
2) "hello"
redis 127.0.0.1:6379>

　　count<0时，按从尾到头的顺序删除，具体如下：
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist6 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist6 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist6 -2 "hello"
(integer) 2
redis 127.0.0.1:6379> lrange mylist6 0 -1
1) "hello"
2) "foo"
redis 127.0.0.1:6379>

　　count=0时，删除全部，具体如下：
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 1
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 2
redis 127.0.0.1:6379> rpush mylist7 "foo"
(integer) 3
redis 127.0.0.1:6379> rpush mylist7 "hello"
(integer) 4
redis 127.0.0.1:6379> lrem mylist7 0 "hello"
(integer) 3
redis 127.0.0.1:6379> lrange mylist7 0 -1
1) "foo"
redis 127.0.0.1:6379>


=============================================================
= 4. Sets - 集合
=============================================================
Redis集合是字符串的无序集合。在Redis中可以添加，删除和测试文件是否存在在O(1)的时间复杂度的成员。


redis 127.0.0.1:6379> sadd tutoriallist redis
(integer) 1
redis 127.0.0.1:6379> sadd tutoriallist mongodb
(integer) 1
redis 127.0.0.1:6379> sadd tutoriallist rabitmq
(integer) 1
redis 127.0.0.1:6379> sadd tutoriallist rabitmq
(integer) 0
redis 127.0.0.1:6379> smembers tutoriallist

1) "rabitmq"
2) "mongodb"
3) "redis"

注意：在上面的例子中rabitmq设置属性加两次，但由于唯一性只加一次。

成员中集最大数量为232 - 1(4294967295，集合成员超过4十亿)。



#############################################################
# 5. zset 集合排序
#############################################################

Redis的集合排序类似于Redis集合，字符串不重复的集合。不同的是，一个有序集合的每个成员关联分数，
用于以便采取有序set命令，从最小的到最大的分数有关。虽然成员都是独一无二的，分数可能会重复。
例子


127.0.0.1:6379> zadd zset1 4 one
127.0.0.1:6379> zadd zset1 1 two
127.0.0.1:6379> zadd zset1 -4 three
127.0.0.1:6379> zadd zset1 -4 four
127.0.0.1:6379> zrange zset1 0 -1 withscores
1) "four"
2) "-4"
3) "three"
4) "-4"
5) "two"
6) "1"
7) "one"
8) "4"

127.0.0.1:6379> zrem zset1 two
(integer) 1
127.0.0.1:6379> zrange zset1 0 -1 withscores
1) "four"
2) "-4"
3) "three"
4) "-4"
5) "one"
6) "4"

127.0.0.1:6379> ZINCRBY zset1 3 four
"-1"
127.0.0.1:6379> zrange zset1 0 -1 withscores
1) "three"
2) "-4"
3) "four"
4) "-1"
5) "one"
6) "4"


127.0.0.1:6379> ZRANK zset1 one
(integer) 2


127.0.0.1:6379> zrevrank zset1 four   # 返回从大到小排序后的下标
(integer) 1

127.0.0.1:6379> zrevrange zset1 0 -1 withscores   # 从大到小
1) "one"
2) "4"
3) "four"
4) "-1"
5) "three"
6) "-4"


127.0.0.1:6379> zrangebyscore zset1 -4 -1 withscores  # 按着分数返回
1) "three"
2) "-4"
3) "four"
4) "-1"

127.0.0.1:6379> zcard zset1
(integer) 3

127.0.0.1:6379> zcount zset1 -1 4   # 返回给定分数区间的数量(闭区间)
(integer) 2

127.0.0.1:6379> zremrangebyrank zset1 1 2         # 按着索引批量删除
(integer) 2
127.0.0.1:6379> zrange zset1 0 -1 withscores
1) "three"
2) "-4"

127.0.0.1:6379> zremrangebyscore zset1 -4 2       # 按着分数批量删除
(integer) 1
127.0.0.1:6379> zrange zset1 0 -1 withscores
(empty list or set)

