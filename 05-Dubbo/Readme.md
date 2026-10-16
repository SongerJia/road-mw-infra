> **模块边界**：Netty 的传输细节 → Netty 模块；ZooKeeper / Nacos 本体 → 各自模块；Sentinel 的规则与算法 → Spring Cloud / 分布式模块。本模块覆盖 Dubbo 的 SPI 内核、调用链路、集群容错与 3.x 演进。

**① Dubbo 总览**

1. 是什么、定位（高性能 Java RPC 框架）
2. 与 Spring Cloud 对比
3. 三大核心能力（面向接口的 RPC / 智能容错与负载 / 注册发现）
4. 版本演进（2.7 → 3.x）
5. 应用场景

**② 架构** 6. 角色（Provider / Consumer / Registry / Monitor / Container） 7. 调用关系 8. 分层架构（十层） 9. URL 统一配置模型（一切皆 URL）

**③ SPI 机制（Dubbo 内核）** 10. Dubbo SPI vs JDK SPI 11. @SPI / @Adaptive / @Activate 12. 自适应扩展（@Adaptive 原理） 13. Wrapper（AOP）与 IoC（扩展注入） 14. ExtensionLoader

**④ 服务暴露与引用** 15. 服务暴露流程（本地暴露 vs 远程暴露、injvm 协议） 16. 服务引用流程 17. 代理生成（JavassistProxyFactory） 18. Invoker 链与 ProtocolFilterWrapper / Filter 过滤链 19. 服务目录 Directory（StaticDirectory / RegistryDirectory）与路由 Router

**⑤ 注册中心与元数据** 20. 注册中心抽象 21. ZooKeeper / Nacos 22. 服务注册与订阅、本地缓存 23. 元数据中心（服务元数据与注册分离）

**⑥ 远程调用** 24. 调用流程 25. Invoker / Invocation 26. 编解码 27. 网络传输（Netty） 28. 序列化（Hessian2 / Protobuf 与反序列化安全） 29. 协议：dubbo 协议特点（单一长连接 + NIO + Hessian2）

**⑦ 负载均衡** 30. 负载均衡策略（Random / RoundRobin / LeastActive / ConsistentHash / ShortestResponse） 31. 权重与一致性哈希

**⑧ 集群容错与并发控制** 32. 容错策略（Failover / Failfast / Failsafe / Failback / Forking / Broadcast） 33. Failover 的 retries 与 timeout（易放大雪崩） 34. Merger 35. 并发控制（actives / executes）

**⑨ 服务治理** 36. 服务降级与 Mock 37. 路由规则 38. 动态配置与配置优先级 39. 服务分组与版本 40. Token 鉴权与服务鉴权 41. 熔断集成（Sentinel / 自带） 42. 隐式参数

**⑩ 泛化、异步与网关协作** 43. 泛化调用（GenericService） 44. 泛化调用在网关中的应用（HTTP → Dubbo） 45. 异步调用与回调

**⑪ 线程模型** 46. IO 线程与业务线程 47. Dispatcher（all / direct / message / execution / connection） 48. 线程池与拒绝策略

**⑫ Spring 集成与监控** 49. XML / 注解 / Spring Boot Starter 50. Monitor 与 Metrics / Prometheus 集成

**⑬ 运维与故障** 51. telnet 与 QOS 52. 优雅停机 53. 常见故障（线程池打满 / 连接数耗尽 / 超时叠加重试雪崩）

**⑭ 3.x 新特性** 54. 应用级服务发现（动机：接口级注册数据量爆炸） 55. Triple 协议（HTTP/2 + gRPC 兼容 + 云原生穿透） 56. 云原生方向与可观测性 57. 常见坑清单（Failover 重试致非幂等重复 / 超时不合理 / 序列化版本不兼容 / SPI 路径错 / 线程池打满） 58. Dubbo vs Spring Cloud 选型对照表