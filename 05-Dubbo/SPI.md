## 扩展机制
为什么需要扩展机制->JDK SPI问题->Dubbo SPI->自适应扩展->激活扩展->Wrapper->ExtensionLoader->Filter
### 为什么需要扩展
```
// Dubbo 是"微内核 + 插件式"架构
// 内核很小，功能都靠扩展点（SPI）实现

// 例：
// 协议：Dubbo/RMI/HTTP（可换）
// 序列化：Hessian2/Kryo/Protobuf（可换）
// 负载均衡：Random/LeastActive（可换）
// 注册中心：Zookeeper/Nacos（可换）

// 全靠 SPI 实现"可插拔"
// → 想换实现，改配置即可（不用改代码）
```
### JDK SPI问题
```
// JDK SPI 用法：
// ① 接口 com.example.SpiService
// ② 文件 META-INF/services/com.example.SpiService
//    内容：com.example.ImplA
// ③ ServiceLoader.load(SpiService.class)

// JDK SPI 的 4 大问题：
// ① 一次性加载所有实现（不能按需）
//    → 浪费资源、慢
// ② 无法获取指定实现（只能遍历）
//    → ServiceLoader 是迭代器，要自己选
// ③ 没有 IOC / AOP 支持
// ④ 扩展点没有名字（不能按名指定）
```
```
// JDK SPI 只能这样：
ServiceLoader<Foo> loader = ServiceLoader.load(Foo.class);
for (Foo foo : loader) {       // 遍历所有实现
    if (foo instanceof FooA) { ... }  // 自己判断
}
// 无法 FooA foo = loader.get("fooA")
```
### Dubbo SPI 改进
```
// ① 接口加 @SPI 注解（指定默认实现）
@SPI("dubbo")                    // 默认用 name=dubbo 的实现
public interface Protocol {
    int getDefaultPort();
}
```
```
// ② 配置文件：META-INF/dubbo/接口全限定名
// 文件名：org.apache.dubbo.rpc.Protocol
// 内容（key=value）：
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
rmi=org.apache.dubbo.rpc.protocol.rmi.RmiProtocol
http=org.apache.dubbo.rpc.protocol.http.HttpProtocol
```
```
// ③ 使用
Protocol protocol = ExtensionLoader
    .getExtensionLoader(Protocol.class)
    .getExtension("dubbo");      // 按名字拿指定实现！
```

