## 工作原理
启动流程->执行流程->四大处理器->完整串联->高频面试题
### 启动流程
```
//两个阶段
// MyBatis 运行分两个阶段：
// ① 初始化阶段：解析配置，构建 SqlSessionFactory（启动时一次）
// ② 执行阶段：SqlSession 执行 SQL（每次操作）

// 理解：初始化"搭台子"，执行"唱戏"
```
流程
```
① 读取 mybatis-config.xml（或 Java 配置）
    ↓
② 创建 SqlSessionFactoryBuilder
    ↓
③ XMLConfigBuilder 解析配置
    ↓
④ 构建 Configuration 对象
    - 解析 <properties>（数据源配置）
    - 解析 <settings>（全局设置）
    - 解析 <typeAliases>（类型别名）
    - 解析 <plugins>（插件）
    - 解析 <environments>（环境、数据源）
    - 解析 <mappers>（Mapper 注册）
    ↓
⑤ 解析每个 Mapper（XML 或注解）
    - 每个 SQL → MappedStatement
    - 存入 Configuration.mappedStatements
    ↓
⑥ 构建 SqlSessionFactory（持有 Configuration）
```
关键产物
```
// Configuration：MyBatis 的"大脑"（全局配置）
public class Configuration {
    // 数据源
    protected Environment environment;
    // MappedStatement 注册表
    protected final Map<String, MappedStatement> mappedStatements = ...;
    // Mapper 接口注册
    protected final Map<Class<?>, MapperProxyFactory<?>> knownMappers = ...;
    // 插件链
    protected final InterceptorChain interceptorChain;
    // 缓存
    protected final Map<String, Cache> caches;
    // ...
}

// 所有配置、SQL、Mapper 都在 Configuration 里
// SqlSessionFactory 就是持有它的工厂
```
### 执行阶段
完整流程
```
① 应用调用 mapper.findById(1L)（动态代理）
    ↓
② MapperProxy.invoke（JDK 动态代理）
    - 根据方法找到 MappedStatement 的 id
    ↓
③ SqlSession.selectOne(statementId, param)
    - 从 Configuration 取 MappedStatement
    ↓
④ Executor.query（执行器）
    - 查缓存（一级/二级）
    - 未命中 → 真正查询
    ↓
⑤ 创建 StatementHandler
    - 创建 PreparedStatement
    - 拼 SQL（动态 SQL）
    ↓
⑥ ParameterHandler 设置参数
    - 把参数设置到 PreparedStatement 的占位符
    ↓
⑦ JDBC 执行 SQL
    ↓
⑧ ResultSetHandler 映射结果
    - ResultSet → Java 对象（列 → 属性）
    ↓
⑨ 返回结果（并写入缓存）
```
### 四大处理器
```
// Executor 之后，MyBatis 用"四大处理器"完成 SQL 执行：

// ① StatementHandler（语句处理器）
// 职责：创建 Statement、拼 SQL、执行
// 实现：
// - SimpleStatementHandler（普通）
// - PreparedStatementHandler（预编译，默认）
// - CallableStatementHandler（存储过程）

// ② ParameterHandler（参数处理器）
// 职责：把 Java 参数设置到 SQL 占位符
// 实现：DefaultParameterHandler
// 处理：#{} → ?（PreparedStatement 的 setXxx）

// ③ ResultSetHandler（结果处理器）
// 职责：把 ResultSet 映射成 Java 对象
// 实现：DefaultResultSetHandler
// 处理：列名 → 属性名（resultMap 规则）

// ④ Executor（执行器）
// 职责：调度上述三个处理器 + 缓存
// 实现：Simple/Reuse/Batch + CachingExecutor
```
关系
```
Executor（总调度）
    ├── 创建 StatementHandler
    ├── StatementHandler 用 ParameterHandler 设参数
    ├── StatementHandler 用 ResultSetHandler 映射结果
    └── 管理缓存
```
### 完整串联
```
// 简化版源码流程：

// ① 代理调用
public Object invoke(Object proxy, Method method, Object[] args) {
    // 找到 MappedStatement
    String statementId = mapperInterface.getName() + "." + method.getName();
    // 执行
    return sqlSession.selectOne(statementId, args[0]);
}

// ② SqlSession
public <T> T selectOne(String statement, Object parameter) {
    List<T> list = this.selectList(statement, parameter);
    return list.size() == 1 ? list.get(0) : null;
}

// ③ Executor.query
public <E> List<E> query(MappedStatement ms, Object parameter, ...) {
    // 查缓存
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
        return list;
    }
    // 没缓存 → 查数据库
    list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    return list;
}

// ④ queryFromDatabase → doQuery
protected <E> List<E> doQuery(MappedStatement ms, ...) {
    StatementHandler handler = configuration.newStatementHandler(...);
    // ① 创建 Statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    // ② 设置参数 + 执行 + 映射结果（三大处理器）
    return handler.query(stmt, resultHandler);
}
```
### 面试高频题
#### 题目1：Mybaits工作原理
```
// 初始化：解析配置 → 构建 Configuration → SqlSessionFactory
// 执行：代理 → Executor → 四大处理器 → JDBC
```
#### 题目2：四大处理器
```
// Executor、StatementHandler、ParameterHandler、ResultSetHandler
```
#### 题目3：初始化阶段做了什么
```
// 解析配置 + 每个 SQL 转 MappedStatement + 注册 Mapper
// 存入 Configuration
```
#### 题目4：执行流程阶段
```
// 代理 → MappedStatement → Executor → StatementHandler
// → ParameterHandler → JDBC → ResultSetHandler
```
#### 题目5：Configuration是什么
```
// MyBatis 全局配置（大脑）
// 数据源、MappedStatement、Mapper、插件、缓存
```
#### 题目6：插件拦截在哪
```
// 四大对象可被拦截（Executor/StatementHandler/ParameterHandler/ResultSetHandler）
```
