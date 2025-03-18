# Redis 基础知识

## Redis的工作模式？主要特点？
[高可用](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#%E9%AB%98%E5%8F%AF%E7%94%A8)
## 基本数据类型及其使用场景
[3.Redis 有哪些数据类型？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#3redis-%E6%9C%89%E5%93%AA%E4%BA%9B%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B)
## Redis的IO复用模型？使用哪种机制来实现（epoll）
[5.能说一下 I/O 多路复用吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#5%E8%83%BD%E8%AF%B4%E4%B8%80%E4%B8%8B-io-%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E5%90%97)
# Redis 数据结构与实现原理
[49.说说 Redis 底层数据结构？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#49%E8%AF%B4%E8%AF%B4-redis-%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)
## Redis的ZSet底层结构？
Redis的ZSet（有序集合）底层结构采用跳跃表（SkipList）与哈希表（Hash Table）的结合，具体实现如下：
- 哈希表（Hash Table）​
    - 用于存储成员（Member）到分数（Score）的映射关系，支持O(1)时间复杂度的成员查找。例如，成员"Tom"的分数为95，哈希表会以"Tom"为键，95为值存储。
    - 实现机制：键为成员（Member），值为分数（Score）
- 跳跃表（SkipList）​
    - 用于维护成员的有序性，通过多层链表结构实现O(log N)时间复杂度的范围查询、插入和删除操作。每个节点包含成员、分数及指向其他节点的指针，层数随机生成以平衡性能与内存。
    - 结构特性：
        - 跳表由zskiplist和zskiplistNode组成，zskiplist用于存储跳表基本信息，zskiplistNode用于存储节点信息；
        - 跳表的每个节点包含多个指针，指向不同层级的节点，支持快速跳跃查询； 
        - 每个节点包含成员（Member）、分数（Score）和多层指针（层级随机生成以平衡性能与内存）。
        - 跳跃表通过多级索引实现快速跳转，类似二分查找逻辑。

动态选择结构：压缩列表（ziplist）与ListPack
- Redis 7.0之前：
    - 小于128个元素且单元素大小小于等于64字节时，使用压缩列表（ziplist）存储。
    - 大于128个元素时，使用跳跃表（SkipList）存储。

## 如果在ZSet中添加一个元素，过程是什么样的？
1. ​判断成员是否存在
    - 若成员已存在：
        - ​更新分数：哈希表修改分数映射，跳跃表删除旧节点并重新插入新分数对应的节点。
        - ​原子性保证：通过 ZADD 命令的 XX（仅更新）或 NX（仅新增）选项控制行为。
2. ​生成随机层数（仅跳跃表）​
    - 跳跃表节点的层数由随机算法生成（如 Redis 源码中的 zslRandomLevel），概率遵循指数分布（50% Level1, 25% Level2 等），最大层数限制为32。
    - ​目的：避免频繁调整索引结构，平衡查询效率与插入性能。
3. ​插入元素到数据结构
    - ​ListPack/ziplist：
        - 遍历找到插入位置（O(N)复杂度）。
        - 将成员和分数插入到有序位置，若超出阈值则触发结构转换。
    - ​跳跃表+哈希表：
        - ​哈希表：添加/更新成员到分数的映射。
        - ​跳跃表：从最高层开始逐层查找插入位置，创建新节点并建立前后指针。
4. ​排序与索引维护
    - 成员按分数升序排列，分数相同则按字典序排序。
    - 跳跃表通过多级索引快速定位插入点，减少遍历次数。

# Redis 持久化

## Redis的持久化方式？AOF和RDB的区别与实现原理？
[10.Redis 持久化⽅式有哪些？有什么区别？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#10redis-%E6%8C%81%E4%B9%85%E5%8C%96%E5%BC%8F%E6%9C%89%E5%93%AA%E4%BA%9B%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB)

# Redis 缓存设计与优化

## Redis和MySQL的数据一致性问题？如何保证缓存与数据库的一致性？旁路缓存模式有什么问题？
[31.如何保证缓存和数据库的数据⼀致性？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#31%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E7%BC%93%E5%AD%98%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9A%84%E6%95%B0%E6%8D%AE%E8%87%B4%E6%80%A7)
旁路缓存模式的问题：
- 会有短暂的不一致问题（缓存删除失败或并发竞争）
- 频繁写入场景下，缓存穿透问题严重
## Redis的淘汰策略？当内存不足时如何选择被淘汰的键？
[39.Redis 有哪些内存淘汰策略？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#39redis-%E6%9C%89%E5%93%AA%E4%BA%9B%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5)
## 缓存减扣命令具体是什么？如何用Redis实现计数器和库存扣减？
1. Redis 缓存减扣命令
    Redis 提供以下核心命令用于数值减扣操作，适用于计数器或库存扣减场景：
    - `DECR`：将键的值减1，若键不存在则初始化为0。
    - `DECRBY`：将键的值减去指定的整数值，若键不存在则初始化为0。
    - `HINCRBY`：对哈希表中指定字段的值进行减扣操作，若字段不存在则初始化为0。
    - `ZINCRBY`：对有序集合中指定成员的分数进行减扣操作，若成员不存在则初始化为0。
2. Redis 实现计数器
    - 初始化与递增：
        - 使用 `INCR` 或 `INCRBY` 命令初始化计数器，设置初始值为0。
        - 每次操作时调用 `INCR` 或 `INCRBY` 命令进行递增。
    - 读取计数器：
        - 使用 `GET` 命令获取当前计数器值。
        - 注意：计数器值为字符串类型，需转换为整数进行计算。
    - 递减与重置：
        - 使用 `DECR` 或 `DECRBY` 命令进行递减操作。
        - 使用 `SET` 命令将计数器重置为指定值。
3. Redis 实现库存扣减
    - 初始化库存：
        - 使用 `SET` 命令设置初始库存值。
    - 扣减库存：
        - 使用 `DECR` 或 `DECRBY` 命令进行库存扣减操作。
        - 注意：需确保库存值不小于0，避免超卖。
    - 读取库存：
        - 使用 `GET` 命令获取当前库存值。
        - 注意：库存值为字符串类型，需转换为整数进行计算。
    - 处理超卖：
        - 使用 `WATCH` 命令监视库存键，确保在扣减操作前检查库存值。
            - 若库存值小于0，则拒绝扣减操作并返回错误信息。
            - 若库存值大于等于0，则执行扣减操作。
        - 使用 `MULTI` 和 `EXEC` 命令实现事务处理，确保操作的原子性。 
## 限流算法，漏斗算法，令牌桶算法的区别及实现方式？
| 特性           | 漏斗算法（漏桶算法）                         | 令牌桶算法                                     |
| -------------- | -------------------------------------------- | ---------------------------------------------- |
| 流量控制逻辑   | 固定速率处理请求（类似恒定流速的漏水）       | 按固定速率生成令牌，允许突发流量消耗累积令牌   |
| 核心参数       | 容量（桶大小）、漏水速率（处理速率）         | 容量（最大令牌数）、令牌生成速率               |
| 突发流量处理   | 不支持，请求速率严格受限于漏水速率           | 支持，可一次性消耗累积的令牌应对突发请求       |
| 适用场景       | 保护下游系统（如数据库），需严格控制处理速率 | 允许合理突发（如秒杀场景），需弹性应对流量波动 |
| 典型实现复杂度 | 中（需维护漏水和请求记录）                   | 高（需动态生成令牌并处理并发竞争）             |

1. 漏斗算法
   - 请求处理：请求像水流一样进入漏桶，桶以固定速率漏水，处理请求。若桶满，则丢弃请求。
   - 流量整形：控制请求速率，避免突发流量冲击下游系统。
   - 实现方式：使用定时器定期处理请求，维护请求队列和漏水速率。
2. 令牌桶算法
   - 令牌生成：以固定速率生成令牌，存入桶中。请求需消耗令牌才能处理。
   - 突发流量：允许突发流量，令牌可累积，支持短时间内大量请求。
   - 实现方式：使用定时器生成令牌，维护令牌数量和请求队列。

## Redis的限流具体实现？

## 缓存击穿、缓存穿透、缓存雪崩的区别？各自解决方案？
[29.缓存击穿、缓存穿透、缓存雪崩了解吗](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#29%E7%BC%93%E5%AD%98%E5%87%BB%E7%A9%BF%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F%E7%BC%93%E5%AD%98%E9%9B%AA%E5%B4%A9%E4%BA%86%E8%A7%A3%E5%90%97)
## 使用空对象解决了缓存穿透问题，如果此时在数据库中新增了该空对象，也就是说它现在不是一个空对象了，这个时候怎么处理？
1. 主动删除空对象缓存
   - 数据库变更触发缓存删除
   - 监听数据库变更日志，主动删除空对象缓存
2. 异步更新缓存
   - 消息队列异步更新缓存
3. 设置较短的过期时间
   - 设置空对象缓存的过期时间，定期清理过期缓存
## 热key问题的识别与解决方案？
[33.怎么处理热 key？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#33%E6%80%8E%E4%B9%88%E5%A4%84%E7%90%86%E7%83%AD-key)
[35.热点 key 重建？问题？解决？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#35%E7%83%AD%E7%82%B9-key-%E9%87%8D%E5%BB%BA%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3)
## 大key问题的危害与解决方案
[41.大 key 问题了解吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#41%E5%A4%A7-key-%E9%97%AE%E9%A2%98%E4%BA%86%E8%A7%A3%E5%90%97)
## 超卖问题的解决方案？如何保证库存操作的原子性和数据一致性？
1. 在数据库层面
    - 查询商品库存时加排他锁，确保同一时间只有一个线程能操作库存。
2. 使用分布式锁
    - 使用 Redis 分布式锁（如 Redisson）来控制对库存的访问，确保同一时间只有一个线程能操作库存。
3. 使用分布式锁+分段缓存
    - 将库存分成多个段，每个段使用独立的锁进行控制，减少锁竞争。
4. 使用Redis的原子操作+异步队列
    - 系统初始化时，将库存信息加载到 Redis 中
    - 使用 Redis 的原子操作（如 DECR）来减少库存。若redis中库存不足，直接返回失败。
    - 将请求放入异步队列，异步处理库存扣减操作。
## MySQL和Redis和ElasticSearch的区别？各自适用场景？
| 维度       | MySQL                                                 | Redis                                        | Elasticsearch                                        |
| ---------- | ----------------------------------------------------- | -------------------------------------------- | ---------------------------------------------------- |
| 数据模型   | 关系型数据库（表结构严格，支持复杂关联查询）          | 键值存储（支持字符串、列表、哈希等数据结构） | 文档型数据库（JSON 格式，支持半结构化/非结构化数据） |
| 核心优势   | ACID 事务、强一致性、复杂 SQL 能力（如 JOIN、子查询） | 超高吞吐量（微秒级响应）、低延迟、内存操作   | 全文搜索、近实时分析、复杂聚合（如分词、模糊查询）   |
| 性能瓶颈   | 全文搜索性能差，单表数据量超千万级需分库分表          | 数据规模受限于内存容量（通常 GB 级）         | 写入吞吐量低于 Redis（需 1 秒刷新索引）              |
| 扩展性     | 垂直扩展（升级硬件）或手动分库分表                    | 主从复制或集群分片                           | 天然分布式，自动分片和水平扩展                       |
| 一致性模型 | 强一致性（事务隔离级别）                              | 最终一致性（集群模式下）                     | 最终一致性（近实时搜索）                             |


# Redis 分布式应用

## Redis分布式锁的实现原理及优缺点？
[48.Redis 实现分布式锁了解吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#48redis-%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%BA%86%E8%A7%A3%E5%90%97)
## 一致性哈希算法的原理？如何解决数据分布不均匀问题？
1. 一致性哈希的核心原理
    - 哈希环的构造：将哈希值空间组织成一个固定范围的虚拟圆环（通常为 $2^{32}$ 或 $2^{64}$），所有节点（如服务器）和数据（如缓存键）通过哈希函数映射到环上。例如，节点哈希值计算为 hash(IP+端口)，数据哈希值计算为 hash(key)
    - 数据定位规则：数据在环上的位置按顺时针方向查找，遇到的第一个节点即为归属节点。例如，数据哈希值为 k1，若环上顺时针最近的节点为 node2，则 k1 由 node2 处理
    - 动态扩缩容：
        - 新增节点：仅影响新节点逆时针方向至相邻节点之间的数据，需将这部分数据迁移至新节点。例如，新增节点 nodeX 位于 nodeA 和 nodeB 之间，仅迁移 nodeA 到 nodeX 区间的数据
        - 删除节点：失效节点的数据迁移至顺时针方向的下一个节点，其他数据不受影响。例如，节点 nodeC 宕机后，其数据由 nodeD 接管
2. 一致性哈希的优点
    - 动态扩缩容：新增或删除节点时，仅需迁移少量数据，避免全量迁移带来的性能损失
    - 数据均匀分布：通过哈希函数将数据均匀分布在节点上，避免热点问题
    - 高可用性：节点宕机后，数据可自动迁移至其他节点，保证系统的高可用性
    - 负载均衡：节点数量变化时，数据分布不受影响，避免了传统哈希算法的负载不均问题
3. 一致性哈希的应用场景
    - 分布式缓存：如 Memcached、Redis 集群等，使用一致性哈希算法将数据均匀分布在多个节点上
    - 分布式存储：如 Amazon DynamoDB、Google Bigtable 等，使用一致性哈希算法实现数据的高可用性和负载均衡
    - CDN 节点选择：根据用户请求的地理位置，将请求路由到最近的 CDN 节点，提高访问速度
    - Nginx 负载均衡：根据请求的 URL 或 IP 地址，将请求分发到不同的后端服务器，实现负载均衡
4. 数据分布不均匀问题和解决方案
    - 问题根源：当节点数量较少或哈希函数分布不理想时，节点在环上可能集中分布，导致数据倾斜（如大部分数据集中在某几个节点），形成“热点”。例如，仅有两个节点的哈希环可能使 80% 的数据落在其中一个节点上
    - 解决方案：
        - 虚拟节点：为每个物理节点创建多个虚拟节点，增加哈希环上的节点数量，平衡数据分布。例如，物理节点 nodeA 对应虚拟节点 nodeA1、nodeA2 等
        - 加权一致性哈希：为不同节点设置权重，权重越高的节点在哈希环上占据更多位置，适用于性能差异较大的节点
## 布隆过滤器的原理？应用场景？如何在Redis中实现？有什么缺陷？
[30.能说说布隆过滤器吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#30%E8%83%BD%E8%AF%B4%E8%AF%B4%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8%E5%90%97)
## Raft选举算法的原理？在Redis中的应用？
[10.说说 Raft 算法？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/fenbushi.md#10%E8%AF%B4%E8%AF%B4-raft-%E7%AE%97%E6%B3%95)
## Redis锁如何续期？Redis锁超时，任务没完怎么办？
[美团面试：Redis锁如何续期？Redis锁超时，任务没完怎么办？](https://mp.weixin.qq.com/s/h3QY545R5US_WkjQXl8mbw)

# Redis 集群与高可用

## Redis集群模式的类型及各自特点
[高可用](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#%E9%AB%98%E5%8F%AF%E7%94%A8)
## 集群模式下，选主规则及故障转移过程
[27.能说说 Redis 集群的原理吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#27%E8%83%BD%E8%AF%B4%E8%AF%B4-redis-%E9%9B%86%E7%BE%A4%E7%9A%84%E5%8E%9F%E7%90%86%E5%90%97)
## Redis集群怎么决定命中哪一个Redis节点？集群通信的基本流程
[26.集群中数据如何分区？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#26%E9%9B%86%E7%BE%A4%E4%B8%AD%E6%95%B0%E6%8D%AE%E5%A6%82%E4%BD%95%E5%88%86%E5%8C%BA)
## Redis的扩容机制？集群扩容的实现原理
[28.说说集群的伸缩？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#28%E8%AF%B4%E8%AF%B4%E9%9B%86%E7%BE%A4%E7%9A%84%E4%BC%B8%E7%BC%A9)
## 节点宕机后如何恢复？自动故障转移的实现原理
[27.能说说 Redis 集群的原理吗？](https://github.com/itwanger/toBeBetterJavaer/blob/master/docs/src/sidebar/sanfene/redis.md#27%E8%83%BD%E8%AF%B4%E8%AF%B4-redis-%E9%9B%86%E7%BE%A4%E7%9A%84%E5%8E%9F%E7%90%86%E5%90%97)