# XXL-CACHE 项目架构分析文档

## 1. 项目概述

### 1.1 项目定位
XXL-CACHE 是一个高性能的多级缓存框架，专注于高效组合本地缓存和分布式缓存（Redis + Caffeine），提供企业级缓存解决方案。

### 1.2 核心特性
- **多级缓存架构**：L1（本地缓存）+ L2（分布式缓存）双层设计
- **一致性保障**：基于 Redis Pub/Sub 的缓存一致性同步机制
- **高性能**：本地缓存前置，减少远程通信开销
- **高扩展性**：模块化设计，支持自定义缓存提供者和序列化方案
- **透明接入**：业务代码零侵入，简化开发成本
- **缓存治理**：支持 TTL、Category 隔离、防穿透等企业级特性

### 1.3 技术栈
- **开发语言**：Java 17+
- **构建工具**：Maven 3+
- **本地缓存**：Caffeine
- **分布式缓存**：Redis（基于 Jedis 客户端）
- **序列化**：支持 JDK、HESSIAN2、JSON、PROTOSTUFF、KRYO 等多种协议
- **框架集成**：支持 Spring Boot 无缝集成

## 2. 系统架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
├─────────────────────────────────────────────────────────────┤
│                   XxlCacheHelper API                       │
├─────────────────────────────────────────────────────────────┤
│                   XxlCacheFactory                          │
├─────────────────────────────────────────────────────────────┤
│  L1 Cache Manager  │  L2 Cache Manager  │  Broadcast       │
│  (Local)           │  (Distributed)     │  Manager         │
├─────────────────────────────────────────────────────────────┤
│  Caffeine Cache    │  Redis Cache       │  Redis Pub/Sub   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件架构

#### 2.2.1 缓存层次结构
- **L1 缓存（本地缓存）**
  - 基于 Caffeine 实现
  - 应用内存级别缓存
  - 支持 TTL 和容量限制
  - 高性能读写，零网络开销

- **L2 缓存（分布式缓存）**
  - 基于 Redis 实现
  - 集群共享缓存
  - 支持数据持久化
  - 网络通信开销

#### 2.2.2 一致性保障机制
```
写操作流程：
1. 写入 L1 缓存
2. 写入 L2 缓存
3. 发布广播消息（Redis Pub/Sub）
4. 其他节点接收消息，清理本地 L1 缓存

读操作流程：
1. 查询 L1 缓存，命中则返回
2. 查询 L2 缓存，命中则回填 L1 并返回
3. 未命中则返回 null，并在 L1 中标记空值
```

## 3. 核心模块设计

### 3.1 工厂模式 - XxlCacheFactory

**职责**：
- 缓存组件的统一管理和配置
- L1/L2 缓存管理器的初始化
- 广播机制的启动和停止
- 生命周期管理（start/stop）

**关键配置**：
```java
// L1 缓存配置
String l1Provider = "caffeine";
int maxSize = 10000;
long expireAfterWrite = 600; // 秒

// L2 缓存配置
String l2Provider = "redis";
String serializer = "java";
String nodes = "127.0.0.1:6379";
String user = "";
String password = "";
```

### 3.2 缓存抽象层 - Cache Interface

**设计模式**：策略模式 + 工厂模式

```java
public interface Cache {
    void set(String key, CacheValue cacheValue);
    CacheValue get(String key);
    void del(String key);
    Boolean exists(String key);
}
```

**实现类**：
- `CaffeineCache`：本地缓存实现
- `RedisCache`：分布式缓存实现

### 3.3 缓存管理器 - CacheManager

**职责**：
- 缓存实例的创建和管理
- 支持按 Category 分类管理
- 缓存实例的生命周期控制

**核心特性**：
- 线程安全的缓存实例管理（ConcurrentHashMap）
- 懒加载缓存实例创建
- 支持多种缓存类型（通过 CacheTypeEnum 枚举）

### 3.4 序列化层 - Serializer

**设计模式**：策略模式

```java
public interface Serializer {
    byte[] serialize(Object obj) throws Exception;
    Object deserialize(byte[] bytes, Class<?> clazz) throws Exception;
}
```

**支持的序列化协议**：
- Java 原生序列化（默认）
- 可扩展支持 HESSIAN2、JSON、PROTOSTUFF、KRYO 等

### 3.5 广播机制 - CacheBroadcastMessage

**职责**：
- 缓存变更事件的广播
- 集群节点间的缓存一致性同步

