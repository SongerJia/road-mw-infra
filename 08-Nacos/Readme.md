> **模块边界**：Spring Cloud 各组件的用法 → Spring Cloud 模块；ZooKeeper / Eureka 本体 → 各自模块；配置中心的权限与安全体系 → 安全方向。本模块覆盖 Nacos 的注册发现、配置管理与集成。

**① Nacos 总览**

1. 是什么、定位
2. 核心能力（服务发现 / 配置管理 / 动态 DNS）
3. 与 Eureka / Consul / ZK / Apollo 对比
4. 版本演进（1.x HTTP+UDP → 2.x gRPC 长连接）
5. 版本对应关系（Spring Cloud Alibaba ↔ Spring Boot ↔ Nacos）
6. 应用场景

**② 架构** 7. 核心组件（Naming / Config / Console） 8. 服务端与客户端 9. 存储（MySQL / 内嵌 Derby、表结构） 10. 集群架构、集群节点数量与 Raft 过半 11. 数据模型（Namespace / Group / Service / Cluster / Instance；Namespace / Group / DataId）

**③ 服务注册与发现** 12. 服务注册 13. 心跳（5s / 15s / 30s）与健康检查、持久实例的服务端主动探测 14. 临时实例 vs 持久实例 15. 一致性分流：临时实例走 Distro（AP）、持久实例走 Raft（CP） 16. 订阅与推送（2.x gRPC 长连接） 17. 客户端多级缓存（内存 + 本地文件快照 + 服务端降级） 18. 权重与元数据 19. 保护阈值（防雪崩） 20. 与服务发现负载均衡的集成（LoadBalancer / Ribbon） 21. 与 Eureka 的核心差异收口（推 vs 拉 / 健康检查 / AP） 22. 服务上下线延迟的成因与规避

**④ 配置中心** 23. 配置 CRUD 与导入导出 24. 推拉结合（长轮询 29.5s + 变更通知戳） 25. 配置 MD5 校验 26. 本地快照与容灾 27. 优先级 28. 监听与动态刷新 29. 灰度发布（Beta） 30. 历史版本与回滚

**⑤ Spring Cloud Alibaba 集成** 31. 服务发现 starter 32. 配置 starter、@RefreshScope 衔接 33. 与 Spring Cloud Gateway 的集成（服务发现 + 路由） 34. 与 Dubbo 的集成

**⑥ 集群与高可用** 35. 集群部署与负载均衡

**⑦ 安全** 36. 鉴权（1.2+ RBAC） 37. Namespace 隔离作用与常见误用

**⑧ 运维** 38. 控制台与权限 39. 监控指标（服务数 / 实例数 / 配置数 / 长连接数）

**⑨ 收口** 40. 常见坑清单（配置不刷新 / 上下线延迟 / Namespace 混用 / 版本不对齐 / 单机 Derby 丢数据） 41. 注册中心选型对照（Nacos vs Eureka vs ZK vs Consul） 42. 边界：Spring Cloud 组件用法 → Spring Cloud 模块 43. 边界：ZooKeeper / Eureka 本体 → 各自模块 44. 边界：配置安全体系 → 安全方向 45. 配置与服务发现的组合用法（多环境多租户实践） 46. 与 Spring Boot 的配置优先级衔接 47. Nacos 2.x 长连接的资源占用与调优 48. 灰度与蓝绿在配置中心的落法 49. 生产部署建议（集群 + MySQL + 独立 Namespace 规划）