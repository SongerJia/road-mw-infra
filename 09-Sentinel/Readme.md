> **模块边界**：Hystrix / Resilience4j 本体 → 各自模块；Spring Cloud 组件用法 → Spring Cloud 模块；限流算法的一般理论 → 分布式模块。本模块覆盖 Sentinel 的插槽链、流控熔断、规则管理与集成。

**① Sentinel 总览**

1. 是什么、定位（流量控制 / 熔断降级 / 系统保护）
2. 与 Hystrix / Resilience4j 对比
3. 核心概念（资源 / 规则 / Entry / Context / 节点 / Slot Chain）
4. 应用场景
5. 版本演进与性能开销（统计内存与 CPU）

**② 核心概念** 6. 资源 Resource 与资源埋点方式（注解 / SphU API / 适配器自动埋点） 7. 规则 Rule 8. Entry 与 Context（命名空间作用） 9. 节点 Node（DefaultNode / ClusterNode / StatisticNode） 10. Slot Chain

**③ 工作流程** 11. Slot 链全序（NodeSelector → ClusterBuilder → Log → Statistic → Authority → System → Flow → Degrade） 12. 一次请求处理流程 13. 统计与滑动窗口（LeapArray）、统计维度（通过 / 拒绝 / 异常 / RT / 并发）

**④ 流控** 14. 流控规则 15. 流控模式（直接 / 关联 / 链路） 16. 流控效果（快速失败 / Warm Up / 排队等待） 17. 阈值类型（QPS / 线程数） 18. 限流核心算法（滑动窗口计数；Warm Up / 排队等待的桶思路）

**⑤ 熔断降级** 19. 熔断机制（基于统计的跳闸 + 时间窗恢复，与 Hystrix 三态的区别） 20. 熔断策略（慢调用比例 / 异常比例 / 异常数） 21. 熔断参数（阈值 / 熔断时长 / 最小请求数 / 统计时长） 22. 降级

**⑥ 系统保护** 23. 系统规则（Load / CPU / RT / 线程数 / 入口 QPS） 24. 系统保护原理（Load1 与 BBR）

**⑦ 热点参数限流** 25. 热点规则与实现（LRU + 参数索引）

**⑧ 授权规则** 26. 黑白名单

**⑨ 规则管理** 27. 规则加载方式（代码 / 文件 / Nacos / ZK） 28. Push / Pull 推送模式 29. 持久化方案（Nacos 作为规则数据源的标准做法） 30. 资源名匹配与注解式资源定义

**⑩ 控制台与监控** 31. Dashboard 与实时监控 32. 与 Prometheus / Grafana 集成

**⑪ 集群流控** 33. Token Server / Client、嵌入 / 独立两种模式

**⑫ 注解与集成** 34. @SentinelResource 35. blockHandler 与 fallback 的语义与执行顺序 36. Spring Cloud / OpenFeign 集成 37. Gateway 集成 38. 与 Spring Cloud Alibaba 整体整合视角（Nacos + Gateway + Feign） 39. 常见坑（规则未持久化 / 资源名不一致 / 链路模式入口 / Gateway 粒度） 40. Sentinel vs Hystrix 对照表 41. 边界：Hystrix / Resilience4j 本体 → 各自模块；限流算法一般理论 → 分布式模块