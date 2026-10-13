## 调用流程
总览->服务暴露->服务引用->代理调用->完整调用链->面试高频题
### 总览（两大阶段）
```
// Dubbo 的核心流程分两个阶段：
// ① 服务暴露（Provider 端）：把服务"发布"出去
// ② 服务引用 + 调用（Consumer 端）："拿到"服务并调用

// 消费者调用时，完整链路：
// 代理 → 集群 → 路由 → 负载均衡 → 过滤链 → 网络 → 服务端 → 真实实现
```
### 服务暴露（Provider端）
```
// ① 解析配置（ServiceConfig）
// ② 生成 Invoker（对真实服务实现的包装）
// ③ 暴露服务：本地暴露 + 远程暴露
// ④ 注册到 Registry

// 代码（配置方式）：
@Service  // Dubbo 的
public class UserServiceImpl implements UserService { ... }
```
两个URL
```
// ① 本地暴露 URL：injvm://...
//    → 同 JVM 内调用（不经过网络）
//    → 如果 Consumer 和 Provider 在同一个 JVM，走本地调用

// ② 远程暴露 URL：dubbo://192.168.1.10:20880/com.xxx.UserService
//    → 经过网络
```
暴露过程
```
ServiceConfig.export()
    ↓
① 本地暴露：exportLocal() → injvm Invoker 保存到本地 Map
    ↓
② 远程暴露：
   - 生成 DubboExporter
   - 启动 Netty Server（监听端口）
   - 注册到 Registry（dubbo://...）
```
```
// 关键：Provider 启动时
// ① 打开端口（默认 20880，Netty）
// ② 把服务地址注册到 Zookeeper/Nacos
```
### 服务引用（Consumer端）
```
// ① 解析配置（ReferenceConfig）
// ② 从 Registry 订阅服务
// ③ 生成 Invoker（远程调用包装）
// ④ 通过集群（Cluster）包装
// ⑤ 生成动态代理（返回给业务）
```
```
// 代码：
@Reference
private UserService userService;   // 拿到的是代理对象
```
引用过程
```
ReferenceConfig.get()
    ↓
① 检查本地引用缓存
    ↓
② 远程引用：
   - 注册到 Registry（作为消费者）
   - 订阅 Provider 地址列表
   - 生成 Invoker（DubboInvoker）
    ↓
③ 集群包装：Cluster.join() → 生成 ClusterInvoker
   - 内部含 Directory（服务目录）+ Router + LoadBalance
    ↓
④ 生成代理：ProxyFactory.getProxy()
   - 返回 UserService 的代理对象
   - 调用代理方法 → 实际走远程调用
```
### 代理调用（动态代理）
```
// Consumer 注入的 UserService 不是真实对象，是代理！

// 调用 userService.getUser(1L)
// → 代理拦截 → 转换成远程调用

// 动态代理实现：
// ① Javassist（默认，性能好）
// ② JDK 动态代理
// ③ ByteBuddy（新版本）
```
代理做了什么
```
// ProxyFactory 生成的代理，拦截方法调用：
proxy.getUser(1L)
    ↓
// ① 把方法名 + 参数 + 接口名 封装成 Invocation
Invocation invocation = new RpcInvocation(methodName, paramTypes, args);
    ↓
// ② 交给 Invoker 链处理
invoker.invoke(invocation)
```
代理生成 Provider端getInvoker
```
// org.apache.dubbo.rpc.proxy.javassist.JavassistProxyFactory

public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    final Wrapper wrapper = Wrapper.getWrapper(
        proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            // 动态生成的 Wrapper 直接反射调用真实方法（避免反射开销）
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```
代理生成 Consumer端getProxy
```
// JavassistProxyFactory
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    // ① Proxy.getProxy 生成字节码
    // ② InvokerInvocationHandler 拦截方法调用
    return (T) Proxy.getProxy(interfaces)
                  .newInstance(new InvokerInvocationHandler(invoker));
}
```
```
// org.apache.dubbo.rpc.proxy.InvokerInvocationHandler —— 调用的总入口！
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ① 处理 Object 方法（toString/hashCode 等）
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker.getInterface(), args);
    }
    // ② 封装成 RpcInvocation
    RpcInvocation invocation = new RpcInvocation(method, invoker.getInterface().getName(), args);
    // ③ 进入 Invoker 链
    return invoker.invoke(invocation).recreate();
}
```
### 完整调用链
Consumer端
```
① 业务调用 userService.getUser(1L)
    ↓（代理拦截）
② ProxyInvoker → Invocation（封装方法+参数）
    ↓
③ ClusterInvoker（集群容错层：失败重试等）
    ↓
④ Directory（服务目录：拿到 Provider 列表）
    ↓
⑤ Router（路由过滤：按规则筛选）
    ↓
⑥ LoadBalance（负载均衡：选一台 Provider）
    ↓
⑦ Filter 链（Consumer 端过滤器）
    ↓
⑧ Client 网络层（基于 Netty 发送请求）
    ↓ 网络
```
Provider端
```
⑨ Netty Server 收到请求
    ↓
⑩ 解码（反序列化）
    ↓
⑪ Filter 链（Provider 端过滤器）
    ↓
⑫ Invoker → 真正调用 UserServiceImpl.getUser(1L)
    ↓
⑬ 结果编码（序列化）
    ↓ 网络
⑭ 返回给 Consumer
```
```
Consumer                  Provider
────────                  ────────
Proxy（代理）
  ↓
Invoker
  ↓
Cluster（集群）
  ↓
Directory + Router
  ↓
LoadBalance
  ↓
Filter 链
  ↓
Client（Netty）───网络───→ Server（Netty）
                              ↓
                           解码反序列化
                              ↓
                           Filter 链
                              ↓
                           Invoker
                              ↓
                           真实实现 UserServiceImpl
                              ↓
                           编码序列化 ───→ 返回
```
### 关键组件
| 组件                        | 作用                  |
| ------------------------- | ------------------- |
| **ProxyFactory**          | 生成代理（Javassist/JDK） |
| **Invoker**               | 执行调用的统一抽象（核心）       |
| **Directory**             | 服务目录，维护 Provider 列表 |
| **Router**                | 路由规则（筛选 Provider）   |
| **Cluster**               | 集群容错（失败策略）          |
| **LoadBalance**           | 负载均衡（选一台）           |
| **Filter**                | 过滤器链                |
| **Protocol**              | 协议 + 暴露/引用          |
| **ExchangeClient/Server** | 信息交换层（请求响应）         |
| **Transport**             | 网络传输层（Netty）        |
```
// 核心抽象是"Invoker"！
// 无论本地/远程、集群/单机，都用 Invoker 统一表示
// "一切皆 Invoker"
```
Dubbo的分层
```
Service（业务层）
Config（配置层）
Proxy（代理层）       ← 生成代理
Registry（注册层）    ← 服务发现
Cluster（集群层）     ← 容错/路由/负载均衡
Monitor（监控层）
Protocol（协议层）    ← RPC 调用
Exchange（信息交换层）← 请求响应封装
Transport（网络传输层）← Netty
Serialize（序列化层）
```
集群层
```
// org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker
public Result invoke(final Invocation invocation) throws RpcException {
    // ① Directory 拿到 Provider 列表
    List<Invoker<T>> invokers = list(invocation);
    // ② 初始化负载均衡
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    // ③ 交给子类（如 FailoverClusterInvoker）
    return doInvoke(invocation, invokers, loadbalance);
}
```
网络调用
```
// org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker
protected Result doInvoke(final Invocation invocation) throws Throwable {
    ...
    ExchangeClient currentClient = clients[index];
    ...
    // 请求发送 + 异步 → 同步等待结果
    return (Result) currentClient.request(inv, timeout)
                                 .get(timeout, TimeUnit.MILLISECONDS);
}
```

