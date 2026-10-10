## 缓存机制
两级缓存总览->一级缓存->二级缓存->对比->缓存失效->面试高频题
### 两级缓存总览
```
查询请求
    ↓
① 二级缓存（Namespace 级，跨 Session）
    ↓ 未命中
② 一级缓存（SqlSession 级）
    ↓ 未命中
③ 数据库
    ↓
写入缓存（反向）

// 一级：SqlSession 内部（默认开启）
// 二级：Namespace 共享（需配置开启）
```

|缓存|作用域|默认|底层|
|---|---|---|---|
|**一级缓存**|SqlSession|✅ 开启|PerpetualCache（HashMap）|
|**二级缓存**|Namespace（Mapper）|❌ 关闭|PerpetualCache + 装饰器|
### 一级缓存
```
// 一级缓存：同一个 SqlSession 内有效
// 作用域：SqlSession（会话级）
// 默认：开启，无法关闭（只能清空）

// 例：
SqlSession session = factory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class);

User u1 = mapper.findById(1L);  // 查数据库（发 SQL）
User u2 = mapper.findById(1L);  // 直接返回缓存（不发 SQL）✅
// 同一个 Session 内，相同查询复用缓存
```
原理
```
// 缓存在 BaseExecutor 里：
public abstract class BaseExecutor {
    // 一级缓存（PerpetualCache，本质 HashMap）
    protected PerpetualCache localCache;

    public <E> List<E> query(...) {
        // 缓存 key：statementId + 参数 + 分页 + SQL
        CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
        // 查缓存
        list = localCache.getObject(key);
        if (list != null) {
            return list;   // 命中
        }
        // 未命中 → 查库 → 存缓存
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        return list;
    }
}
```
一级缓存失效
```
// 一级缓存什么时候清空：
// ① 执行了 update/insert/delete（增删改会清空缓存）
// ② 调用 session.clearCache()
// ③ 不同的 SqlSession
// ④ 查询条件不同（key 不同）

SqlSession session = factory.openSession();
User u1 = mapper.findById(1L);  // 查库
User u2 = mapper.findById(1L);  // 缓存

mapper.updateUser(...);         // 增删改 → 清空一级缓存！
User u3 = mapper.findById(1L);  // 重新查库（缓存已清）
```
### 二级缓存
```
// 二级缓存：同一个 Mapper（Namespace）内有效
// 作用域：Mapper 级（跨 SqlSession）
// 默认：关闭，需配置开启

// 特点：
// ① 跨 Session 共享
// ② 以 Namespace 为单位
// ③ 需要对象可序列化（可能跨 Session）
```
开启方式
```
<!-- ① 全局开关（application.yml） -->
<!-- mybatis.configuration.cache-enabled: true（默认 true） -->

<!-- ② Mapper 级开启（XML） -->
<mapper namespace="com.example.UserMapper">
    <cache/>   <!-- 开启本 Mapper 的二级缓存 -->
    <!-- 或带配置： -->
    <cache eviction="LRU"           <!-- 淘汰策略 -->
           flushInterval="60000"     <!-- 刷新间隔 -->
           size="1024"               <!-- 最大对象数 -->
           readOnly="false"/>        <!-- 是否只读 -->
</mapper>
```
```
// ③ 注解方式
@CacheNamespace  // 开启二级缓存
public interface UserMapper { ... }
```
原理
```
// 二级缓存由 CachingExecutor 装饰：
// ① Executor 外层包 CachingExecutor
// ② CachingExecutor 查二级缓存（TransactionalCacheManager）
// ③ 事务提交后才真正写入二级缓存

// 流程：
// 查询 → CachingExecutor（二级缓存）
// → 未命中 → BaseExecutor（一级缓存）
// → 未命中 → 数据库
```
二级缓存失效
```
// ① 同一个 Namespace 内增删改 → 清空本 Namespace 二级缓存
// ② 其他 Namespace 的增删改 → 不清（可能读到脏数据！）
// ③ session.commit() 后才写入
```
### 一级与二级对比
| 对比            | 一级缓存           | 二级缓存              |
| ------------- | -------------- | ----------------- |
| **作用域**       | SqlSession     | Namespace（Mapper） |
| **默认**        | ✅ 开启           | ❌ 关闭              |
| **跨 Session** | ❌              | ✅                 |
| **共享**        | 一个 Session 内   | 所有 Session        |
| **序列化**       | 不需要            | 需要（可能跨线程）         |
| **失效**        | 增删改/clearCache | 本 Namespace 增删改   |
| **性能**        | 会话内            | 全局                |
### 缓存失效汇总
```
// 一级缓存失效：
// ① 增删改（本 Session）
// ② clearCache()
// ③ 不同 Session
// ④ 不同查询条件

// 二级缓存失效：
// ① 本 Namespace 增删改
// ② flushInterval 到期
// ③ size 超限（淘汰）
// ④ rollback

// 注意（脏数据）：
// 二级缓存跨 Namespace 不互相清空
// 多表操作可能读到脏数据
// → 谨慎使用二级缓存！
```
### 二级缓存的坑
①跨NameSpace脏数据
```
// UserMapper 开了二级缓存
// OrderMapper 修改了 user 表（但没清 UserMapper 缓存）
// → UserMapper 读到脏数据！

// 因为：二级缓存只清"本 Namespace"
// 解决：<cache-ref namespace="目标Mapper"/>（共享缓存）
```
②多表关联脏数据
```
// user 表关联 orders 表
// OrderMapper 改了 orders
// UserMapper 的缓存（含 orders 数据）没清 → 脏数据
```
③分布式
```
// 二级缓存在"单机 JVM"内存
// 多台服务器各有一份 → 不一致！
// 解决：用 Redis 做二级缓存（MyBatis 集成 Redis）
```
建议
```
// 实际生产：大部分项目不开二级缓存！
// 原因：
// ① 脏数据风险（跨 Namespace/多表）
// ② 分布式不一致
// ③ 已有 Redis 等外部缓存

// 若用：只用于"单表、不常改"的查询
```
### 面试高频题
#### 题目1：一级缓存、二级缓存区别
```
// 一级：Session 级（默认开）
// 二级：Namespace 级（默认关）
```
#### 题目2：二级缓存什么时候写入
```
// 事务提交（commit）后
```
#### 题目3：一级缓存能关吗
```
// 默认开启，无法直接关闭
// 只能通过 clearCache 或增删改清空
```