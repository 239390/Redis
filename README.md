# 数据结构
# 持久化
# 过期删除与内存淘汰
# 主从/哨兵/集群
# 缓存穿透/雪崩/击穿
# 缓存一致性
# 分布式锁

# 二、缓存更新策略
## 1.旁路缓存读写流程
读：

- 先读缓存，命中则返回。

- 未命中则读 DB，并将结果写入缓存，再返回。

写：

- 常见策略：先更 DB，再删缓存（推荐）；
- 或先删缓存，再更 DB（有并发问题）；
- 或缓存更新（不推荐，写多读少时效率低）。
## 2.为什么先更DB再删缓存
保证DB是最新的，删缓存后，下次读会加载最新的DB值到缓存
# 三、缓存三大问题
## 穿透/雪崩/击穿的现象+方案+对比
| 问题 | 现象 | 核心区别 | 解决方案 |
|---|---|---|---|
| 穿透 | 查不存在的数据，缓存和 DB 都没有，请求直达DB | 数据本身不存在 | 布隆过滤器、缓存空值 |
| 击穿 | 单个热点 Key 过期，大量请求打到 DB | 一个热点 Key 失效 | 互斥锁、逻辑过期 |
| 雪崩 | 大量 Key 同时过期，或 Redis 宕机 | 一片 Key 失效或缓存整体故障 | 随机TTL、高可用Redis(集群、主从)、限流降级 |
# 四、全局ID
## 1.Snowflake位结构
Snowflake（雪花算法）生成 64 位 ID：

- 1 位：符号位（0）。

- 41 位：时间戳（毫秒级，约 69 年）。

- 10 位：机器 ID（5 位数据中心 + 5 位工作机 ID，共 1024 个节点）。

- 12 位：序列号（每毫秒 4096 个 ID）。
## 2.UUID/DB自增/Redis自增/Snowflake对比表
| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| UUID | 本地生成、不依赖中心 | 无序、长、索引差、页分裂 | 非主键追踪 ID |
| DB 自增 | 简单、有序、短 | 扩展差、依赖 DB、分库分表难 | 单机、小流量 |
| Redis 自增 | 高性能、有序 | 依赖 Redis、持久化风险 | 中小规模分布式 |
| Snowflake | 有序、高性能、分布式 | 依赖时钟、需机器 ID 管理 | 大规模分布式系统 |
# 五、锁机制
## 乐观锁和悲观锁区别
- 悲观锁：假设冲突概率高，每次操作都加锁（如  synchronized 、 ReentrantLock 、DB 行锁）。

  - 适用：写多读少，冲突频繁。

- 乐观锁：假设冲突概率低，不加锁，更新时判断版本号（ version ）或 CAS。

  - 适用：读多写少，冲突少。

- 区别：

  - 悲观锁阻塞，乐观锁自旋/重试。

  - 悲观锁保证安全，但性能可能低；乐观锁性能高，但冲突时需重试。
# 六、Redisson
## 1.可重入锁Hash结构
Redisson 的  RLock  基于 Hash：

- Key：锁名。

- Field：线程 ID。

- Value：重入次数（计数器）。

- 加锁：线程首次加锁，Hash 中新增 field，value=1；再次加锁，value++。

- 解锁：value--，为 0 时删除 field。
## 2.Lua三段式
加锁 Lua 逻辑大致为：

- 1. 判断锁是否存在。
- 2. 不存在则加锁并设置过期时间。
- 3. 存在且是当前线程，则重入次数加 1。
- 4. 否则返回锁剩余过期时间。

解锁 Lua 逻辑大致为：

- 1. 判断锁是否属于当前线程。
- 2. 是则重入次数减 1。
- 3. 减到 0 删除锁，并发布解锁通知。
## 3.WatchDog续约
默认锁超时 30 秒，看门狗每 10 秒（ lockWatchdogTimeout / 3 ）检查一次。

若线程仍持有锁，续约 TTL 为 30 秒，防止业务未结束锁已过期。
## 4.MultiLock
MultiLock 是把多个独立锁组合成一个联合锁。

