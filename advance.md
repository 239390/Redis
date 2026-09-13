# 一、 缓存架构基础认知

## 1.1 什么是缓存 (Cache)？
*   定义：数据交换的缓冲区，利用内存的高速读写特性，作为磁盘（数据库）与应用程序之间的高效中间层。
*   常见层级：CPU L1/L2 缓存 rightarrow 浏览器/客户端缓存 rightarrow 业务代码内存缓存 (Caffeine) rightarrow 分布式缓存 (Redis)。

## 1.2 引入缓存的核心价值与代价
*   收益 (收益 > 成本时引入)：
    *   降低后端负载：拦截绝大多数读请求，保护数据库不被高并发击穿。
    *   提升系统性能：毫秒级响应，大幅降低用户感知延迟。
*   引入成本 (Trade-off)：
    *   数据一致性成本：双写（DB + Cache）必然带来短暂或长期的不一致风险。
    *   代码维护成本
    *   运维成本

# 二、 缓存更新策略
## 1.三种更新策略

- 内存淘汰：Redis自动触发（达到max-memory时）
- 超时剔除：设置TTL后自动删除过期数据
- 主动更新：手动删除缓存，解决缓存与数据库不一致问题

## 2.旁路缓存策略
### 2.1 读操作流程
1.  查缓存：命中则直接返回数据；未命中则向下查询数据库。
2.  查库回填：从数据库查询到数据后，将其写入缓存，并设置合理的过期时间 (TTL)。
3.  返回数据：将数据返回给业务方。

### 2.2 写操作流程（重点：为何是先更新DB，再删除缓存？）
*   标准流程：先修改数据库 rightarrow 修改成功后再删除对应的缓存。
*   为什么不先删缓存？（延迟双删）
    *   若先删缓存，再更新 DB，在更新 DB 的这段极短时间窗口内，若有并发读请求，会查询到旧 DB 数据并重新写入缓存，导致脏数据常驻缓存。
*   为什么不更新缓存？
    *   缓存更新需要计算（如反序列化 rightarrow 修改属性 rightarrow 序列化 rightarrow 写回），性能开销大。
    *   并发写时，多线程同时更新缓存，极易导致数据错乱。
    *   直接删除缓存，让下一次读请求重新从 DB 加载，既安全又解耦。
```
@Service
public class CacheAsideService {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private OrderMapper orderMapper; // 模拟业务 Mapper

    // 1. 读操作：先读缓存，未命中查库回填
    public Order getOrderById(Long id) {
        String cacheKey = "order:info:" + id;
        // 查缓存
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            return JSONUtil.toBean(json, Order.class);
        }

        // 查数据库
        Order order = orderMapper.selectById(id);
        if (order != null) {
            // 回填缓存并设置随机TTL，防止缓存雪崩
            int randomTime = 300 + ThreadLocalRandom.current().nextInt(60); 
            redisTemplate.opsForValue().set(cacheKey, JSONUtil.toJsonStr(order), randomTime, TimeUnit.SECONDS);
        }
        return order;
    }

    // 2. 写操作：先更新数据库，再删除缓存
    public void updateOrder(Order order) {
        // 更新数据库
        orderMapper.updateById(order);
        // 删除缓存（下次读取时自动加载最新数据）
        String cacheKey = "order:info:" + order.getId();
        redisTemplate.delete(cacheKey);
    }
}
```
# 三、 缓存三大核心问题与解决方案

引入缓存后，必须面对以下三个经典的分布式问题，并制定兜底方案。

## 3.1 缓存穿透 (Cache Penetration)
*   现象：客户端恶意请求一个在缓存和数据库中都不存在的 Key（如 id=-1）。导致所有请求全部穿透到数据库，缓存失去保护屏障。
*   解决方案：
    1.  缓存空对象（通用轻量方案）：
        *   查库为空时，将 Key 的值设为 null 并缓存一个极短的 TTL（如 3-5 分钟）。
        *   缺点：消耗少量额外内存，且在 TTL 内存在短暂的“数据库更新后，缓存还是空”的不一致期。
    2.  布隆过滤器 (Bloom Filter)：
        *   在缓存前增加一层布隆过滤器，将海量合法 Key 映射到 Hash 集合中。请求进来时先过布隆过滤器，命中不了则直接拦截。
        *   缺点：存在极小概率的误判（即实际不存在的数据被误判为存在），且不支持 Key 的删除操作。
