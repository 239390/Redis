# 通用命令
 set key value [ex seconds] ：设置键值。支持直接设置过期时间（秒），比如  set token:123 "abc" ex 3600 （1小时后过期）。

 get key ：获取键对应的值。

 del key [key ...] ：删除一个或多个键。

 exists key ：检查键是否存在。存在返回 1，不存在返回 0。

 ttl key ：查看键的剩余过期时间（单位：秒）。返回 -1 代表永不过期，-2 代表键已过期/不存在。

 expire key seconds ：给已有的键设置过期时间（秒）。

