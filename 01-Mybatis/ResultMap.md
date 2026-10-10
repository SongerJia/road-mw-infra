## 结果映射
结果映射是什么->resultType vs resultMap->字段映射->关联映射->TypeHandler->面试高频题
### 结果映射是什么
结果映射：把数据库查询结构的列，映射到Java对象属性。
```
// 数据库列 → Java 属性
// user_name（列）→ userName（属性）
// MyBatis 负责这个"翻译"

// 两种方式：
// ① resultType：自动映射（列名 = 属性名或驼峰转换）
// ② resultMap：手动映射（自定义规则，处理复杂场景）
```
### resultType vs resultMap
resultType自动映射
```
<select id="findById" resultType="User">
    SELECT id, user_name, age FROM user WHERE id = #{id}
</select>
<!-- 自动映射：
     id → id
     user_name → userName（开启驼峰映射后）
     age → age -->
```
```
# 开启驼峰映射（列 snake_case → 属性 camelCase）
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```
resultMap 手动映射
```
<!-- 定义 resultMap -->
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>                <!-- 主键 -->
    <result property="userName" column="user_name"/>  <!-- 普通字段 -->
    <result property="age" column="age"/>
</resultMap>

<select id="findById" resultMap="userResultMap">
    SELECT id, user_name, age FROM user WHERE id = #{id}
</select>
<!-- 手动指定映射规则（列 user_name → 属性 userName） -->
```
对比

|对比|resultType|resultMap|
|---|---|---|
|**映射**|自动（名称匹配）|手动（自定义）|
|**驼峰**|需开启配置|手动指定|
|**复杂对象**|❌ 难|✅ 支持|
|**关联查询**|❌|✅|
|**性能**|简单快|略慢（解析多）|
|**推荐**|简单查询|复杂映射|
### 字段映射规则
resultType
```
// 规则：
// ① 列名 == 属性名 → 直接映射
// ② 列名 snake_case → 属性 camelCase（开启驼峰配置后）
// ③ 不匹配的字段 → 忽略（不报错，但值为 null）

// 例：
// 列 user_name → 属性 userName（驼峰开启）
// 列 NAME（大写）→ 属性 name（大小写不敏感）
```
resultMap
```
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <result property="userName" column="user_name"/>
    <result property="email" column="email_addr"/>  <!-- 列名和属性名完全不同 -->
</resultMap>
```
resultMap的标签

|标签|作用|
|---|---|
|**id**|主键（优化性能，区分对象）|
|**result**|普通字段|
|**association**|一对一关联（嵌套对象）|
|**collection**|一对多关联（集合）|
|**discriminator**|鉴别器（按列值选不同映射）|
### 关联映射
一对一 association
```
<!-- 场景：订单 → 用户（一个订单属于一个用户） -->
<resultMap id="orderResultMap" type="Order">
    <id property="id" column="id"/>
    <result property="orderNo" column="order_no"/>
    <!-- 关联对象：user -->
    <association property="user" javaType="User">
        <id property="id" column="user_id"/>
        <result property="name" column="user_name"/>
    </association>
</resultMap>

<select id="findOrder" resultMap="orderResultMap">
    SELECT o.id, o.order_no, o.user_id, u.name AS user_name
    FROM orders o JOIN user u ON o.user_id = u.id
    WHERE o.id = #{id}
</select>
```
一对多 collection
```
<!-- 场景：用户 → 订单（一个用户有多个订单） -->
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 关联集合：orders -->
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
        <result property="orderNo" column="order_no"/>
    </collection>
</resultMap>

<select id="findUserWithOrders" resultMap="userResultMap">
    SELECT u.id, u.name, o.id AS order_id, o.order_no
    FROM user u LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = #{id}
</select>
```
两种关联的写法
```
// ① 嵌套结果（上面这种）：一次 JOIN 查询 + resultMap 嵌套映射
// ② 嵌套查询（分步查询）：
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <collection property="orders" ofType="Order"
                select="findOrdersByUserId" column="id"/>
</resultMap>
<select id="findOrdersByUserId" resultType="Order">
    SELECT * FROM orders WHERE user_id = #{id}
</select>
// 触发 N+1 问题  每个外层查询都会触发N次findOrdersByUserId查询
```
### TypeHandler 类型处理器
```
// 职责：Java 类型 ↔ JDBC 类型的转换
// 例：Java String ↔ 数据库 VARCHAR

// 内置 TypeHandler：
// StringTypeHandler：String ↔ VARCHAR
// IntegerTypeHandler：Integer ↔ INT
// DateTypeHandler：Date ↔ TIMESTAMP
// BooleanTypeHandler：Boolean ↔ TINYINT

// 自定义 TypeHandler（如 JSON 转换）：
@MappedTypes(User.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class JsonTypeHandler extends BaseTypeHandler<User> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
            User parameter, JdbcType jdbcType) throws SQLException {
        // Java 对象 → JSON 字符串
        ps.setString(i, JSON.toJSONString(parameter));
    }

    @Override
    public User getNullableResult(ResultSet rs, String columnName) throws SQLException {
        // JSON 字符串 → Java 对象
        return JSON.parseObject(rs.getString(columnName), User.class);
    }
}

// 使用：
<resultMap type="User">
    <result property="config" column="config"
            typeHandler="com.example.JsonTypeHandler"/>
</resultMap>
```
### 面试高频题
#### 题目1：resultType和resultMap的区别
```
// resultType：自动映射（名称匹配/驼峰）
// resultMap：手动映射（复杂场景/关联）
```
#### 题目2：字段不一致怎么办
```
// ① 开启驼峰映射（user_name → userName）
// ② resultMap 手动映射
// ③ SQL 中 AS 别名
```
#### 题目3：association和collection
```
// association：一对一（嵌套对象）
// collection：一对多（集合）
```