**消息结构**：
```java
public class CacheBroadcastMessage {
    private String category;  // 缓存分类
    private String key;       // 缓存键
}
```

### 3.6 Redis实现类分析

项目中的Redis实现主要包括以下类：

- `RedisCache`：实现了`Cache`接口，提供基本的缓存操作（set、get、del、exists等）
- `RedisManager`：负责Redis连接的管理，支持单机和集群模式
- `XxlCacheFactory`：负责缓存组件的统一管理和配置，包括L1/L2缓存管理器的初始化和广播机制的启动

## 4. 数据流设计

### 4.1 缓存写入流程
```
用户调用 set(key, value)
    ↓
生成 CacheValue（包含 TTL）
    ↓
写入 L1 缓存（Caffeine）
    ↓
写入 L2 缓存（Redis）
    ↓
发布广播消息（Redis Pub/Sub）
    ↓
其他节点接收消息，清理对应 L1 缓存
```

### 4.2 缓存读取流程
```
用户调用 get(key)
    ↓
查询 L1 缓存
    ↓
命中且有效？ ──Yes──→ 返回结果
    ↓ No
查询 L2 缓存
    ↓
命中？ ──Yes──→ 回填 L1 缓存 ──→ 返回结果
    ↓ No
在 L1 中标记空值（防穿透）
    ↓
返回 null
```

### 4.3 缓存删除流程
```
用户调用 del(key)
    ↓
删除 L1 缓存
    ↓
删除 L2 缓存
    ↓
发布广播消息
    ↓
其他节点清理对应 L1 缓存
```

## 5. 关键技术特性

### 5.1 TTL 支持
- **实现方式**：CacheValue 包装类携带过期时间
- **L1 缓存**：应用层校验过期时间（Caffeine 不直接支持 key 级别过期）
- **L2 缓存**：Redis 原生 TTL 支持

### 5.2 Category 隔离
- **实现方式**：key 前缀 + 独立缓存实例
- **优势**：不同业务模块缓存隔离，避免 key 冲突
- **管理**：CacheManager 按 category 管理缓存实例
- **实现细节**：通过`CacheUtil.generateKey(category, key)`方法生成带有分类前缀的缓存键

### 5.3 缓存问题防护机制

#### 5.3.1 缓存击穿防护

缓存击穿是指热点key过期时，大量并发请求直接访问数据库的问题。XXL-CACHE通过以下机制防止缓存击穿：

- **多级缓存架构**：即使L2(Redis)缓存中的热点数据过期，L1(本地缓存)仍可能保留数据副本，减轻数据库压力
- **广播机制**：当一个节点更新缓存时，通过Redis Pub/Sub机制通知其他节点更新本地缓存，保持数据一致性
- **TTL差异化**：L1和L2缓存可以设置不同的过期时间，避免同时失效

#### 5.3.2 缓存穿透防护

缓存穿透是指查询一个不存在的数据，导致请求直接落到数据库上的问题。XXL-CACHE通过以下机制防止缓存穿透：

- **空值缓存**：当L2缓存未命中时，会在L1中标记空值(`new CacheValue(null)`)，防止相同的无效查询重复访问数据库
- **多级过滤**：请求需要先经过L1缓存，再到L2缓存，最后才会到达数据库，有效减少穿透概率

#### 5.3.3 缓存雪崩防护

缓存雪崩是指大量缓存同时失效，导致请求直接落到数据库上的问题。XXL-CACHE通过以下机制防止缓存雪崩：

- **多级缓存架构**：即使Redis缓存出现问题，本地缓存仍可提供服务，降低雪崩影响
- **过期时间设计**：支持设置不同的过期时间，避免缓存同时失效
- **本地缓存前置**：高频访问数据优先从本地缓存获取，减少Redis网络通信，提高系统稳定性

### 5.4 一致性保障
- **最终一致性**：通过 Redis Pub/Sub 实现
- **主动失效**：缓存更新时主动通知其他节点清理
- **容错机制**：广播失败不影响缓存操作的正常执行

## 6. 性能优化设计

### 6.1 多级缓存优势
- **热点数据本地化**：高频访问数据优先从 L1 获取
- **网络开销最小化**：减少 Redis 网络通信
- **并发性能提升**：本地缓存支持更高的并发访问

