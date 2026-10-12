## 与Spring集成
为什么需要集成->SqlSessionTemplate->Mapper注册->事务管理->面试高频题
### 为什么需要集成
```
// 原生 MyBatis 的问题：
SqlSession session = factory.openSession();
try {
    UserMapper mapper = session.getMapper(UserMapper.class);
    mapper.insert(user);
    session.commit();          // 手动提交
} catch (Exception e) {
    session.rollback();        // 手动回滚
} finally {
    session.close();           // 手动关闭
}
// 繁琐！每次都要管理 Session

// Spring 集成后：
@Autowired
private UserMapper userMapper;   // 直接注入

userMapper.insert(user);         // 事务由 Spring 管理
// 简洁！Session 自动管理、事务自动提交/回滚
```
### SqlSessionTemplate
```
// SqlSessionTemplate：线程安全的 SqlSession 代理
// 作用：管理 SqlSession 的生命周期（自动获取/关闭）

// 特点：
// ① 线程安全（可单例）
// ② 内部用 ThreadLocal 绑定 Session
// ③ 每次操作自动获取/提交/关闭

// 原理：
public class SqlSessionTemplate implements SqlSession {
    private final SqlSessionFactory sqlSessionFactory;

    @Override
    public <T> T getMapper(Class<T> type) {
        // 返回 Mapper 代理（结合 SqlSessionInterceptor）
        return getConfiguration().getMapper(type, this);
    }

    // 内部拦截器（每次操作）
    private class SqlSessionInterceptor implements InvocationHandler {
        public Object invoke(Object proxy, Method method, Object[] args) {
            // ① 获取 SqlSession（从事务管理器或新建）
            SqlSession sqlSession = getSqlSession(sqlSessionFactory);
            try {
                // ② 执行方法
                Object result = method.invoke(sqlSession, args);
                // ③ 事务提交（非事务时）
                if (!isSqlSessionTransactional(sqlSession, sqlSessionFactory)) {
                    sqlSession.commit(true);
                }
                return result;
            } finally {
                // ④ 关闭 Session
                closeSqlSession(sqlSession, sqlSessionFactory);
            }
        }
    }
}
```
ThreadLocal绑定
```
// Spring 事务同步：
// 每个线程的 SqlSession 存在 ThreadLocal（TransactionSynchronizationManager）
// 保证同一事务内用同一个 Session（缓存/事务一致）

// 无事务时：每次操作新 Session
// 有事务时：整个事务一个 Session
```
### Mapper注册 @MapperScan
```
// 启动类扫描 Mapper 接口：
@SpringBootApplication
@MapperScan("com.example.mapper")   // 扫描 Mapper 包
public class Application { }

// 原理：
// ① @MapperScan 导入 MapperScannerConfigurer
// ② 扫描指定包下的接口
// ③ 为每个接口注册 BeanDefinition（MapperFactoryBean）
// ④ 注入时返回 Mapper 代理（前面讲的 JDK 动态代理）
```
Mapper注册流程
```
@MapperScan 扫描包
    ↓
找到接口（UserMapper)
    ↓
注册 BeanDefinition（MapperFactoryBean）
    ↓
容器创建 Bean：
    MapperFactoryBean.getObject()
    → sqlSession.getMapper(UserMapper.class)
    → JDK 动态代理生成实现
    ↓
@Autowired 注入（拿到代理）
```
为什么不用@Mapper
```
// @Mapper：单个接口标注
@Mapper
public interface UserMapper { }

// @MapperScan：批量扫描（推荐）
@MapperScan("com.example.mapper")

// 区别：@MapperScan 一次扫所有，不用每个都标
```
### 事务管理
```
// Spring 事务 + MyBatis 的配合：
@Transactional   // Spring 事务
public void createOrder(Order order) {
    orderMapper.insert(order);       // MyBatis 操作
    stockMapper.deduct(order);       // 同一事务
    // 抛异常 → Spring 回滚（SqlSession 也回滚）
}
```
事务同步原理
```
// ① Spring 事务开启 → 绑定 Connection 到 ThreadLocal
// ② SqlSessionTemplate 获取 Session 时：
//    发现线程有事务 → 用事务的 Connection（不新建）
// ③ 全部操作共用同一个 Connection（同一事务）
// ④ 事务提交/回滚 → SqlSession 同步提交/回滚

// 关键：SpringManagedTransaction
// 把 MyBatis 的 Connection 交给 Spring 事务管理器
```
没有@Transactional会怎样
```
// 没有事务注解：
userMapper.insert(user);   // 立即自动提交（autoCommit）
// 每次操作独立提交（无法回滚多个操作）

// 有事务：
@Transactional
public void method() {
    userMapper.insert(user1);
    userMapper.insert(user2);   // 两个一起成功/失败
}
```
### Spring Boot自动配置
```
# Spring Boot 自动配置 MyBatis
mybatis:
  mapper-locations: classpath:mapper/*.xml   # Mapper XML 位置
  type-aliases-package: com.example.entity   # 别名包
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: true
```
```
// 自动配置类（MybatisAutoConfiguration）：
// ① 检测 DataSource
// ② 创建 SqlSessionFactory（读配置）
// ③ 创建 SqlSessionTemplate
// ④ 注册 MapperScannerConfigurer
// @MapperScan 或 @Mapper 触发 Mapper 注册

// 引入 starter：
// <artifactId>mybatis-spring-boot-starter</artifactId>
```
### 面试高频题
#### 题目1：MyBatis和Spring怎么集成
```
// SqlSessionTemplate（线程安全）+ 自动配置
// Mapper 自动注册 + 事务同步
```
#### 题目2：SqlSessionTemplate作用
```
// 线程安全的 Session 代理
// 管理 Session 生命周期（自动获取/关闭）
```
#### 题目3：Mapper怎么注册
```
// @MapperScan 扫描接口
// 注册 MapperFactoryBean
// getObject 返回 JDK 动态代理
```
#### 题目4：事务怎么同步
```
// Spring 事务绑定 Connection 到 ThreadLocal
// SqlSession 复用同一 Connection
// 提交/回滚同步
```
#### 题目5：为什么SqlSession线程安全
```
// 原生 SqlSession 非线程安全
// SqlSessionTemplate 用 ThreadLocal 隔离
// 每个线程独立 Session
```
