## 延迟加载
延迟加载是什么->立即加载 vs 延迟->配置->N+1问题->解决方案->面试高频题
### 延迟加载是什么
延迟加载：关联对象在真正使用时才加载，而不是查询对象时立即加载。
```
// 场景：查询订单（订单关联用户）
Order order = orderMapper.findById(1L);

// 立即加载：查订单时就把用户也查出来（一次查全）
order.getUser().getName();   // 已经加载好了

// 延迟加载：查订单时只查订单，用到 user 才查
order.getUser().getName();   // 触发时才去查用户（懒加载）
```
### 立即加载 vs 延迟加载
| 对比         | 立即加载      | 延迟加载       |
| ---------- | --------- | ---------- |
| **时机**     | 查询时立即查关联  | 使用时才查关联    |
| **SQL 次数** | 1 次（JOIN） | N 次（每次用都查） |
| **内存**     | 占多（所有数据）  | 占少（按需）     |
| **首次查询**   | 慢（关联都查）   | 快（只查主对象）   |
| **后续访问**   | 快（已有）     | 慢（触发查询）    |
| **适用**     | 一定会用关联    | 不一定会用关联    |
配置
```
mybatis:
  configuration:
    # 全局开关：是否开启延迟加载
    lazy-loading-enabled: true
    # 触发时机：true=访问时加载，false=调用方法时加载
    aggressive-lazy-loading: false
    # 是否在查询时加载所有的延迟属性（覆盖 aggressive）
    lazy-load-trigger-methods: equals,clone,hashCode,toString
```
```
<!-- 单个 resultMap 指定延迟 -->
<resultMap id="userResultMap" type="User">
    <collection property="orders" ofType="Order"
                select="findOrdersByUserId" column="id"
                fetchType="lazy"/>   <!-- lazy=延迟，eager=立即 -->
</resultMap>
```
### 延迟加载的触发
```
// 延迟加载用"代理对象"实现：
// 关联对象被替换为代理
// 访问属性/方法时 → 代理触发真正的查询

// 触发时机（aggressive-lazy-loading=false）：
Order order = orderMapper.findById(1L);  // 只查订单
order.getUserId();                       // 订单自己的属性，不触发
order.getUser();                         // ❌ 只拿到代理，不触发
order.getUser().getName();               // ✅ 访问 user 属性 → 触发查询

// 触发时机（aggressive-lazy-loading=true）：
order.getUser();   // 一调用就触发（更激进）
```
触发方法
```
// lazy-load-trigger-methods：这些方法调用会触发加载
// equals、clone、hashCode、toString
// 例：打印 order 会触发关联加载（因为 toString）
```
### N+1问题
什么是N+1
```
// 延迟加载（或嵌套查询）导致的性能问题：

// 查 1 个用户 → 1 次 SQL
List<Order> orders = orderMapper.findAll();  // 1 次 SQL（查 N 个订单）

// 遍历每个订单访问 user → N 次 SQL
for (Order order : orders) {
    order.getUser().getName();  // 每个订单触发 1 次查询用户
}
// 总 SQL：1（订单）+ N（用户）= N+1 次！

// 例：100 个订单 → 1 + 100 = 101 次 SQL
// 严重性能问题！
```
为什么N+1慢
```
// ① 数据库交互次数多（网络往返）
// ② 每次查询有开销（解析、连接）
// ③ 100 个订单 → 101 次查询 → 慢

// 对比：
// 立即加载（JOIN）：1 次 SQL 查完
// 延迟加载：1 + N 次 SQL
```
### N+1的解决方案
①立即加载
```
<!-- 用 JOIN + 嵌套结果（一次查完） -->
<resultMap id="orderResultMap" type="Order">
    <id property="id" column="id"/>
    <association property="user" javaType="User">
        <id property="id" column="user_id"/>
        <result property="name" column="user_name"/>
    </association>
</resultMap>

<select id="findAll" resultMap="orderResultMap">
    SELECT o.*, u.id AS user_id, u.name AS user_name
    FROM orders o JOIN user u ON o.user_id = u.id
</select>
<!-- 1 次 SQL 查完所有（无 N+1） -->
```
②批量加载
```
mybatis:
  configuration:
    lazy-loading-enabled: true
    # 批量加载：延迟加载时，把多个查询合并
    default-executor-type: batch
```
```
// 或 @BatchSize（MyBatis-Plus）
// 访问第一个 user 时，批量查所有需要的 user
```
③用@BatchSize/全局批量
```
<collection property="orders" ofType="Order"
            select="findOrdersByUserId" column="id"
            fetchType="lazy"/>
<!-- 配合 batch：一次查多个（不是逐个） -->
```

|方案|SQL 次数|说明|
|---|---|---|
|**立即加载（JOIN）**|1 次|✅ 推荐（一次查）|
|**延迟加载（默认）**|N+1 次|❌ 性能差|
|**延迟 + batch**|1 + M 次|折中（M < N）|
### 高频面试题
#### 题目1：延迟加载是什么
```
// 关联对象使用时才加载
// 代理对象触发
```
#### 题目2：立即 vs 延迟
```
// 立即：一次 JOIN 查完（1 次 SQL）
// 延迟：按需查（N+1 次 SQL）
```
#### 题目3：什么是N+1问题
```
// 查主对象 1 次 + 每个关联 N 次 = N+1
// 延迟加载的典型问题
```
