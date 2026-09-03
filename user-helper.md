# 项目介绍（基于代码分析）

以下内容基于仓库源码分析编写（不参考 README 的介绍），总结项目的整体功能、技术栈、架构与关键实现点，便于在主目录下快速了解本项目。

## 概述

本项目是一个基于 Spring Boot 的后端服务，围绕商铺（Shop）等业务实体提供数据查询、缓存与地理位置检索能力。项目以 MySQL 作为持久化存储，广泛使用 Redis 提升查询性能、支持高并发场景并实现分布式功能。项目包含若干用于导入与缓存店铺数据、分布式 ID 生成和并发测试的工具代码，显示出对热点数据与高并发场景的关注（例如秒杀/下单场景）。

## 技术栈

- Java 8，Spring Boot（父 POM 使用 2.3.12.RELEASE）
- MyBatis-Plus（持久层）
- Redis：spring-data-redis、Lettuce 客户端、Redisson（分布式锁）
- RabbitMQ（依赖存在，暗示异步/消息队列场景）
- 其他：Lombok、Hutool、Apache Commons Pool2

## 高层架构与模块

- Web 层：Spring MVC 提供 HTTP 接口（通过 spring-boot-starter-web 推断）
- 服务层：Service 接口与实现（示例：ShopServiceImpl、IShopService）
- 持久层：MyBatis-Plus 映射与实体（示例：Shop 实体）
- 缓存/工具层：CacheClient（自定义缓存工具）、RedisIdWorker（分布式 ID 生成器）、RedisConstants（Redis key 与 TTL 配置）
- 基础设施：Redis（缓存、GEO、HyperLogLog、Bitmap、ZSet 等场景）、Redisson（分布式锁）、RabbitMQ（异步处理）

## 关键功能与实现要点

1. Redis GEO（附近商铺检索）
   - 将数据库中的店铺按类型分组，生成 GeoLocation 列表并一次性写入 Redis（stringRedisTemplate.opsForGeo().add）。
   - 用途：按经纬度范围查询附近商铺并按距离排序。

2. 热点缓存策略（逻辑过期）
   - CacheClient 提供 setWithLogicalExpire 方法，测试代码演示将 Shop 对象以“逻辑过期”方式写入缓存（保存数据 + 逻辑过期时间），配合后台异步刷新以避免缓存击穿。

3. 分布式 ID（RedisIdWorker）
   - 提供高并发场景下的唯一 ID ���成（用于订单等资源）。测试包含大量并发生成 ID 的压力验证代码。

4. 并发与分布式锁
   - 项目引入 Redisson，常用场景为分布式锁、确保“在集群下一人一单”或保护关键更新（如库存扣减）。

5. 消息队列（RabbitMQ）
   - POM 中包含 spring-boot-starter-amqp，项目中可能使用消息队列异步化下单、库存扣减或秒杀场景（依赖表明存在消费者/生产者的实现或预留）。

6. 其他 Redis 高级数据类型应用
   - HyperLogLog：用于 UV 统计（节省内存的基数估算）。
   - Bitmap：用于用户签到记录与连续签到天数统计。
   - ZSet：用于点赞排行榜（score 可用时间戳或权重排序）。

## 并发/性能考量

- 使用 Redis 缓存与多种缓存策略（空值缓存、互斥锁、逻辑过期）来减少 DB 压力。
- 引入 Redisson 实现分布式锁避免集群下的竞态条件。
- 使用消息队列（或 Redis Stream）做异步下单/订单落库，提升吞吐。
- 提供分布式 ID 生成以支持多实例部署下的全局唯一 ID。

## 运行与部署（概览）

- 构建：mvn clean package
- 启动：java -jar target/hm-dianping-0.0.1-SNAPSHOT.jar
- 运行前需配置：
  - MySQL 数据源（application.yml/properties 中配置）
  - Redis 连接（lettuce/redisson）
  - 若使用消息队列：RabbitMQ 连接配置
- 测试：项目含若干集成测试（src/test/java/*），可用来检验导入 GEO、缓存写入、ID 生成等核心能力。

## 注意事项与建议

- 依赖兼容性：父 POM 使用 Spring Boot 2.3.x，而部分显式依赖（如 spring-data-redis 2.6.2、lettuce 6.x）版本可能与 Spring Boot 2.3 存在兼容性差异，部署前建议确认各依赖版本的兼容性并执行集成测试。
- 资源控制：测试中使用较大的线程池（例如固定线程池 500），在 CI 或本地运行时请注意机器资源，避免 OOM。
- 配置管理：生产环境中请使用外部化配置或环境变量管理数据库与 Redis 凭据，避免敏感信息泄露。
- 监控与告警：建议监控 Redis 命中率、Pending 队列长度、消息队列积压、缓存刷新失败率等关键指标。

## 后续建议（可选）

- 生成接口文档（自动扫描 Controller）并导出 REST API 列表。
- 列出所有 Redis key（由 RedisConstants 汇总）与用途，方便运维和容量评估。
- 对关键路径（下单/秒杀）进行端到端压测与容量规划。
- 如需，我可以继续：
  - 自动扫描仓库并生成 API/类清单、Redis key 列表与消息队列使用点；
  - 或把��文件翻译成英文版本并加入 CONTRIBUTING/DEPLOY 文档。

---

> 文件由仓库源码分析生成 — 侧重代码与实现，不引用 README 内容。
