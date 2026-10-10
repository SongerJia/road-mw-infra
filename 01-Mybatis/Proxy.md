## 接口代理
为什么能直接调接口->动态代理原理->MapperProxy源码->方法找不到问题->面试高频题
### Mapper为什么能直接调
```
// Mapper 接口没有实现类，为什么能调用？

public interface UserMapper {
    User findById(Long id);   // 没有实现！
}

// 直接调用：
@Autowired
private UserMapper userMapper;
userMapper.findById(1L);   // 居然能执行 SQL！
```
Jdk动态代理
```
// MyBatis 用 JDK 动态代理，运行时生成实现类
// 每个 Mapper 接口 → 一个代理对象
// 调用方法 → 代理拦截 → 执行对应 SQL

// 印证：Mapper 必须是接口（JDK 代理只能代理接口）
// （对比：CGLIB 代理类，但 MyBatis 用 JDK）
```
### 动态代理原理
关键组件
```
// ① MapperProxyFactory（代理工厂）
public class MapperProxyFactory<T> {
    private final Class<T> mapperInterface;

    // 创建代理对象
    public T newInstance(SqlSession sqlSession) {
        MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }

    // JDK 动态代理生成
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(
            mapperInterface.getClassLoader(),
            new Class[]{mapperInterface},   // 代理的接口
            mapperProxy                       // InvocationHandler
        );
    }
}

// ② MapperProxy（调用处理器）
public class MapperProxy<T> implements InvocationHandler {
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 拦截方法调用
        // ① Object 的方法（toString/hashCode）→ 直接执行
        // ② default 方法 → 直接执行
        // ③ 其他 → 交给 MapperMethod 执行 SQL

        // 缓存 MethodInvoker（避免重复解析）
        MapperMethodInvoker invoker = methodCache.computeIfAbsent(...);
        return invoker.invoke(proxy, method, args, sqlSession);
    }
}

// ③ MapperMethod（方法映射）
public class MapperMethod {
    private final SqlCommand command;    // 命令类型（SELECT/INSERT/UPDATE/DELETE）
    private final MethodSignature method; // 方法签名（参数、返回值）

    public Object execute(SqlSession sqlSession, Object[] args) {
        switch (command.getType()) {
            case SELECT:
                return executeQuery(sqlSession, args);
            case INSERT:
                return executeUpdate(sqlSession, args);  // rowCount
            // ...
        }
    }
}
```
流程
```
userMapper.findById(1L)
    ↓
MapperProxy.invoke（JDK 代理拦截）
    ↓
找 MapperMethod（缓存）
    ↓
MapperMethod.execute
    ↓
判断命令类型（SELECT）→ sqlSession.selectOne
    ↓
Executor 执行 SQL（前面的流程）
```
### 方法怎么映射到SQL
```
// 接口方法 → SQL 的映射规则：
// ① statementId = 接口全限定名 + "." + 方法名
//    例：com.example.UserMapper.findById
// ② 从 Configuration.mappedStatements 找对应 MappedStatement

// 接口方法：
public interface UserMapper {
    User findById(Long id);
}

// XML：
// <mapper namespace="com.example.UserMapper">
//   <select id="findById" resultType="User">
//     SELECT * FROM user WHERE id = #{id}
//   </select>
// </mapper>
// → MappedStatement id = "com.example.UserMapper.findById"

// 调用 findById → 拼 id → 找 MappedStatement → 执行
```
为什么用namespace
```
// namespace 绑定接口全限定名
// 保证不同 Mapper 的同名方法不冲突（id 唯一）
// UserMapper.findById vs OrderMapper.findById（不同 namespace）
```
### 源码细节
①Object方法不代理
```
// MapperProxy.invoke 首先判断：
if (method.getDeclaringClass() == Object.class) {
    return method.invoke(this, args);  // toString/hashCode/equals 直接执行
}
// 因为 Object 方法不是 SQL 方法
```
②Default方法不代理
```
// 接口的 default 方法直接执行（Java 8+）
if (method.isDefault()) {
    return invokeDefaultMethod(proxy, method, args);
}
// 因为 default 方法有实现，不需要走 SQL
```
③方法缓存
```
// MethodInvoker 缓存（避免每次反射解析）
private final Map<Method, MapperMethodInvoker> methodCache;
// 第一次调用解析，后续复用
```
### 和动态代理的关联
```
// MyBatis Mapper = JDK 动态代理应用

// 对比 Spring AOP：
// Spring AOP：代理 Bean，加切面（增强）
// MyBatis Mapper：代理接口，执行 SQL（实现）

// 相同点：
// ① 都用 JDK 动态代理
// ② 都通过 InvocationHandler 拦截

// 不同点：
// ① AOP 有目标对象（增强目标）
// ② MyBatis 无目标对象（接口本身就是"目标"，执行 SQL）

// 面试串联：
// "MyBatis Mapper 用 JDK 动态代理生成接口实现，
//  调用方法 → 拦截 → 根据方法名找 MappedStatement → 执行 SQL。
//  和 Spring AOP 一样用 JDK 代理，但 MyBatis 没有目标对象。"
```
### 面试高频题
#### 题目1：Mapper接口为什么能调用
```
// JDK 动态代理生成实现
// 接口本身无实现，代理负责执行 SQL
```
#### 题目2：用的什么代理
```
// JDK 动态代理（不是 CGLIB）
// 所以 Mapper 必须是接口
```
#### 题目3：方法怎么映射SQL
```
// namespace + 方法名 = statementId
// 按 id 找 MappedStatement
```
#### 题目4：MapperProxy做什么
```
// InvocationHandler 拦截调用
// 区分 Object/default/SQL 方法
// 交给 MapperMethod 执行
```
#### 题目5：namespace作用
```
// 绑定接口全限定名
// 保证 statementId 唯一（避免冲突）
```
#### 题目6：和Spring AOP的区别
```
// 都用 JDK 代理
// AOP：增强目标对象；Mapper：无目标，执行 SQL
```

