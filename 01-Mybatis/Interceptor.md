## 插件机制
插件是什么->四大对象->拦截原理->自定义插件->分页插件->面试高频题
### 插件是什么
Interceptor：MyBatis的扩展机制，通过拦截四大对象的方法，实现功能增强，分页、SQL监控和数据脱敏等。
```
// 类比：AOP 切面
// Spring AOP：拦截 Bean 方法，加切面
// MyBatis 插件：拦截四大对象方法，加功能

// 原理相同：动态代理 + 责任链
```
### 四大拦截对象
| 对象                   | 作用    | 可拦截方法                                   |
| -------------------- | ----- | --------------------------------------- |
| **Executor**         | 执行器   | update、query、commit、rollback            |
| **StatementHandler** | 语句处理器 | prepare、parameterize、batch、update、query |
| **ParameterHandler** | 参数处理器 | getParameterObject、setParameters        |
| **ResultSetHandler** | 结果处理器 | handleResultSets、handleOutputParameters |
```
// @Intercepts 指定拦截哪类方法：
@Intercepts({
    @Signature(
        type = Executor.class,           // 拦截 Executor
        method = "query",                // 方法名
        args = {MappedStatement.class, Object.class, RowBounds.class,
                ResultHandler.class}     // 参数类型（决定重载）
    )
})
```
### 拦截原理 动态代理+责任链
插件链创建
```
// ① Configuration 中注册插件（InterceptorChain）
public class InterceptorChain {
    private final List<Interceptor> interceptors = new ArrayList<>();

    // 用插件包装目标对象（责任链）
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);   // 层层包装
        }
        return target;
    }
}

// ② 四大对象创建时被包装：
// Executor 创建：
executor = (Executor) interceptorChain.pluginAll(executor);
// → 每个插件包一层代理

// ③ Interceptor.plugin（默认实现）：
default Object plugin(Object target) {
    return Plugin.wrap(target, this);   // 用 JDK 动态代理包装
}
```
Plugin.wrap JDK动态代理
```
public class Plugin implements InvocationHandler {
    private final Object target;        // 被代理对象
    private final Interceptor interceptor;  // 插件
    private final Map<Class<?>, Set<Method>> signatureMap;  // 要拦截的方法

    public static Object wrap(Object target, Interceptor interceptor) {
        // 解析 @Intercepts，找出要拦截的方法
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
        // 创建 JDK 动态代理
        return Proxy.newProxyInstance(
            type.getClassLoader(),
            getInterfaces(type),   // 目标的所有接口
            new Plugin(target, interceptor, signatureMap)
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 判断当前方法是否在拦截列表
        Set<Method> methods = signatureMap.get(method.getDeclaringClass());
        if (methods != null && methods.contains(method)) {
            // 拦截 → 交给插件的 intercept
            return interceptor.intercept(new Invocation(target, method, args));
        }
        // 不拦截 → 直接执行
        return method.invoke(target, args);
    }
}
```
拦截流程
```
调用 executor.query(...)
    ↓
Plugin.invoke（动态代理拦截）
    ↓
判断是否要拦截该方法
    ├── 是 → interceptor.intercept(invocation)
    │         → 插件逻辑（前置）
    │         → invocation.proceed()（执行原方法）
    │         → 插件逻辑（后置）
    └── 否 → method.invoke(target, args)
```
### 自定义插件
```
// 需求：SQL 耗时监控
@Intercepts({
    @Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
@Component
public class SqlCostInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            // 执行原方法（放行）
            return invocation.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;
            if (cost > 1000) {
                // 慢 SQL 告警
                MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
                log.warn("慢 SQL: {} 耗时 {}ms", ms.getId(), cost);
            }
        }
    }

    @Override
    public Object plugin(Object target) {
        // 用默认实现（Plugin.wrap）
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        // 插件配置属性（可选）
    }
}
```
其他插件场景
```
// ① 分页插件（PageHelper）
// 拦截 Executor.query，改写 SQL 加 LIMIT
// ② SQL 监控（记录执行时间）
// ③ 数据脱敏（拦截结果，脱敏敏感字段）
// ④ 多租户（拦截 SQL，加租户条件）
// ⑤ 读写分离（拦截 SQL，路由到主/从库）
```
### 分页插件 PageHelper
```
// PageHelper 就是基于 MyBatis 插件实现的

// 使用：
PageHelper.startPage(1, 10);   // 第 1 页，每页 10 条
List<User> users = userMapper.findAll();  // 自动分页
PageInfo<User> pageInfo = new PageInfo<>(users);

// 原理：
// ① 拦截 Executor.query
// ② 从 ThreadLocal 拿分页参数（startPage 存入）
// ③ 改写 SQL（加 LIMIT）
// ④ 执行分页查询 + 查总数

// 源码流程：
// ① PageInterceptor.intercept
// ② 判断是否需要分页（ThreadLocal 有分页参数？）
// ③ 改写 SQL：
//    SELECT * FROM user → SELECT * FROM user LIMIT ?, ?
// ④ 执行
// ⑤ 包装结果（Page 对象含总数）
```
分页两种方式

|方式|SQL|性能|
|---|---|---|
|**物理分页（PageHelper）**|LIMIT ?|✅ 快（数据库分页）|
|**逻辑分页（RowBounds）**|查全部 + 内存截取|❌ 慢（全查）|
### 高频面试题
#### 题目1：MyBatis插件能拦截什么
```
// 四大对象：Executor、StatementHandler、
// ParameterHandler、ResultSetHandler
```
#### 题目2：插件原理
```
// 动态代理（Plugin.wrap）+ 责任链
// 拦截指定方法，包裹增强
```
#### 题目3：插件能拦截哪些方法
```
// @Intercepts + @Signature 指定类和方法的签名
```
#### 题目4：PageHelper原理
```
// 拦截 Executor.query
// ThreadLocal 拿分页参数
// 改写 SQL 加 LIMIT
```
#### 题目5：多个插件的顺序
```
// 责任链：配置顺序，层层包装
// 先配置的在外层（先执行）
```
#### 题目6：插件和AOP的区别
```
// 都用动态代理
// AOP 拦 Bean，插件拦 MyBatis 四大对象
```