```
    @Service
    public class PenetrationService {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 声明一个全局布隆过滤器
    private BloomFilter<CharSequence> bloomFilter;

    @PostConstruct
    public void init() {
        // 初始化布隆过滤器（预期插入100万数据，误判率0.01）
        bloomFilter = BloomFilter.create(Funnels.stringFunnel(StandardCharsets.UTF_8), 1000000, 0.01);
        // TODO: 在系统启动或数据变更时，将所有合法的ID加入布隆过滤器
    }

    public Order safeGetOrder(Long id) {
        String cacheKey = "order:info:" + id;
        
        // 1. 布隆过滤器拦截（判断一定不存在的数据直接返回）
        if (bloomFilter != null && !bloomFilter.mightContain(id.toString())) {
            return null;
        }

        // 2. 查缓存
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            // 区分空对象标识（如 "NULL"）和真实数据
            return "NULL".equals(json) ? null : JSONUtil.toBean(json, Order.class);
        }

        // 3. 查数据库
        Order order = orderMapper.selectById(id);
        if (order != null) {
            // 存入真实数据
            redisTemplate.opsForValue().set(cacheKey, JSONUtil.toJsonStr(order), 30, TimeUnit.MINUTES);
        } else {
            // 4. 防穿透核心：缓存空对象（设置较短的TTL）
            redisTemplate.opsForValue().set(cacheKey, "NULL", 2, TimeUnit.MINUTES);
        }
        return order;
    }

```

## 3.2 缓存雪崩 (Cache Avalanche)
*   现象：大量缓存 Key 在同一时刻集体失效，或者 Redis 集群宕机，导致海量请求瞬间直达数据库，引发数据库崩溃。
*   解决方案：
    1.  TTL 随机化：在设置过期时间时，给基础 TTL 加上一个随机值（如 base_ttl + random(1~60s)），避免 Key 集中过期。
    2.  多级缓存架构：引入 Caffeine 本地缓存作为第一道防线，减少 Redis 穿透压力。
    3.  高可用与降级：采用 Redis Sentinel 或 Cluster 集群；对非核心业务添加限流、熔断降级策略。

## 3.3 缓存击穿 (Cache Breakdown)
*   现象：某个极高并发访问的热点 Key（如热点商品、秒杀活动）突然过期，瞬间所有并发请求同时去查库重建缓存，打爆数据库。
*   解决方案：
  
### 1.  互斥锁 (Mutex Lock)：

*   未命中缓存时，只有一个线程获取到分布式锁（如 setnx）去查库并重建缓存；其他线程休眠重试或自旋等待。
*   优缺点：数据强一致，但并发性能较低，且有死锁风险（需设置锁超时时间）。
         
```
    @Service
    public class CacheMutexService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    public Product getShopWithMutex(Long id) {
        String cacheKey = "shop:info:" + id;
        String lockKey = "lock:shop:" + id;

        // 1. 查询缓存
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            return JSONUtil.toBean(json, Product.class);
        }

        // 2. 尝试获取互斥锁 (setnx + 超时时间，防止死锁)
        Boolean isLock = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, "1", 10, TimeUnit.MINUTES);

        // 3. 锁获取失败，线程休眠后递归重试（或自旋等待）
        if (isLock == null || !isLock) {
            try {
                Thread.sleep(50); // 短暂休眠，减少CPU消耗
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return getShopWithMutex(id); // 重新去查缓存
        }

        // 4. 获取锁成功，双重检查缓存（防止重复查库）
        json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            // 释放锁并返回数据
            redisTemplate.delete(lockKey);
            return JSONUtil.toBean(json, Product.class);
        }

        // 5. 查数据库并重建缓存
        try {
            Product product = productMapper.selectById(id);
            if (product != null) {
                // 写入缓存，设置基础过期时间+随机偏移（防雪崩）
                String dataJson = JSONUtil.toJsonStr(product);
                redisTemplate.opsForValue().set(cacheKey, dataJson, 30, TimeUnit.MINUTES);
            }
            return product;
        } finally {
            // 6. 必须释放锁
            redisTemplate.delete(lockKey);
        }
    }
}
```
### 2.  逻辑过期 (Logical Expire)：

