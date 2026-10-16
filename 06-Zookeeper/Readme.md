> **模块边界**：Dubbo / Kafka 里的注册与元数据用法 → 各自模块；分布式协调的理论（CAP / 选主通用方案）→ 分布式模块。本模块覆盖 ZooKeeper 的数据模型、ZAB、集群与典型应用。

**① ZooKeeper 总览**

1. 是什么、定位（分布式协调服务）
2. 应用场景（配置 / 注册 / 锁 / 选主 / 命名 / 集群管理）
3. 与 etcd / Consul 对比
4. 版本演进（3.5+ 动态配置）
5. ZooKeeper 与 Kafka / Dubbo / HBase 的关系

**② 数据模型** 6. ZNode 与四类节点（持久 / 临时 / 顺序 / 临时顺序） 7. 树形结构与命名空间、Chroot 8. 节点数据 stat（czxid / mzxid / version / ephemeralOwner） 9. 节点限制（1MB）与内存存储 10. ACL 权限模型

**③ Watcher** 11. 一次性触发 12. 事件类型（NodeCreated / Deleted / DataChanged / ChildrenChanged） 13. 注册与通知、客户端串行 14. 惊群效应与规避

**④ 会话** 15. 会话建立与 sessionTimeout 16. 会话状态（CONNECTING / CONNECTED / RECONNECTING / LOST） 17. 会话心跳与过期导致临时节点被删、会话跟踪与临时节点清理 18. 会话重连与恢复

**⑤ ZAB 协议** 19. ZAB 协议与顺序一致性（可能读到旧数据、sync() 的作用） 20. 崩溃恢复（选主 + 数据同步）与消息广播（类 2PC） 21. ZXID（epoch + counter） 22. Leader 选举流程（LOOKING / LEADING / FOLLOWING / OBSERVING）与比较规则 23. 数据同步

**⑥ 集群** 24. 角色（Leader / Follower / Observer、Observer 不投票） 25. 集群规模为何是奇数与多数派 quorum、脑裂如何被避免 26. 读写请求处理（写全走 Leader + 过半确认）、单点写瓶颈

**⑦ 客户端与 Curator** 27. 原生客户端 28. Curator Recipe（分布式锁 / Leader 选举 / 分布式计数器 / 屏障）与连接重试

**⑧ 典型应用** 29. 配置中心 30. 服务注册与发现、与 Nacos 作为注册中心的对比 31. 分布式锁（临时顺序节点 + 监听前一节点） 32. 选主、命名服务

**⑨ 运维、性能与容灾** 33. 配置文件（zoo.cfg）与生产配置建议（奇数节点 / 独立磁盘放事务日志 / maxClientCnxns） 34. 事务日志与快照（fsync / snapCount 与性能） 35. 四字命令（ruok / stat / mntr / cons / srvr / conf）与监控指标 36. 扩容与迁移（加 Observer 扩读能力） 37. 常见坑清单（会话超时致锁丢失 / Watcher 只触发一次 / 读旧数据 / 节点数过多 / jute.maxbuffer 限制）