## 核心角色
Dubbo是什么->五大角色->调用时序->关键设计->面试高频题
### Dubbo是什么
Dubbo是一款高性能的Java RPC框架+服务治理平台。核心：让远程调用像本地调用一样简单，并提供一整套的服务治理能力。
```
// 两个关键词：
// ① RPC 框架：像调用本地方法一样调用远程服务
// ② 服务治理：注册发现、负载均衡、容错、限流、监控

// 本质：
// Consumer 调用接口 → 底层通过网络（Netty）远程执行 → 返回结果
// 对开发者透明（用代理隐藏了网络细节）
```
```
// ② Provider 实现 + 暴露
@Service                                    // Dubbo 的 @Service
public class UserServiceImpl implements UserService {
    public User getUser(Long id) { return new User(id, "张三"); }
}

// ③ Consumer 引用 + 调用（像本地调用）
@Reference                                   // Dubbo 的 @Reference
private UserService userService;

public void test() {
    User user = userService.getUser(1L);     // 远程调用，透明
}
```
### 五大角色
| 角色            | 说明                              |
| ------------- | ------------------------------- |
| **Provider**  | 服务提供者（暴露服务的应用）                  |
| **Consumer**  | 服务消费者（调用远程服务的应用）                |
| **Registry**  | 注册中心（服务注册与发现，如 Zookeeper、Nacos） |
| **Monitor**   | 监控中心（统计调用次数、耗时）                 |
| **Container** | 服务运行容器（负责启动、加载、运行 Provider）     |
各角色职责
```
// ① Provider：提供并暴露服务
//    - 启动时注册自己到 Registry
//    - 处理 Consumer 的调用请求

// ② Consumer：调用远程服务
//    - 启动时订阅所需服务
//    - 从 Registry 拿到 Provider 列表
//    - 直接调用 Provider

// ③ Registry：服务注册与发现
//    - 存储服务地址列表
//    - 服务变更时推送通知

// ④ Monitor：监控统计
//    - 收集调用次数、耗时
//    - 用于运维分析

// ⑤ Container：服务容器
//    - 负责启动、加载 Provider
//    - 通常就是 Spring 容器
```
### 调用时序
```
Container                    Registry
    │                           │
    │ 0. 启动加载 Provider       │
    ↓                           │
Provider ──1. 注册服务──────────→│
    │                           │
Consumer ──2. 订阅服务──────────→│
    │                           │
    │ ←─────3. 通知地址列表───────│
    │                           │
    │ ──────4. 调用────────────→ Provider
    │                           │
    │ ←─────4. 返回结果────────── Provider
    │
    │ ──5. 上报统计──→ Monitor ←──5. 上报统计── Provider
```
详细步骤
```
// 0. Container 启动，加载并运行 Provider（通常是 Spring 容器）
// 1. Provider 启动时，向 Registry 注册自己提供的服务
// 2. Consumer 启动时，向 Registry 订阅自己所需的服务
// 3. Registry 返回 Provider 地址列表给 Consumer
//    （如果地址有变更，Registry 会主动推送最新列表给 Consumer）
// 4. Consumer 从地址列表中，按负载均衡算法选一台 Provider 调用
// 5. Consumer 和 Provider 定时上报调用统计给 Monitor（内存统计）
```
关键点
```
// ⭐ 核心：Consumer 和 Provider 之间是"直连"，不经注册中心！
// 注册中心只负责"服务发现"（告诉 Consumer 有哪些 Provider）
// 真正的调用是 Consumer → Provider 直连（点对点）
// → 注册中心不是流量瓶颈

// ⭐ 注册中心宕机也不影响调用：
// Consumer 本地缓存了 Provider 列表
// → 注册中心挂了，调用照常（只是新服务无法发现）

// ⭐ 长连接：
// Consumer 和 Provider 之间保持长连接（单一长连接）
// → 减少连接开销
```
### 为什么这么设计
问题：为什么调用不经注册中心
```
// 如果所有请求都经注册中心转发：
// ① 注册中心成瓶颈
// ② 多一跳延迟
// ③ 注册中心挂了全盘崩（单点）

// Dubbo 的设计：
// 注册中心只做"服务发现"
// 调用走直连 → 高性能 + 高可用
```
问题：注册中心挂了怎么办
```
// Consumer 本地缓存 Provider 列表
// ① 注册中心宕机 → Consumer 用本地缓存继续调用
// ② 注册中心恢复 → Consumer 重新订阅
// → 高可用
```
问题：地址变更怎么感知
```
// Registry 用"推送"机制（事件通知）
// Provider 上线/下线 → Registry 通知所有 Consumer
// Consumer 更新本地列表
```
### Dubbo的两种角色视角
```
// 同一个应用，可能既是 Provider 又是 Consumer！

// 例：订单服务
// ① 对外提供 OrderService（它是 Provider）
// ② 内部调用 UserService、GoodsService（它也是 Consumer）

// 所以 Dubbo 里"提供者/消费者"是相对某个服务而言的
```
服务注册的源码链
```
// 入口：ServiceConfig.export() → 最终走到 RegistryProtocol
// org.apache.dubbo.registry.integration.RegistryProtocol

public <T> Exporter<T> export(final Invoker<T> originInvoker) {
    // ① 注册到注册中心
    register(registryUrl, registeredProviderUrl);
    // ② 本地暴露
    ...
}

public void register(URL registryUrl, URL registeredProviderUrl) {
    Registry registry = registryFactory.getRegistry(registryUrl);
    registry.register(registeredProviderUrl);   // 委托给具体 Registry
}
```
```
// 具体实现：ZookeeperRegistry（基于 Curator）
// org.apache.dubbo.registry.zookeeper.ZookeeperRegistry

public void doRegister(URL url) {
    zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
    // 在 Zookeeper 上创建节点，节点 value 就是 URL
}
```
服务发现的源码链
```
// Consumer 订阅
// ZookeeperRegistry.doSubscribe()
public void doSubscribe(final URL url, final NotifyListener listener) {
    // ① 订阅
    zkClient.create(path, false);
    // ② 拉取当前子节点（Provider 列表）
    List<String> children = zkClient.addChildListener(path, ...);
    // ③ 把子节点转成 URL 列表，回调 listener
    List<URL> urls = toUrlsWithoutEmpty(url, children);
    notify(url, listener, urls);
}
```
```
// 关键：NotifyListener 就是"地址变更推送"的落地点
// Balancer/Directory 订阅后，收到通知 → 更新本地 Provider 列表
```
