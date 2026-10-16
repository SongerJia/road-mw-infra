> **模块边界**：XA / TCC / SAGA 的一般理论 → 分布式模块；Nacos / ZK 本体 → 各自模块；Spring 事务机制 → Spring 模块。本模块覆盖 Seata 的角色、四模式实现与工程落地。

**① Seata 总览**

1. 是什么、定位（分布式事务解决方案）
2. 与其他方案对比
3. 核心角色（TC / TM / RM）
4. 角色协同流程（TM 开启 → RM 注册分支 → TC 驱动提交/回滚）
5. 应用场景
6. 版本演进（AT 默认、XA 1.2+、2.x 优化）
7. 模式选择建议

**② 核心概念** 8. 全局事务与分支事务 9. XID 与调用链透传（Feign / Dubbo） 10. 全局锁

**③ AT 模式原理（重点）** 11. 一阶段（本地事务内：业务 SQL + undo log + 注册分支 + 申请全局锁） 12. 二阶段：提交（异步删 undo log）/ 回滚（基于 beforeImage 生成反向 SQL 补偿） 13. undo log 结构与字段（xid / branch_id / rollback_info / log_status、beforeImage / afterImage） 14. SQL 解析（Druid）与镜像生成 15. 全局锁的申请与释放、冲突时的重试与超时回滚 16. 隔离性：写隔离（全局锁防脏写）/ 读隔离（默认读未提交，需 SELECT FOR UPDATE） 17. 适用与受限场景（需主键、SQL 可解析等）

**④ TCC 模式** 18. 三阶段（Try / Confirm / Cancel） 19. 空回滚 20. 悬挂（为何 Cancel 先于 Try） 21. 幂等

**⑤ SAGA 模式** 22. 正向与补偿服务 23. 状态机引擎与 JSON 定义

**⑥ XA 模式** 24. 基于数据库 XA、与 AT 的区别

**⑦ 事务协调（TC）** 25. 事务状态机（Begin / Active / Committing / Rollbacking / Finished…） 26. 事务开启与分支注册、提交与回滚驱动 27. 重试机制与事务超时自动回滚

**⑧ 部署与高可用** 28. TC Server 与事务分组（tx-service-group） 29. 存储三模式取舍（file 单机 / db 集群 / redis） 30. 集群部署与 TC 寻址 31. 注册中心集成

**⑨ Spring 集成** 32. @GlobalTransactional 与代理失效 33. 数据源代理（DataSourceProxy）与 undo_log 表初始化 34. undo_log 定时清理

**⑩ 性能、坑与选型** 35. 性能开销与优化（全局锁竞争 / undo log 存储 / SQL 解析成本） 36. 常见坑（@GlobalTransactional 不生效 / undo_log 未创建 / TC 未启动 / 数据源未代理 / 事务不回滚 / 全局锁冲突） 37. 生产部署清单（TC 集群 + db 存储 + 注册中心 + 事务分组规划） 38. 分布式事务方案选型矩阵（Seata 四模式 + 本地消息表 + 可靠消息 + 最大努力通知） 39. 边界：XA / TCC / SAGA 一般理论 → 分布式模块 40. 边界：Nacos / ZK 本体 → 各自模块 41. 边界：Spring 事务机制 → Spring 模块 42. 与 Spring Cloud Alibaba 生态的协作（Nacos 存储 + Seata 事务）