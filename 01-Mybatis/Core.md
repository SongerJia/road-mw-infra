## 核心组件
Mybatis是什么->四大组件->SqlSessionFactory->SqlSession->Executor->MappedStatement->面试高频题
### Mybatis是什么
Mybatis：半自动的持久层框架。SQL由开发者手写，Mybatis负责参数映射、SQL执行和结果映射。
为什么叫半自动
```
// 全自动（Hibernate/JPA）：框架生成 SQL
userRepository.findByName("张三");  // 框架生成 SQL

// 半自动（MyBatis）：开发者写 SQL
@Select("SELECT * FROM user WHERE name = #{name}")
User findByName(String name);   // SQL 自己写
// 好处：SQL 可控（能优化）、灵活
```
### 四大核心组件
| 组件                    | 作用                       |
| --------------------- | ------------------------ |
| **SqlSessionFactory** | 工厂：创建 SqlSession（全局一个）   |
| **SqlSession**        | 会话：执行 SQL、管理事务、获取 Mapper |
| **Executor**          | 执行器：真正执行 SQL（含缓存策略）      |
| **MappedStatement**   | SQL 封装：一条 SQL 的完整信息      |
```Java
// 构建 Factory（应用启动时一次）
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);

// 获取 Session（每次操作一个）
try (SqlSession session = factory.openSession()) {
    // 获取 Mapper（动态代理）
    UserMapper mapper = session.getMapper(UserMapper.class);
    User user = mapper.findById(1L);   // 执行 SQL
    session.commit();  // 提交
}
```
### SqlSessionFactory
```
// 职责：读取配置、创建 SqlSession
// 特点：全局唯一（单例）、线程安全、重量级

// 构建方式：
// ① 基于 XML 配置
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);

// ② 基于 Java API
Configuration configuration = new Configuration();
configuration.addMapper(UserMapper.class);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(configuration);

// 内部关键：Configuration（全局配置对象）
// 包含：数据源、Mapper 注册、类型处理器、插件等

// 为什么全局一个：
// 创建成本高（解析配置、注册 Mapper）
// 线程安全（构造后不可变）
```
### SqlSession
```
// 职责：一次数据库会话（执行 SQL、事务、Mapper）
// 特点：非线程安全（每次操作新建/从模板获取）

// 主要方法：
public interface SqlSession extends Closeable {
    <T> T selectOne(String statement);          // 查一条
    <T> List<T> selectList(String statement);   // 查多条
    int insert(String statement);               // 插入
    int update(String statement);               // 更新
    int delete(String statement);               // 删除
    void commit();                              // 提交
    void rollback();                            // 回滚
    <T> T getMapper(Class<T> type);             // 获取 Mapper（代理）
    void close();                               // 关闭
}

// 为什么非线程安全：
// SqlSession 对应一个数据库连接（Connection）
// 多线程共享 → 事务混乱 → 非线程安全

// 与 Spring 集成后：
// SqlSessionTemplate（线程安全，内部 ThreadLocal 绑定 Session）
```
和Spring集成
```
// Spring 集成后不用手动 openSession
// SqlSessionTemplate（线程安全）管理
// 每个线程有自己的 SqlSession（ThreadLocal）

@Autowired
private UserMapper userMapper;  // 直接注入 Mapper（自动管理 Session）
userMapper.findById(1L);
```
### Executor 执行器
```
// 职责：真正执行 SQL（含缓存、批处理策略）
// 三种实现：

// ① SimpleExecutor（默认）：每次执行新 Statement
// ② ReuseExecutor：复用 Statement（相同 SQL 复用）
// ③ BatchExecutor：批量执行（批量更新优化）

// 装饰器（缓存）：
// CachingExecutor（二级缓存装饰）
// BaseExecutor（一级缓存，所有 Executor 的基类）

// 执行流程：
// Executor → StatementHandler（创建 Statement）
// → ParameterHandler（参数设置）
// → ResultSetHandler（结果映射）
```
缓存分层
```
// Executor 内部有缓存：
// 一级缓存：BaseExecutor（Session 级，默认开启）
// 二级缓存：CachingExecutor（Namespace 级，需配置）

// 所以：Executor = 执行 + 缓存
```
### MappedStatement SQL封装
```
// 职责：封装一条 SQL 的完整信息
// 存储：Configuration 中（Map<String, MappedStatement>）

// 包含：
// ① id（namespace + method）
// ② sqlSource（SQL 语句）
// ③ resultMap（结果映射）
// ④ parameterMap（参数映射）
// ⑤ 缓存配置、超时、类型等

// 例：
// <select id="findById" resultType="User">
//   SELECT * FROM user WHERE id = #{id}
// </select>
// → 一个 MappedStatement

// key = "com.example.UserMapper.findById"
// 每次执行 SQL：根据 id 找 MappedStatement
```
四大组件协作
```
SqlSessionFactory（读配置，创建 Session）
    ↓
SqlSession（会话，执行接口）
    ↓
Executor（执行器，查缓存 + 执行）
    ↓
MappedStatement（找到具体 SQL）
    ↓
StatementHandler → ParameterHandler → ResultSetHandler
    ↓
JDBC 执行 → 结果
```
### 一次SQL的完整流程
```
① 应用调用 mapper.findById(1L)（动态代理）
    ↓
② 代理 → MapperProxy.invoke
    ↓
③ 找到 MappedStatement（by id）
    ↓
④ SqlSession → Executor.query
    ↓
⑤ 查一级缓存（命中直接返回）
    ↓
⑥ 查二级缓存（命中直接返回）
    ↓
⑦ 创建 StatementHandler（PreparedStatement）
    ↓
⑧ ParameterHandler 设置参数（#{} → ?）
    ↓
⑨ JDBC 执行 SQL
    ↓
⑩ ResultSetHandler 映射结果（列 → 属性）
    ↓
⑪ 写入缓存 → 返回结果
```
### 面试高频题
#### 题目1：Mybatis四大组件
```
// SqlSessionFactory、SqlSession、Executor、MappedStatement
```
#### 题目2：SqlSessionFactory vs SqlSession
```
// Factory：全局一个（线程安全，重量级）
// Session：每次一个（非线程安全，轻量）
```
#### 题目3：SqlSession线程安全吗
```
// 不安全！对应一个连接，多线程共享会事务混乱
// Spring 用 SqlSessionTemplate + ThreadLocal 解决
```
#### 题目4：Executor有几种
```
// Simple（默认）、Reuse（复用 Statement）、Batch（批量）
```
#### 题目5：MappedStatement是什么
```
// 一条 SQL 的封装（id、sqlSource、resultMap...）
// Configuration 中按 id 存
```

