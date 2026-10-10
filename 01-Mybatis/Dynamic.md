## 动态SQL
动态SQL是什么->常用标签->if/where/set->choose->foreach->面试高频题
### 动态SQL是什么
动态SQL：根据参数条件，动态拼接不同的SQL语句。
```
// 需求：多条件查询（条件可选）
// 用户可能传 name、也可能传 age、也可能都不传

// ❌ 手写拼接（繁琐、易错）：
String sql = "SELECT * FROM user WHERE 1=1";
if (name != null) sql += " AND name = '" + name + "'";  // 注入风险！
if (age != null) sql += " AND age = " + age;

// ✅ 动态 SQL 标签：
<select id="findUsers">
    SELECT * FROM user
    <where>
        <if test="name != null">AND name = #{name}</if>
        <if test="age != null">AND age = #{age}</if>
    </where>
</select>
```
### 常用标签总览
| 标签                            | 作用                    |
| ----------------------------- | --------------------- |
| `<if>`                        | 条件判断                  |
| `<where>`                     | 自动处理 WHERE + 去掉多余 AND |
| `<set>`                       | 自动处理 SET + 去掉多余逗号     |
| `<choose>/<when>/<otherwise>` | 多分支选择                 |
| `<foreach>`                   | 遍历集合（IN 查询、批量）        |
| `<trim>`                      | 自定义前缀/后缀处理            |
| `<bind>`                      | 变量绑定                  |
### if/where/set
if 条件判断
```xml
<select id="findUsers" resultType="User">
    SELECT * FROM user
    WHERE
    <if test="name != null and name != ''">
        name = #{name}
    </if>
    <if test="age != null">
        AND age = #{age}
    </if>
</select>
<!-- 问题：如果 name 为空，SQL 变成 "WHERE AND age = ?" → 语法错误！ -->
```
where 解决AND问题
```
<select id="findUsers" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null">AND name = #{name}</if>
        <if test="age != null">AND age = #{age}</if>
    </where>
</select>
<!-- where 标签会：
     ① 有内容时自动加 WHERE
     ② 去掉开头多余的 AND/OR
     结果：SELECT * FROM user WHERE name = ? AND age = ? ✅
     若都为空：SELECT * FROM user（没有 WHERE）✅ -->
```
set 更新时自动处理逗号
```
<update id="updateUser">
    UPDATE user
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>
<!-- set 标签会：
     ① 自动加 SET
     ② 去掉末尾多余的逗号
     结果：UPDATE user SET name = ?, age = ? WHERE id = ? ✅
     若只改 name：UPDATE user SET name = ? WHERE id = ? ✅ -->
```
### choose 多分支
```
<!-- 类似 Java 的 if-else if-else（只选一个） -->
<select id="findUsers" resultType="User">
    SELECT * FROM user
    <where>
        <choose>
            <when test="name != null">
                AND name = #{name}
            </when>
            <when test="age != null">
                AND age = #{age}
            </when>
            <otherwise>
                AND status = 1   <!-- 都不满足时 -->
            </otherwise>
        </choose>
    </where>
</select>
<!-- choose：只匹配第一个满足的 when（互斥）
     和 if 的区别：多个 if 可以都满足，choose 只选一个 -->
```
### foreach
IN查询
```
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id"
             open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
<!-- collection：集合参数名
     item：每次迭代的元素
     open：开始符号 ( 
     separator：分隔符 ,
     close：结束符号 )
     生成：SELECT * FROM user WHERE id IN (?, ?, ?) ✅ -->
```
批量插入
```
<insert id="batchInsert">
    INSERT INTO user (name, age) VALUES
    <foreach collection="users" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>
<!-- 生成：INSERT INTO user (name, age) VALUES (?, ?), (?, ?), (?, ?) -->
```
foreach属性

|属性|说明|
|---|---|
|**collection**|集合参数（List/array/Map）|
|**item**|迭代变量（每个元素）|
|**index**|List 是索引，Map 是 key|
|**open**|开始符号|
|**close**|结束符号|
|**separator**|分隔符|
collection 参数名
```
// collection 的值取决于参数怎么传：

// ① List 直接传：collection="list"
public void method(List<Long> ids);
// 或 collection="collection"

// ② 数组：collection="array"
public void method(Long[] ids);

// ③ @Param 指定名：
public void method(@Param("ids") List<Long> ids);
// collection="ids"（推荐，明确）

// ④ 对象属性：
public void method(UserQuery query);  // query 有 ids 属性
// collection="ids"
```
### trim/bind
trim 自定义处理
```
<!-- where/set 本质就是 trim 的特例 -->
<trim prefix="WHERE" prefixOverrides="AND |OR ">
    <if test="name != null">AND name = #{name}</if>
</trim>
<!-- 等价于 <where> -->

<trim prefix="SET" suffixOverrides=",">
    <if test="name != null">name = #{name},</if>
</trim>
<!-- 等价于 <set> -->
```
bind 变量绑定
```
<select id="findUsers">
    <!-- 绑定模糊查询的 % -->
    <bind name="pattern" value="'%' + name + '%'" />
    SELECT * FROM user WHERE name LIKE #{pattern}
</select>
```
### 面试高频题
#### 题目1：动态SQL有哪些标签
```
// if、where、set、choose、foreach、trim、bind
```
#### 题目2：where和set的作用
```
// where：自动加 WHERE + 去多余 AND
// set：自动加 SET + 去多余逗号
```
#### 题目3：if和choose区别
```
// if：可多个都满足（并列）
// choose：只选一个（互斥，类似 if-else）
```
#### 题目4：foreach怎么用
```
// IN 查询、批量插入
// collection/item/open/separator/close
```
#### 题目5：where不加会怎么样
```
// 条件都满足时拼出 "WHERE AND ..." → 语法错误
// where 标签去掉开头的 AND
```