*   数据在 Redis 中不设物理 TTL（永不过期），但在 Value 内部封装一个 expireTime 字段作为逻辑过期时间。
*   查询时发现逻辑过期，由其中一个线程获取互斥锁，提交到异步线程池去后台重建缓存；其他线程直接返回旧数据。
*   优缺点：并发性能极高，不阻塞用户请求，但会短暂返回旧数据（最终一致性）。
         
#### 1. 封装带逻辑过期时间的数据结构
```
import lombok.Data;
import java.time.LocalDateTime;

@Data
public class RedisData {
    private Object data; // 实际数据
    private LocalDateTime expireTime; // 逻辑过期时间
}
```
#### 2. 写入逻辑过期缓存的通用方法（通常由定时任务或后台异步触发）
```
@Service
public class CacheLogicalExpireService {

    // 初始化异步线程池
    private final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void saveWithLogicalExpire(Long id, Long expireTimeSeconds) {
        RedisData redisData = new RedisData();
        // 模拟查库获取数据
        Product product = productMapper.selectById(id); 
        redisData.setData(product);
        // 设置逻辑过期时间（当前时间 + 指定过期时长）
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireTimeSeconds));

        // 写入缓存（不设置TTL，即永不过期）
        redisTemplate.opsForValue()
                .set("shop:info:" + id, JSONUtil.toJsonStr(redisData));
    }
}
```