- 所有锁全部加锁成功，才算整体加锁成功。
- 任意一个失败，则释放已成功的锁。
- 适合多资源同时加锁场景。
# 七、高可用
## 1.RDB vs AOF对比
| | RDB | AOF |
| :--- | :--- | :--- |
| 持久化方式 | 定时对整体内存做快照 | 记录每一次执行的命令 |
| 数据完整性 | 不完整，两次备份之间会丢失 | 相对完整，取决于刷盘策略 |
| 文件大小 | 会有压缩，文件体积小 | 记录命令，文件体积很大 |
| 宕机恢复速度 | 很快 | 慢 |
| 数据恢复优先级 | 低，因为数据完整性不如AOF | 高，因为数据完整性更高 |
| 系统资源占用 | 高，大量CPU和内存消耗 | 低，主要是磁盘IO资源<br>但AOF重写时会占用大量CPU和内存资源 |
| 使用场景 | 可以容忍数分钟的数据丢失，追求更快的启动速度 | 对数据安全性要求较高常见 |
## 2.哨兵选举流程
一旦发现master故障,sentinel需要在salve中选择一个作为新的master,选择依据:
- 首先会判断slave节点与master节点断开时间长短，如果超过指定值（down-after-milliseconds * 10）则会排除该slave节点

- 然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举

- 如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高

- 最后是判断slave节点的运行id大小，越小优先级越高。
## 3.分片集群插槽
Redis Cluster 有 16384 个槽（ 0-16383 ）。

每个节点负责部分槽，数据哈希到槽（ CRC16(key) % 16384 ）。

增删节点时，通过槽迁移（ MIGRATE ）平衡负载，无需停机。

客户端根据槽定位节点（ GETSLOT ），或接收重定向（ MOVED  错误）。
# 主从集群
## 主从同步原理
### 1. 核心概念
- **Replication Id（replid）**：数据集标识。master 有唯一 replid，slave 会继承 master 的 replid；replid 一致表示同一数据集。
- **offset**：同步偏移量，记录在 `repl_baklog` 中。slave 记录已同步 offset；若小于 master offset，说明数据落后。
- slave 同步时需向 master 声明自己的 **replid 和 offset**，master 据此判断同步方式。

### 2. 全量同步
master 生成完整 RDB 发给 slave，后续命令通过 repl_baklog 持续发送。
**触发场景：**
- slave 第一次连接 master；
- slave 断开太久，`repl_baklog` 中对应 offset 已被覆盖。

**判断依据：**
- slave 发送的 replid 与 master 不一致，说明是全新 slave，需要全量同步。

**流程：**
1. slave 请求增量同步；
2. master 判断 replid 不一致，拒绝增量同步；
3. master 生成完整内存数据的 RDB 并发送给 slave；
4. slave 清空本地数据并加载 RDB；
5. master 将 RDB 期间的命令记录到 `repl_baklog`，并持续发送给 slave；
6. slave 执行命令，保持同步。
7. master 将自己的 replid 和 offset 发给 slave，slave 保存；此后 replid 与 master 一致。

### 3. 增量同步
slave 提交 offset，master 只发送 repl_baklog 中 offset 之后的命令。
- 只同步 slave 与 master 存在差异的部分数据。
- slave 提交自己的 offset，master 从 `repl_baklog` 中获取该 offset 之后的命令发送给 slave。
- **触发场景**：slave 断开后恢复，且其 offset 仍能在 `repl_baklog` 中找到。

### 4. repl_baklog 原理
- 固定大小的环形数组，写满后覆盖最早数据。
- 记录 Redis 处理过的命令及 offset，包括 master 当前 offset 和 slave 已拷贝 offset。
- 两者差异即需要增量同步的数据。
- 若 slave 断开过久，未同步数据被覆盖，无法增量同步，只能全量同步。

### 5. 主从同步优化
- 配置 `repl-diskless-sync yes`，启用无磁盘复制，减少全量同步磁盘 IO；
- 控制单节点内存占用，减少 RDB 导致的磁盘 IO；
- 适当提高 `repl_baklog` 大小，并尽快恢复故障 slave，尽量避免全量同步；
- 限制单个 master 的 slave 数量；slave 太多时采用 **主-从-从链式结构**，减轻 master 压力。

