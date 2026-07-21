redis：
键值型数据库
key-value 结构

nosql 数据库一般都是在内存存储，扩展性水平

使用场景： 数据结构不固定，对一致安全性要求不高 ， 对性能要求

特征：
key value
单线程，原子性
速度快（内存，IO 多路，良好编码）
数据持久化
主从集群，分片集群


key、 一般 string
 value 类型多种多样：
 5 个基本类型
 string，hash，list set sorted set
 3 个特殊类型
 geo bitmap hyperlog

String 常用命令：

hash 本质是通过某种哈希函数算出一个值
哈希表还用 hash 思想实现的数据结构
hashmap 是 java 里面基于哈希表实现的一种 map
redis hash 是一种 field value 的数据类型

redis 里面的 hash 类型： 一个 key 里面放很多 field -value



Java 中使用 redis 的客户端：
Jedis
Spring data redis：
spring data 是spring中数据操作的模块，包含对各种数据库的集成
提供了对不同 Redis 客户端的整合


SpringDataReids 的序列化方式。

Redis Template 可以接收任意 Object 作为值写入 redis，只不过写入前会把 Object 序列化为字节形式，默认是采用 JDK 序列化。

key 用 string 序列化 值用那个 json 序列化。

RedisTemplate 的两种序列化实践方案：
方案一：
1.自定义 redistemplate
2.修改 redistemplate 的序列化器为 genericJackson2JsonRedisSerializer
方案二：
1 使用 StringRedisTemplate
2 写入 redis 时，手动把对象序列化为 Json
3 读取 Redis 时，手动把读取到的 Json 反序列化为对象





















 