> **模块边界**：XXL-Job 本体 → XXL-Job 模块；分布式锁与协调理论 → 分布式模块。本模块覆盖 Quartz 的三元组、Misfire、持久化与集群。

**① Quartz 总览**

1. 是什么、定位（调度库，非调度平台）
2. 与 XXL-Job / Spring Task 对比
3. 应用场景
4. 版本演进（2.x API 差异：Builder 链式构建 / 注解）

**② 核心概念** 5. Scheduler 6. Job / JobDetail 7. Trigger 8. JobDataMap 9. JobExecutionContext 10. Calendar（排除特定日期）

**③ Job** 11. Job 接口与 JobFactory 实例化机制 12. 有状态 vs 无状态 Job 13. @DisallowConcurrentExecution 与并发执行控制 14. @PersistJobDataAfterExecution 15. Job 抛异常后的行为、幂等需业务自理

**④ Trigger** 16. SimpleTrigger 17. CronTrigger 18. Cron 表达式详解（七位格式 + `? * / - , L W #`） 19. 触发器优先级与状态

**⑤ Misfire（重点）** 20. Misfire 定义与 misfireThreshold、判定时机 21. SimpleTrigger 的 Misfire 策略 22. CronTrigger 的 Misfire 策略 23. 具体 Misfire 指令（FIRE_ONCE_NOW / DO_NOTHING / SMART_POLICY 等）

**⑥ Scheduler 与线程模型** 24. 创建与生命周期（start / shutdown / Standby） 25. 两阶段调度模型（调度线程算 nextFireTime + 工作线程池执行） 26. nextFireTime 计算与触发判定 27. 工作线程池（SimpleThreadPool）

**⑦ 持久化** 28. RAMJobStore 29. JDBCJobStore、clustered 配置与 instanceId 30. 表结构（QRTZ_ 前缀 11 张表）

**⑧ 集群** 31. 集群原理（数据库行锁抢占、acquireTriggers） 32. 负载均衡（多 Scheduler 抢 Trigger） 33. 故障转移 34. 时钟同步问题

**⑨ 监听器与监控** 35. JobListener / TriggerListener / SchedulerListener 36. 监控（无原生 UI，靠 JMX / 自行埋点）

**⑩ Spring 集成、选型与坑** 37. Spring Task（@Scheduled） 38. Quartz + Spring Boot（SchedulerFactoryBean / JobDetailFactoryBean / CronTriggerFactoryBean） 39. 调度框架选型总表（Quartz / Spring Task / XXL-Job / Elastic-Job）与常见坑清单（时钟不同步致重复 / 未加 @DisallowConcurrentExecution / Misfire 配置不当 / QRTZ 表误删 / Job 异常未处理）