# 哨兵
## 作用
哨兵的作用如下：
- 状态监控：Sentinel 会不断检查您的master和slave是否按预期工作
- 故障恢复（failover）：如果master故障，Sentinel会将一个slave提升为master。当故障实例恢复后会成为slave
- 状态通知：Sentinel充当Redis客户端的服务发现来源，当集群发生failover时，会将最新集群信息推送给Redis的客户端
## 状态监控
Sentinel基于心跳机制监测服务状态，每隔1秒向集群的每个节点发送ping命令，并通过实例的响应结果来做出判断：
- 主观下线（sdown）：如果某sentinel节点发现某Redis节点未在规定时间响应，则认为该节点主观下线。
- 客观下线(odown)：若超过指定数量（通过quorum设置）的sentinel都认为该节点主观下线，则该节点客观下线。quorum值最好超过Sentinel节点数量的一半，Sentinel节点数量至少3台。
### 故障处理
#### 故障转移步骤
- 首先要在sentinel中选出一个leader，由leader执行failover​
- 选定一个slave作为新的master，执行slaveof noone，切换到master模式​
- 然后让所有节点都执行slaveof 新master​
- 修改故障节点配置，添加slaveof 新master

#### sentinel从slave中选取master的依据
- 首先会判断slave节点与master节点断开时间长短，如果超过down-after-milliseconds * 10则会排除该slave节点
- 然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举（默认都是1）。
- 如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高
- 最后是判断slave节点的run_id大小，越小优先级越高（通过info server可以查看run_id）。

### 身份切换
- 在多个sentinel中选举一个leader
- 由leader执行failover
## 选举leader
首先，Sentinel集群要选出一个执行failover的Sentinel节点，可以成为leader。要成为leader要满足三个条件：
- 最先获得超过半数的投票
- 获得的投票数不小于quorum值
- 一般情况下最先发起failover的节点会当选
而sentinel投票的原则有两条：
- 优先投票给目前得票最多的
- 如果目前没有任何节点的票，就投给自己

## failover
- sentinel给备选的slave1节点发送slaveof no one命令，让该节点成为master
- sentinel给所有其它slave发送slaveof 192.168.150.101 7002 命令，让这些节点成为新master，也就是7002的slave节点，开始从新的master上同步数据。
- 最后，当故障节点恢复后会接收到哨兵信号，执行slaveof 192.168.150.101 7002命令，成为slave：
# 分片集群
## 集群配置
### 1. 集群模式必需参数
分片集群中的 Redis 节点必须开启集群模式，配置文件中常用参数：

```conf
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

关键参数说明：
- `cluster-enabled yes`：开启集群模式；
- `cluster-config-file`：集群配置文件名称，无需手动创建，由集群自动维护；
- `cluster-node-timeout`：节点之间心跳超时时间；
- `appendonly yes`：开启 AOF 持久化（示例配置中包含）。

### 2. Docker Compose 部署集群
在 `/root/redis-cluster` 下新建 `docker-compose.yaml`，启动 6 个 Redis 节点：
- 端口分别为 `7001` ~ `7006`；
- 每个节点启动参数包含：
  - `--port`
  - `--cluster-enabled yes`
  - `--cluster-config-file node.conf`
- **必须使用 `network_mode: "host"`**，否则 Docker 部署 Redis 集群会有网络问题。

启动命令：

```bash
docker-compose up -d
```

查看进程：

```bash
ps -ef | grep redis
```

每个节点应以 `[cluster]` 模式运行，但此时节点之间尚未建立连接。

### 3. 创建集群
进入任意节点容器：

```bash
docker exec -it r1 bash
```

执行创建集群命令：

```bash
redis-cli --cluster create --cluster-replicas 1 \
192.168.150.101:7001 192.168.150.101:7002 192.168.150.101:7003 \
192.168.150.101:7004 192.168.150.101:7005 192.168.150.101:7006
```

命令说明：
- `redis-cli --cluster`：集群操作命令；
- `create`：创建集群；
- `--cluster-replicas 1`：每个 master 的副本数为 1。

### 4. master/slave 分配规则
节点总数 ÷ `(replicas + 1)` = master 数量 `n`。

### 5. 查看集群状态
命令：

```bash
redis-cli -p 7001 cluster nodes
```

用于查看集群节点、master/slave 角色及分配关系。

### 6. 核心注意点
- 集群节点必须开启 `cluster-enabled yes`；
- Docker 部署必须使用 host 网络模式；
- `cluster-config-file` 由集群自动维护，无需手动创建；
- 创建集群时通过 `--cluster-replicas 1` 指定每个 master 的副本数；
- 前 n 个节点为 master，其余为 slave；
- 创建前节点只是以 cluster 模式运行，尚未建立连接，需执行 `redis-cli --cluster create` 完成组网。
