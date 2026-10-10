## 参数处理
两种占位符->#{}预编译->${}拼接->SQL注入->使用场景->高频面试题
### 两种占位符
```
// MyBatis 两种参数占位符：

// ① #{}：预编译占位符（推荐）
SELECT * FROM user WHERE id = #{id}
// → 编译后：SELECT * FROM user WHERE id = ?
// → PreparedStatement.setLong(1, id)

// ② ${}：字符串拼接（危险）
SELECT * FROM ${tableName} WHERE id = ${id}
// → 编译后：SELECT * FROM user WHERE id = 1（直接拼进去）
```
### #{}预编译
原理
```
// #{} → ? 占位符 → PreparedStatement 参数设置

// 执行过程：
// ① SQL 带 ? 发送给数据库（预编译）
// ② 参数单独发送（setXxx）
// ③ 数据库执行（参数被当作"数据"，不是"SQL 代码"）

// 源码：
// ParameterHandler.setParameters
ps.setLong(1, parameter.getId());   // 参数按类型安全设置

// 效果：
// 参数值不被解析为 SQL 语法（防注入）
```
为什么安全
```
// 假设输入 id = "1 OR 1=1"
// #{} 处理：
SELECT * FROM user WHERE id = ?   // SQL 结构固定
// 参数 "1 OR 1=1" 被当作字符串值
// → 查 id = "1 OR 1=1" 的用户（查不到，但不会全表）
// → 注入失败 ✅

// 因为参数是"值"，不是"SQL 片段"
```
### ${}拼接
原理
```
// ${} → 直接字符串拼接（不预编译）

// 执行过程：
// ① MyBatis 把 ${} 替换成参数值（字符串拼接）
// ② 拼好的 SQL 发送给数据库（没有预编译）
// ③ 数据库执行（参数值成了 SQL 一部分）

// 源码：
// 动态 SQL 阶段直接拼接：
sql = "SELECT * FROM user WHERE id = " + value;
```
SQL注入
```
// 假设输入 id = "1 OR 1=1"
// ${} 处理：
SELECT * FROM user WHERE id = 1 OR 1=1
// → OR 1=1 永真 → 查全表！
// → 注入成功 ❌

// 更危险：
// id = "1; DROP TABLE user; --"
SELECT * FROM user WHERE id = 1; DROP TABLE user; --
// → 删库！！！
```
### 两者对比
| 对比     | #{}     | ${}     |
| ------ | ------- | ------- |
| **处理** | 预编译（?）  | 字符串拼接   |
| **安全** | ✅ 防注入   | ❌ 有注入风险 |
| **类型** | 自动类型处理  | 无（拼接）   |
| **性能** | 预编译可复用  | 每次新 SQL |
| **场景** | 参数值（默认） | 动态表名/列名 |
### ${}合理场景
```
// ${} 不是完全不能用，用于"SQL 结构本身"的场景：

// ① 动态表名（表名不能预编译！）
SELECT * FROM ${tableName} WHERE id = #{id}
// 表名不能用 ? 占位（? 只能占"值"）

// ② 动态列名（排序字段）
SELECT * FROM user ORDER BY ${sortColumn} ${sortOrder}
// ORDER BY 的列名不能预编译

// ③ 动态拼接 IN（虽然不推荐）
// 注：IN 一般用 <foreach> 生成 #{}
```
${}的安全使用
```
// 用 ${} 时必须校验（白名单）
public List<User> findByTable(String tableName) {
    // 白名单校验（防止注入）
    if (!ALLOWED_TABLES.contains(tableName)) {
        throw new IllegalArgumentException("非法表名");
    }
    return userMapper.selectByTable(tableName);
}
```
### 动态SQL中的参数
```
// <foreach> 生成 IN 用的也是 #{}：
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id"
             open="(" separator="," close=")">
        #{id}   <!-- 用 #{}，安全 -->
    </foreach>
</select>
// 生成：SELECT * FROM user WHERE id IN (?, ?, ?)
// 参数安全设置 ✅（不是字符串拼接）

// <if> 判断：
<if test="name != null">
    AND name = #{name}
</if>
// test 用 OGNL 表达式，值是 #{}
```
### 面试高频题
#### 题目1：#{}和${}的区别
```
// #{}：预编译占位符（?），防注入
// ${}：字符串拼接，有注入风险
```
#### 题目2：#{}为什么安全
```
// 参数作为"值"传给数据库（预编译）
// 不被解析为 SQL 语法
```