### 6.2 内存管理
- **L1 缓存容量控制**：可配置最大容量，LRU 淘汰策略
- **过期数据清理**：定时清理过期的缓存数据
- **内存占用监控**：支持缓存使用情况监控（规划中）

## 7. 扩展性设计

### 7.1 缓存提供者扩展
- **接口抽象**：Cache 接口定义标准操作
- **枚举管理**：CacheTypeEnum 管理支持的缓存类型
- **工厂创建**：CacheManager 负责实例化具体实现

### 7.2 序列化协议扩展
- **策略模式**：Serializer 接口支持多种实现
- **枚举注册**：SerializerTypeEnum 管理序列化器
- **配置化选择**：通过配置文件指定序列化方案
- **支持协议**：JDK、HESSIAN2、JSON、PROTOSTUFF、KRYO等多种协议

## 8. 部署和集成

### 8.1 Spring Boot 集成
```java
@Bean(initMethod = "start", destroyMethod = "stop")
public XxlCacheFactory xxlCacheFactory() {
    XxlCacheFactory factory = new XxlCacheFactory();
    // 配置参数...
    return factory;
}
```

### 8.2 无框架集成
```java
XxlCacheFactory factory = new XxlCacheFactory();
// 设置配置参数
factory.start();
// 使用缓存
factory.stop();
```

### 8.3 配置文件示例
```properties
# L1 缓存配置
xxl.cache.l1.provider=caffeine
xxl.cache.l1.maxSize=10000
xxl.cache.l1.expireAfterWrite=600

# L2 缓存配置
xxl.cache.l2.provider=redis
xxl.cache.l2.serializer=java
xxl.cache.l2.nodes=127.0.0.1:6379
xxl.cache.l2.user=
xxl.cache.l2.password=
```

### 8.4 使用示例

```java
// 获取缓存实例
Cache cache = XxlCacheFactory.getInstance().getL1CacheManager().getCache("user");

// 写入缓存
cache.set("user:1001", new CacheValue("张三", 3600000)); // 存活时间1小时

// 读取缓存
CacheValue cacheValue = cache.get("user:1001");
String userName = (String) cacheValue.getValue();

// 删除缓存
cache.del("user:1001");
```

## 9. 监控和运维

### 9.1 当前支持
- **日志记录**：详细的缓存操作日志
- **异常处理**：完善的异常捕获和处理机制

### 9.2 规划功能
- **缓存命中率监控**：L1/L2 缓存命中率统计
- **内存使用监控**：L1 缓存容量和占用情况
- **性能指标**：缓存操作耗时统计

## 10. 版本演进

### 10.1 历史版本
- **v1.0.0 (2016-07-25)**：初始版本，支持 Redis/Memcached 管理
- **v1.1.0 (2025-02-04)**：重构为多级缓存框架
- **v1.2.0 (2025-02-07)**：增加多序列化协议支持
- **v1.3.1 (2025-08-16)**：优化广播机制和性能
- **v1.4.0 (2025-08-16)**：升级到 JDK 17

### 10.2 未来规划
- 缓存监控功能完善
- Redis 集群模式支持（Sentinel、Cluster）
- L1 缓存主动过期清理
- 更多序列化协议支持

## 11. Redis在XXL-CACHE中的作用

XXL-CACHE采用了L1(本地缓存)+L2(分布式缓存)的多级缓存架构，Redis在其中作为L2缓存层，主要承担以下功能：

### 11.1 分布式缓存存储

- 作为L2缓存层存储共享数据，供集群中的所有节点访问
- 支持数据持久化，避免应用重启导致的缓存丢失
- 提供集群间数据共享能力

### 11.2 广播机制支持

- 通过Redis的Pub/Sub功能实现缓存一致性的广播通知
- 当一个节点更新缓存时，通过广播机制通知其他节点更新本地缓存
- 实现L1及L2之间以及L1各集群节点间缓存数据一致性同步

## 12. 总结

XXL-CACHE 是一个设计精良的多级缓存框架，具有以下核心优势：

1. **架构清晰**：分层设计，职责明确
2. **性能优异**：多级缓存 + 本地优先策略
3. **扩展性强**：模块化设计，支持自定义扩展
4. **易于使用**：API 简洁，集成方便
5. **企业级特性**：一致性保障、TTL、Category 隔离等
6. **缓存问题防护**：有效防止缓存击穿、穿透和雪崩问题

该框架适用于高并发、高性能要求的缓存场景，特别是需要在性能和一致性之间取得平衡的分布式系统。