|问题|JDK SPI|Dubbo SPI|
|---|---|---|
|**按需加载**|❌ 全加载|✅ 按名加载|
|**按名获取**|❌|✅ getExtension("x")|
|**IOC**|❌|✅ 自动注入依赖|
|**AOP**|❌|✅ Wrapper 包装|
|**默认实现**|❌|✅ @SPI("x")|
|**自适应**|❌|✅ @Adaptive|
### 自适应扩展 @Adaptive
```
// 场景：一个扩展点有多种实现，运行时根据 URL 参数决定用哪个
// 但此时还不知道用哪个（要运行时才知道）

// 解决：自适应扩展 —— 生成一个"代理类"
// 运行时根据 URL 参数，动态选择真正实现
```
两种自适应
```
// ① 类级别自适应：@Adaptive 标在类上
//    → 该扩展点直接用这个类作为"自适应实现"

// ② 方法级别自适应：@Adaptive 标在方法上
//    → 生成动态代理类，根据 URL 参数选实现
```
源码示例
```
@SPI("javassist")
public interface ProxyFactory {
    @Adaptive({"proxy"})     // 从 URL 的 proxy 参数选实现
    <T> T getProxy(Invoker<T> invoker, Class<?>[] types);

    @Adaptive({"proxy"})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url);
}
```
```
// 生成的动态代理（伪代码）：
public class ProxyFactory$Adaptive implements ProxyFactory {
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] types) {
        if (invoker == null) throw new IllegalArgumentException("...");
        URL url = invoker.getUrl();
        String extName = url.getParameter("proxy", "javassist");  // 从 URL 拿
        ProxyFactory extension = ExtensionLoader
            .getExtensionLoader(ProxyFactory.class)
            .getExtension(extName);     // 按名字拿
        return extension.getProxy(invoker, types);
    }
}
```
### 激活扩展 @Activate
```
// 场景：Filter 链、Listener 等，需要"按条件批量激活"多个实现
// 解决：@Activate 注解

// 用法：标注在实现类上
@Activate(group = "provider", order = -1000)  // 服务端、优先级
public class EchoFilter implements Filter { ... }

@Activate(group = "consumer")
public class ConsumerFilter implements Filter { ... }
```
```
// 获取激活的扩展（按 group 过滤）
List<Filter> filters = ExtensionLoader
    .getExtensionLoader(Filter.class)
    .getActivateExtension(url, "filter", "provider");
// → 返回所有满足 group=provider 的 Filter
```
@Activate参数
```
@Activate(
    group = {"provider", "consumer"},  // 分组
    value = "cache",                   // key（URL 里有才激活）
    order = -100                       // 顺序（越小越前）
)
```
### Wrapper（AOP支持）
```
// 如果扩展实现类的构造器是"只有一个参数（接口类型）"
// Dubbo 认为它是 Wrapper（包装类）→ 实现 AOP

// 示例：
public class ProtocolFilterWrapper implements Protocol {
    private final Protocol protocol;

    // 构造器参数是接口类型 → 被识别为 Wrapper
    public ProtocolFilterWrapper(Protocol protocol) {
        this.protocol = protocol;
    }

    @Override
    public Exporter export(Invoker invoker) {
        // 前置增强
        return protocol.export(buildFilterChain(invoker));  // 加 Filter 链
    }
}
```
```
// getExtension 时，Dubbo 会：
// ① 找到真正的实现（如 DubboProtocol）
// ② 用所有 Wrapper 层层包装
// 最终拿到：Wrapper1(Wrapper2(DubboProtocol))
// → 天然的 AOP
```
### ExtensionLoader源码
```
// org.apache.dubbo.common.extension.ExtensionLoader —— 扩展机制的核心类

// ① 入口：getExtensionLoader（每个扩展点一个 Loader，缓存）
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    ExtensionLoader<T> loader = EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = EXTENSION_LOADERS.get(type);
    }
    return loader;
}

// ② 按名获取扩展
public T getExtension(String name) {
    Holder<Object> holder = cachedInstances.get(name);   // 缓存
    // 双重检查 + 创建
    Object instance = createExtension(name);
    // ...
    return (T) instance;
}

// ③ 创建扩展（核心）
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);   // 从 META-INF/dubbo 读到
    T instance = (T) clazz.newInstance();
    inspectInstance(instance);         // 校验
    injectExtension(instance);         // ④ IOC：自动注入依赖
    // ⑤ AOP：Wrapper 包装
    for (Class<?> wrapperClass : cachedWrapperClasses) {
        instance = (T) wrapperClass.getConstructor(type).newInstance(instance);
    }
    return instance;
}
```
三个关键能力
```
// ① 按需加载：从 META-INF/dubbo 读取，按 name 加载
// ② IOC（injectExtension）：自动注入 setter 定义的依赖
// ③ AOP（Wrapper）：构造器注入 → 层层包装
```
injectExtension（IOC源码）
```
private T injectExtension(T instance) {
    for (Method method : instance.getClass().getMethods()) {
        if (method.getName().startsWith("set") && method.getParameterTypes().length == 1) {
            // 找到 setter → 从 URL/ExtensionLoader 拿依赖 → 注入
            Object object = getExtensionByType(method.getParameterTypes()[0]);
            method.invoke(instance, object);
        }
    }
    return instance;
}
```
### Filter机制（基于SPI）
```
// Filter 是一种特殊的扩展点，用 @Activate 批量激活

@SPI
public interface Filter {
    Result invoke(Invoker<?> invoker, Invocation invocation);
}
```
```
// 配置（配合 @Activate）
@Activate(group = {CONSUMER, PROVIDER})
public class TimeoutFilter implements Filter {
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        // 前后增强
        return invoker.invoke(invocation);
    }
}
```
```
// Filter 链的组装（ProtocolFilterWrapper 里）
List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class)
    .getActivateExtension(url, key, group);
// 然后组成链：filter1 → filter2 → ... → 真正的 Invoker
```
职责链源码
```
// 最后一个节点是真正的调用
private static class ChannelFilterWrapper {
    // ...
    return filters.get(i).invoke(nextInvoker, invocation);   // 递归成链
}
```