#### 3. 读取逻辑过期缓存的通用模版
```
    public Product getShopWithLogicalExpire(Long id) {
        String cacheKey = "shop:info:" + id;
        String lockKey = "lock:shop:" + id;

        // 1. 查询缓存
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json == null) {
            return null; // 第一次调用或缓存被清空，直接返回null
        }

        // 2. 反序列化为带逻辑时间的对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        Product product = (Product) redisData.getData();
        LocalDateTime expireTime = redisData.getExpireTime();

        // 3. 判断逻辑过期时间是否到期
        if (expireTime.isAfter(LocalDateTime.now())) {
            // 未过期，直接返回旧数据
            return product;
        }

        // 4. 已过期，尝试获取互斥锁
        Boolean isLock = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, "1", 10, TimeUnit.MINUTES);

        // 5. 获取锁成功 -> 开启独立线程异步重建缓存
        if (isLock != null && isLock) {
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    this.saveWithLogicalExpire(id, 1800L); // 重新写入缓存，逻辑过期30分钟
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    redisTemplate.delete(lockKey); // 释放锁
                }
            });
        }

        // 6. 无论是否抢到锁，都直接返回当前的旧数据
        return product;
    }
```
| 缓存问题 | 核心特征 | 推荐解决方案 | 适用场景 |
| :---- | :--- | :--- | :--- |
| **穿透** | 请求不存在的数据 | 缓存空对象 / 布隆过滤器 | 恶意攻击多、不存在的Key较多时 |
| **雪崩** | 大量Key同时过期 | TTL加随机值 / 高可用集群 | 系统基础防护，必须做 |
| **击穿** | 单个热点Key过期 | 互斥锁 / 逻辑过期 | 互斥锁适合数据一致性要求高的场景；逻辑过期适合高并发、允许短暂旧数据的热点场景 |
# 缓存工具的封装
```
@Component
public class RedisCache {
    private StringRedisTemplate stringRedisTemplate;
    // 引入线程池，用于处理逻辑过期中的异步缓存重建
    private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    public RedisCache(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    /**
     * 设置缓存（支持 TTL 随机化，防止缓存雪崩）
     */
    public void set(String key, Object value, Long time, TimeUnit unit) {
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
    }

    public void set(String key, Object value, Long time, TimeUnit unit, boolean isRandomTTL) {
        if (isRandomTTL) {
            // 增加 0~20% 的随机时间
            long max = time * 12 / 10;
            long randomTime = ThreadLocalRandom.current().nextLong(time, max);
            set(key, value, randomTime, unit);
        } else {
            set(key, value, time, unit);
        }
    }

    /**
     * 方案一：解决缓存击穿 -> 互斥锁策略
     * @param key 缓存 Key
     * @param valueClass 反序列化的目标类型
     * @param dbFallback 缓存未命中时，去数据库查询的回调逻辑
     * @param time 缓存过期时间
     * @param unit 时间单位
     */
    public <R, ID> R queryWithMutex(String key, ID id, Class<R> valueClass, 
                                     Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        // 1. 查询缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        if (json != null) {
            return JSONUtil.toBean(json, valueClass);
        }
        // 如果命中缓存空对象，直接返回 null
        if (json != null && !"".equals(json)) {
            return null; 
        }

        // 2. 获取互斥锁
        String lockKey = "lock:" + key;
        R result = null;
        try {
            boolean isLock = tryLock(lockKey);
            if (!isLock) {
                // 3. 获取锁失败，休眠重试
                Thread.sleep(50);
                return queryWithMutex(key, id, valueClass, dbFallback, time, unit);
            }
            // 4. 获取锁成功，查询数据库
            result = dbFallback.apply(id);
            // 5. 数据库查不到，写入空对象并设置短 TTL 防止穿透
            if (result == null) {
                stringRedisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
                return null;
            }
            // 6. 查询成功，写入 Redis (带随机TTL防雪崩)
            this.set(key, result, time, unit, true);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            // 7. 释放锁
            unlock(lockKey);
        }
        return result;
    }

    /**
     * 方案二：解决缓存击穿 -> 逻辑过期策略
     * @param idFunc 用于从实体类中提取 ID 的方法
     */
    public <R, ID> R queryWithLogicalExpire(String key, ID id, Class<R> valueClass, 
                                             Function<ID, R> dbFallback, 
                                             Function<R, ID> idFunc, Long time, TimeUnit unit) {
        // 1. 查询缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        if (json == null) {
            return null;
        }

        // 2. 命中，反序列化（使用 RedisData 包装类）
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        JSONObject data = (JSONObject) redisData.getData();
        R r = JSONUtil.toBean(data, valueClass);
        LocalDateTime expireTime = redisData.getExpireTime();

        // 3. 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())) {
            // 未过期，直接返回
            return r;
        }

        // 4. 已过期，尝试获取互斥锁
        String lockKey = "lock:" + key;
        boolean isLock = tryLock(lockKey);
        if (isLock) {
            // 5. 获取锁成功，开启独立线程异步重建缓存
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    // 5.1 查数据库
                    R dbResult = dbFallback.apply(id);
                    // 5.2 写入 Redis（设置逻辑过期时间）
                    this.setWithLogicalExpire(key, dbResult, time, unit);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    unlock(lockKey);
                }
            });
        }
        // 6. 无论是否抢到锁，都直接返回当前的旧数据
        return r;
    }

    /**
     * 写入逻辑过期数据（物理上不设 TTL，在 Value 内封装 expireTime）
     */
    private <R> void setWithLogicalExpire(String key, R value, Long time, TimeUnit unit) {
        RedisData redisData = new RedisData();
        redisData.setData(JSONUtil.toJsonStr(value));
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
    }

    // --- 互斥锁基础操作 ---
    private boolean tryLock(String key) {
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.MINUTES);
        return Boolean.TRUE.equals(flag);
    }

    private void unlock(String key) {
        stringRedisTemplate.delete(key);
    }
}
```
## 逻辑过期实体类
```
@Data
public class RedisData {
    // 逻辑过期时间
    private LocalDateTime expireTime;
    // 实际缓存的业务数据
    private Object data;
}
```
# 全局 ID 生成器

## 一、什么是全局 ID 生成器
在分布式系统中，单机数据库的自增主键无法跨库保证唯一性。因此，需要一个全局唯一 ID 生成器，为所有节点的数据分配不重复、可用的 ID。

