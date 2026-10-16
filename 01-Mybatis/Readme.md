> **模块边界**：JDBC 与连接池（Druid / HikariCP）→ 基础 / 中间件模块；Spring 事务机制 → Spring 模块（此处只讲 MyBatis 侧整合）；数据库 SQL 语法 → MySQL 模块。本模块覆盖 MyBatis 的执行链路、映射、插件、缓存与 Plus。

**① MyBatis 总览**

1. 是什么、定位（半自动 ORM）
2. 与 Hibernate / JPA 对比
3. 核心组件
4. 与 MyBatis-Plus 的关系与边界
5. 版本演进

**② 核心执行流程** 6. MyBatis 初始化（XMLConfigBuilder → Configuration） 7. SqlSessionFactory / Builder 8. SqlSession 与线程安全 9. Executor 三类型（Simple / Reuse / Batch） 10. StatementHandler 与 StatementType（STATEMENT / PREPARED / CALLABLE） 11. ParameterHandler / ResultSetHandler 12. MapperProxy 动态代理与 MapperRegistry / MapperProxyFactory 13. 一次查询的完整流程

**③ 配置** 14. mybatis-config.xml 15. settings（mapUnderscoreToCamelCase 等） 16. typeAliases / typeHandlers 17. environments 18. 加载 mapper

**④ 映射** 19. resultMap 与 extends 继承、discriminator 鉴别器 20. 一对一 / 一对多 / 多对多 21. 嵌套查询 vs 嵌套结果、N+1 查询问题 22. 延迟加载及原理（ProxyFactory + Javassist / CGLIB） 23. SqlSource 体系（Dynamic / Raw / Static）与 BoundSql 24. 动态 SQL 25. SQL 片段 26. `#{}` vs `${}` 与 SQL 注入防范 27. 主键回填 28. 批量操作三方式（foreach / BatchExecutor / ExecutorType.BATCH）

**⑤ 接口与注解** 29. Mapper 接口 30. 注解式 CRUD 31. @Param 32. @Results / @One / @Many

**⑥ 插件与扩展** 33. 插件机制与实现要点（@Intercepts / @Signature / Plugin.wrap） 34. 四大可拦截对象与责任链顺序 35. PageHelper 原理（SQL 改写 + count） 36. 物理分页 vs 逻辑分页 37. TypeHandler 38. MetaObject / Reflector 反射工具

**⑦ 缓存** 39. 一级缓存、失效场景与 localCacheScope 40. 二级缓存、坑与默认关闭的原因 41. 缓存装饰器链（PerpetualCache / LruCache / BlockingCache）

**⑧ 事务与数据源** 42. 与 Spring 事务整合与 SqlSession 生命周期 43. 多数据源（AbstractRoutingDataSource）

**⑨ Spring 集成** 44. SqlSessionTemplate 45. @MapperScan 46. starter

**⑩ MyBatis-Plus 与规范** 47. 基础 CRUD 48. 条件构造器（QueryWrapper / LambdaQueryWrapper） 49. 分页插件 50. 逻辑删除 51. 乐观锁 / 自动填充 / 防全表更新删除 / 多租户 52. 代码生成器、与原生 MyBatis 混用 53. 常见坑清单（`#`/`$` 注入 / 一级缓存脏读 / 二级缓存跨服务脏读 / 延迟加载失效 / N+1 / foreach 大集合 / @Param 缺失）