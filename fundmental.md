# 通用命令
 set key value [ex seconds] ：设置键值。支持直接设置过期时间（秒），比如  set token:123 "abc" ex 3600 （1小时后过期）。

 get key ：获取键对应的值。

 del key [key ...] ：删除一个或多个键。

 exists key ：检查键是否存在。存在返回 1，不存在返回 0。

 ttl key ：查看键的剩余过期时间（单位：秒）。返回 -1 代表永不过期，-2 代表键已过期/不存在。

 expire key seconds ：给key设置有效期

# String类型
  - string
  - int
  - float
    
*   set: 添加或者修改已经存在的一个String类型的键值对
*   get: 根据key获取String类型的value
*   mset: 批量添加多个String类型的键值对
*   mget: 根据多个key获取多个String类型的value
*   incr: 让一个整型的key自增1
*   incrby:让一个整型的key自增并指定步长，例如：incrby num 2 让num值自增2
*   incrbyfloat: 让一个浮点类型的数字自增并指定步长
*   setnx: 添加一个String类型的键值对，前提是这个key不存在，否则不执行
*   setex: 添加一个String类型的键值对，并且指定有效期
# Hash
key:value(field,value)
*   hset key field value: 添加或者修改hash类型key的field的值
*   hget key field: 获取一个hash类型key的field的值
*   hmset: 批量添加多个hash类型key的field的值
*   hmget: 批量获取多个hash类型key的field的值
*   hgetall: 获取一个hash类型的key中的所有的field和value
*   hkeys: 获取一个hash类型的key中的所有的field
*   hvals: 获取一个hash类型的key中的所有的value
*   hincrby: 让一个hash类型key的字段值自增并指定步长
*   hsetnx: 添加一个hash类型的key的field值，前提是这个field不存在，否则不执行
# List(双向链表)
- 有序
- 元素可以重复
- 插入和删除快
- 查询速度一般

*   lpush key element ... : 向列表左侧插入一个或多个元素
*   lpop key: 移除并返回列表左侧的第一个元素，没有则返回nil
*   rpush key element ... : 向列表右侧插入一个或多个元素
*   rpop key: 移除并返回列表右侧的第一个元素
*   lrange key star end: 返回一段角标范围内的所有元素
*   blpop和brpop: 与lpop和rpop类似，只不过在没有元素时等待指定时间，而不是直接返回nil
# Set
- 无序
- 元素不可重复
- 查找快
- 支持交集、并集、差集等功能

*   sadd key member ... ：向set中添加一个或多个元素
*   srem key member ... ：移除set中的指定元素
*   scard key：返回set中元素的个数
*   sismember key member：判断一个元素是否存在于set中
*   smembers：获取set中的所有元素
*   sinter key1 key2 ... ：求key1与key2的交集
*   sdiff key1 key2 ... ：求key1与key2的差集
*   sunion key1 key2 ...：求key1和key2的并集
# Sorted
- 可排序
- 元素不重复
- 查询速度快

*   zadd key score member: 添加一个或多个元素到sorted set，如果已经存在则更新其score值
*   zrem key member: 删除sorted set中的一个指定元素
*   zscore key member: 获取sorted set中的指定元素的score值
*   zrank key member: 获取sorted set 中的指定元素的排名
*   zcard key: 获取sorted set中的元素个数
*   zcount key min max: 统计score值在给定范围内的所有元素的个数
*   zincrby key increment member: 让sorted set中的指定元素自增，步长为指定的increment值
*   zrange key min max: 按照score排序后，获取指定排名范围内的元素
*   zrangebyscore key min max: 按照score排序后，获取指定score范围内的元素
*   zdiff、zinter、zunion: 求差集、交集、并集
**注意**:所有排名默认升序，降序则在在z后添加rev