## 二、核心要求（五大特性）
| 特性 | 说明 |
| :--- | :--- |
| **唯一性** | 全局唯一，任何情况下都不能重复（最基础要求） |
| **高可用** | 生成服务不能单点故障，需支持故障切换，可用性尽量接近 100% |
| **高性能** | 生成速度快、延迟低，能支撑高并发（QPS 越高越好） |
| **递增性** | 便于数据库索引维护（如 B+ 树）、分页排序；通常要求趋势递增 |
| **安全性** | ID 不可被轻易猜测，避免暴露业务量/订单量等敏感信息 |



## 三、雪花算法（Snowflake）核心结构
*原笔记给出的位分配（简化版）：*
`符号位 1bit + 时间戳 31bit + 序列号 32bit`

- **符号位（1bit）**：恒为 0，保证 ID 为正数。
- **时间戳（31bit）**：记录生成时刻，保证随时间趋势递增。
- **序列号（32bit）**：同一毫秒内通过自增序列区分多个 ID，保证同毫秒内不重复。

> **补充：**
> 业界标准 Snowflake 通常为 `1bit 符号位 + 41bit 时间戳 + 10bit 机器ID + 12bit 序列号`。时间戳保证趋势递增，机器 ID 区分不同节点，序列号应对同毫秒并发。不同实现可调整各段位数，原笔记为其中一种分配方式（秒级时间戳 + 32bit 序列号）。

## 四、常见实现方案对比
| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **UUID** | 128bit 随机/基于 MAC、时间生成 | 本地生成、无需网络、绝对唯一 | 无序（不递增）、过长（占存储）、无业务含义 | 不要求排序/索引友好的场景 |
| **数据库自增** | 利用数据库 `auto_increment`，可多库步长错位 | 简单、递增、有序 | 依赖数据库、并发瓶颈、性能有限 | 中小规模、单库或分库步长方案 |
| **Redis 自增 ID 策略** | 利用 `INCR` / `INCRBY` 原子自增 | 性能高、递增、实现简单 | 依赖 Redis 可用性；ID 可被猜测（安全性弱）；需考虑持久化/集群一致性 | 中高并发、可接受 Redis 依赖的场景 |
| **Snowflake 算法** | 位拼接：时间戳 + 机器 ID + 序列号 | 高性能（本地生成无网络开销）、趋势递增、有序 | 依赖时钟（时钟回拨会产生重复风险）、需设计机器 ID 分配 | 高并发分布式系统（主流方案） |

## 五、Redis 实现示例（本笔记采用方案）

```java
public long nextId(String keyPrefix) {
    // 1. 生成时间戳
    LocalDateTime now = LocalDateTime.now();
    long timestamp = now.toEpochSecond(ZoneOffset.UTC) - BEGIN_TIMESTAMP;
    
    // 2. 生成序列号
    String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
    long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
    
    // 3. 拼接并返回
    return timestamp << COUNT_BITS | count;
}
```
实现要点：

- 1. 时间戳（高 31/32 位）：取当前秒数与固定起始时间 BEGIN_TIMESTAMP 的差值，保证 ID 随时间趋势递增。
- 2. 序列号（低 32 位）：用 Redis 的 INCR 原子自增，key 为 icr + 业务前缀 + 当天日期，按天重置计数，同一业务同一天内从 0 开始自增。
- 3. 拼接方式：timestamp << COUNT_BITS | count，时间戳左移 32 位、低 32 位放序列号，组成一个 long 型（64 位）全局 ID。
- 4. 位分配：与第三节简化版 Snowflake 一致。符号位 1bit + 时间戳 31bit + 序列号 32bit，即符号位隐含在 long 的正数范围内，序列号靠 Redis 原子性保证同秒内不重复。
# 乐观锁和悲观锁
## 悲观锁

**悲观锁：**

 悲观锁可以实现对于数据的串行化执行，比如syn，和lock都是悲观锁的代表，同时，悲观锁中又可以再细分为公平锁，非公平锁，可重入锁，等等

