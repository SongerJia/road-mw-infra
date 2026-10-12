## 手写MyBatis
为什么手写->核心目标->整体架构->关键实现->面试怎么描述
### 为什么手写
```
// 面试考察点：
// "能手写一个简易 ORM 框架吗？"
// 考察：反射、动态代理、注解、JDBC 的理解深度

// 价值：
// ① 理解 MyBatis 原理（不再是黑盒）
// ② 展示底层功底（反射+代理+JDBC）
// ③ 面试加分（体现深度）
```
### 核心目标
```
// 实现一个最简版 MyBatis，支持：
// ① 自定义注解（@Select / @Insert）
// ② Mapper 接口（无实现，动态代理）
// ③ 参数注入（#{id} → ?）
// ④ 结果映射（ResultSet → 对象）
// ⑤ SqlSession（执行 + 事务）

// 目标：能这样用
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(Long id);

    @Insert("INSERT INTO user(name, age) VALUES (#{name}, #{age})")
    int insert(User user);
}

// 前端调用
SqlSession session = new SqlSessionFactoryBuilder().build().openSession();
UserMapper mapper = session.getMapper(UserMapper.class);
User user = mapper.findById(1L);
```
### 整体架构
```
① 配置阶段：
   扫描 Mapper 接口 → 解析注解（@Select 等）
   → 封装成 MappedStatement → 存入 Configuration

② 执行阶段：
   getMapper → JDK 动态代理 → 方法调用拦截
   → 找 MappedStatement → 解析 SQL
   → 参数注入 → JDBC 执行 → 结果映射

核心组件（对应真 MyBatis）：
① Configuration（配置）
② MappedStatement（SQL 封装）
③ SqlSession（会话）
④ Executor（执行器）
⑤ 动态代理（MapperProxy）
⑥ 参数处理器 + 结果处理器
```
### 关键实现
①自定义注解
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Select {
    String value();   // SQL 语句
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Insert {
    String value();
}
```
②MappedStatement SQL封装
```
public class MappedStatement {
    private String sql;              // SQL 语句
    private Class<?> resultType;     // 返回类型
    private Class<?> parameterType;  // 参数类型

    // getter/setter
}
```
③Configuration 配置
```
public class Configuration {
    // statementId → MappedStatement
    private final Map<String, MappedStatement> statementMap = new HashMap<>();

    public void addMappedStatement(String id, MappedStatement ms) {
        statementMap.put(id, ms);
    }

    public MappedStatement getMappedStatement(String id) {
        return statementMap.get(id);
    }
}
```
④配置解析 扫描注解
```
public class MapperParser {
    public void parse(Class<?> mapperInterface, Configuration config) {
        for (Method method : mapperInterface.getMethods()) {
            // 解析 @Select
            if (method.isAnnotationPresent(Select.class)) {
                Select select = method.getAnnotation(Select.class);
                MappedStatement ms = new MappedStatement();
                ms.setSql(select.value());
                ms.setResultType(method.getReturnType());
                ms.setParameterType(method.getParameterTypes()[0]);
                // statementId = 接口全限定名 + 方法名
                config.addMappedStatement(
                    mapperInterface.getName() + "." + method.getName(), ms);
            }
            // 解析 @Insert 类似...
        }
    }
}
```
⑤Mapper动态代理
```
public class MapperProxy implements InvocationHandler {
    private final Configuration config;

    public MapperProxy(Configuration config) {
        this.config = config;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // ① 找 MappedStatement
        String statementId = method.getDeclaringClass().getName() + "." + method.getName();
        MappedStatement ms = config.getMappedStatement(statementId);

        if (ms == null) {
            // 不是 SQL 方法（如 Object 方法）
            return method.invoke(proxy, args);
        }

        // ② 判断语句类型
        if (ms.getSql().trim().toUpperCase().startsWith("SELECT")) {
            return executeQuery(ms, args);   // 查询
        } else {
            return executeUpdate(ms, args);  // 增删改
        }
    }

    // 创建代理
    public static <T> T newMapperProxy(Class<T> mapperInterface, Configuration config) {
        return (T) Proxy.newProxyInstance(
            mapperInterface.getClassLoader(),
            new Class[]{mapperInterface},
            new MapperProxy(config)
        );
    }
}
```
⑥执行引擎（JDBC+参数+结果）
```
public class Executor {
    private final Connection connection;
    private final Configuration config;

    public Executor(Connection connection, Configuration config) {
        this.connection = connection;
        this.config = config;
    }

    // 查询
    public <T> T query(MappedStatement ms, Object[] args) throws Exception {
        // ① SQL 解析：#{id} → ?
        String sql = parseSql(ms.getSql());

        // ② 预编译
        PreparedStatement ps = connection.prepareStatement(sql);

        // ③ 参数注入
        setParameters(ps, args);

        // ④ 执行
        ResultSet rs = ps.executeQuery();

        // ⑤ 结果映射
        return mapResult(rs, ms.getResultType());
    }

    // SQL 解析（#{} → ?）
    private String parseSql(String sql) {
        // 正则替换 #{xxx} → ?
        return sql.replaceAll("#\\{[^}]+}", "?");
    }

    // 参数注入
    private void setParameters(PreparedStatement ps, Object[] args) throws Exception {
        for (int i = 0; i < args.length; i++) {
            ps.setObject(i + 1, args[i]);   // 简化：直接 setObject
        }
        // 真 MyBatis 会解析参数名、按类型 setXxx
    }

    // 结果映射（反射）
    private <T> T mapResult(ResultSet rs, Class<T> resultType) throws Exception {
        if (!rs.next()) return null;

        // 反射创建对象
        T instance = resultType.getDeclaredConstructor().newInstance();

        // 遍历列，反射设置字段
        ResultSetMetaData meta = rs.getMetaData();
        for (int i = 1; i <= meta.getColumnCount(); i++) {
            String columnName = meta.getColumnName(i);   // 列名
            Object value = rs.getObject(i);

            // 列名 → 属性名（驼峰转换）
            String fieldName = toCamelCase(columnName);

            // 反射设置字段
            try {
                Field field = resultType.getDeclaredField(fieldName);
                field.setAccessible(true);
                field.set(instance, value);
            } catch (NoSuchFieldException e) {
                // 忽略不存在的字段
            }
        }
        return instance;
    }

    private String toCamelCase(String columnName) {
        // user_name → userName
        StringBuilder sb = new StringBuilder();
        boolean upper = false;
        for (char c : columnName.toLowerCase().toCharArray()) {
            if (c == '_') {
                upper = true;
            } else {
                sb.append(upper ? Character.toUpperCase(c) : c);
                upper = false;
            }
        }
        return sb.toString();
    }
}
```
⑦SqlSession 会话
```
public class SqlSession {
    private final Configuration config;
    private final Connection connection;

    public SqlSession(Configuration config, Connection connection) {
        this.config = config;
        this.connection = connection;
    }

    // 获取 Mapper（动态代理）
    public <T> T getMapper(Class<T> mapperInterface) {
        return MapperProxy.newMapperProxy(mapperInterface, config);
    }

    // 执行（代理内部调用）
    public <T> T selectOne(MappedStatement ms, Object[] args) throws Exception {
        return new Executor(connection, config).query(ms, args);
    }

    public int update(MappedStatement ms, Object[] args) throws Exception {
        return new Executor(connection, config).update(ms, args);
    }

    public void commit() throws Exception {
        connection.commit();
    }

    public void rollback() throws Exception {
        connection.rollback();
    }
}
```
⑧工厂 构建入口
```
public class SqlSessionFactory {
    private final Configuration config;

    public SqlSessionFactory(Configuration config) {
        this.config = config;
    }

    public SqlSession openSession() throws Exception {
        // 获取连接
        Connection conn = DriverManager.getConnection(URL, USER, PWD);
        conn.setAutoCommit(false);   // 手动提交
        return new SqlSession(config, conn);
    }
}

// 构建（扫描 Mapper）
public class SqlSessionFactoryBuilder {
    public SqlSessionFactory build(List<Class<?>> mapperInterfaces) {
        Configuration config = new Configuration();
        MapperParser parser = new MapperParser();
        for (Class<?> mapper : mapperInterfaces) {
            parser.parse(mapper, config);   // 解析注解
        }
        return new SqlSessionFactory(config);
    }
}
```
### 面试怎么说
```
面试官："能说说 MyBatis 的原理吗？"

"MyBatis 的核心我理解有三块：

① 动态代理：Mapper 接口没有实现，
   运行时用 JDK 动态代理生成实现，
   调用方法时拦截（InvocationHandler）。

② SQL 映射：启动时解析注解/XML，
   把每个 SQL 封装成 MappedStatement，
   @Select 的 SQL 存起来（namespace + 方法名做 key）。

③ 执行引擎：
   代理拦截 → 找到 MappedStatement →
   解析 SQL（#{} → ?）→ PreparedStatement 参数注入 →
   执行 → ResultSet 反射映射成对象。

我手写过简易版，核心就是：
反射（创建对象/设置字段）
+ 动态代理（Mapper 实现）
+ JDBC（执行）
+ 注解解析（SQL 配置）
这四样凑起来。"

// 如果面试官让现场写，重点写：
// ① 动态代理（MapperProxy）
// ② 结果映射（反射 set 字段）
// 这两个是核心，其他可以简化
```