**乐观锁：**

  乐观锁：会有一个版本号，每次操作数据会对版本号+1，再提交回数据时，会去校验是否比之前的版本大1 ，如果大1 ，则进行操作成功，这套机制的核心逻辑在于，如果在操作过程中，版本号只比原来大1 ，那么就意味着操作过程中没有人对他进行过修改，他的操作就是安全的，如果不大1，则数据被修改过，当然乐观锁还有一些变种的处理方式比如cas

  乐观锁的典型代表：就是cas，利用cas进行无锁化机制加锁，var5 是操作前读取的内存值，while中的var1+var2 是预估值，如果预估值 == 内存值，则代表中间没有被人修改过，此时就将新值去替换 内存值

  其中do while 是为了在操作失败时，再次进行自旋操作，即把之前的逻辑再操作一次。

```
int var5;
do {
    var5 = this.getIntVolatile(var1, var2);
} while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

return var5;
```
# 分布式锁-Redisson
## 入门
### 1.引入依赖
```
<dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.42.0</version>
</dependency>
```
### 2.配置Redisson
```
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient(){
        // 配置
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.150.101:6379")
            .setPassword("123321");
        // 创建RedissonClient对象
        return Redisson.create(config);
    }
}
```
### 3.使用Redisson
```
@Resource
private RedissionClient redissonClient;

@Test
void testRedisson() throws Exception{
    //获取锁(可重入)，指定锁的名称
    RLock lock = redissonClient.getLock("anyLock");
    //尝试获取锁，参数分别是：获取锁的最大等待时间(期间会重试)，锁自动释放时间，时间单位
    boolean isLock = lock.tryLock(1,10,TimeUnit.SECONDS);
    //判断获取锁成功
    if(isLock){
        try{
            System.out.println("执行业务");          
        }finally{
            //释放锁
            lock.unlock();
        }   
    }  
}
```
## 可重入锁原理
可重入锁就是为了防止“同一个线程，在持有锁的情况下，因为内部调用还需要这把锁，从而导致自己被卡死”的情况。
### 1. Redis 的数据结构：Hash（哈希表）

在单机 JUC 锁中， state  变量存在 JVM 内存里；但在分布式锁中，这个状态存在 Redis 里。Redisson 采用的是 Hash 结构：

大 Key (KEYS[1])：就是你给锁起的名字（例如  my_lock ）。它的存在与否，决定了这把锁当前有没有人持有。

小 Key (ARGV[2])：格式为  客户端UUID + ":" + 线程ID （例如  88888888-xxxx-1 ）。它唯一标识了是哪台机器、哪个线程加了这把锁。

Value：一个整数，代表重入的次数。


### 2. Lua 脚本的三段式逻辑解析

这段脚本在 Redis 内部是原子执行的，它包含了加锁、重入、拒锁三个完整逻辑：

**KEYS[1] ： 锁名称**

**ARGV[1]：  锁失效时间**

**ARGV[2]：  id + ":" + threadId; 锁的小key**


🟢 场景一：锁不存在（成功加锁）
```
if (redis.call('exists', KEYS[1]) == 0) then 
    redis.call('hset', KEYS[1], ARGV[2], 1);      // 创建Hash，重入次数设为1
    redis.call('pexpire', KEYS[1], ARGV[1]);       // 设置全局过期时间（防止死锁）
    return nil;                                    // 返回null，表示加锁成功
end;
```
如果 Redis 里根本没有这个大 Key，说明锁是空的，直接加锁成功。

🔵 场景二：锁已存在，且是同一个线程重入（重入成功）
```
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
    redis.call('hincrby', KEYS[1], ARGV[2], 1);    // 重入次数 +1
    redis.call('pexpire', KEYS[1], ARGV[1]);       // 重新刷新过期时间（看门狗机制的延伸）
    return nil;                                    // 返回null，表示重入成功
end;
```
如果锁被占用了，但占用者的小 Key 和当前线程完全一致，说明是自己人，允许重入，并把计数器 +1。

🔴 场景三：锁已存在，且不是当前线程（加锁失败）
```
return redis.call('pttl', KEYS[1]); // 返回锁的剩余过期时间（毫秒）
```
如果以上两个条件都不满足，说明这把锁被别人占着。Redisson 框架层接收到这个非空的返回值（剩余时间）后，就会在 Java 端发起  while(true)  自旋或阻塞等待，直到对方释放锁。

