## 注册中心
注册中心的职责->Zookeeper数据结构->注册/订阅/通知->ZK vs Nacos->缓存容错->面试高频题
### 注册中心的职责
```
// 四大职责：
// ① 服务注册：Provider 上线 → 记录地址
// ② 服务发现：Consumer 订阅 → 拿到地址列表
// ③ 通知：Provider 变化 → 推送给 Consumer
// ④ 健康检查：剔除宕机的 Provider

// Dubbo 支持的注册中心：
// Zookeeper（默认，2.x）、Nacos、Redis、Multicast、Simple
```
### Zookeeper的数据结构
```
Zookeeper 节点树：

/dubbo
  └── com.example.UserService              （服务接口）
       ├── providers                        （提供者目录）
       │    └── dubbo://192.168.1.10:20880?...   （URL，临时节点）
       ├── consumers                        （消费者目录）
       │    └── consumer://192.168.1.20/...?...
       ├── routers                          （路由规则）
       │    └── condition://...?rule=...
       └── configurators                    （动态配置）
            └── override://...?timeout=5000
```
节点类型
```
// providers 节点 = 临时节点（EPHEMERAL）★
// 特点：会话结束（连接断开）→ 节点自动删除
// → Provider 宕机，ZK 会自动剔除

// routers/configurators = 持久节点
// → 重启不丢失
```
```
// 这是 ZK 做"健康检查"的机制：
// Provider 与 ZK 保持心跳（会话）
// Provider 挂了 → 会话过期 → 临时节点删除 → 通知 Consumer
```
### 注册/订阅/通知机制
①注册 Provider
```
// 创建临时节点，value = URL
zkClient.create("/dubbo/UserService/providers/dubbo://192.168.1.10:20880?...",
                EPHEMERAL);
```
②订阅 Consumer
```
// ① 订阅 providers 节点（Watcher）
// ② 拉取当前子节点列表
// ③ 注册 Watcher：子节点变化时通知
```
③通知 Watcher
```
// Provider 上线/下线 → providers 子节点变化
// → ZK 触发 Watcher → Consumer 收到通知
// → Consumer 重新拉取最新列表 → 更新本地缓存
```
完整流程
```
Provider 启动
    ↓ 注册临时节点
ZK /providers/dubbo://...
    ↓ Consumer 订阅（Watcher）
Consumer 拿到列表，缓存本地
    ↓
Provider 宕机
    ↓ 会话过期，临时节点删除
ZK 通知 Consumer（Watcher 触发）
    ↓
Consumer 重新拉取列表，更新本地缓存
    ↓
调用时避开已下线的 Provider
```
### 源码 ZookeeperRegistry
```
// org.apache.dubbo.registry.zookeeper.ZookeeperRegistry

// ① 注册
public void doRegister(URL url) {
    zkClient.create(toUrlPath(url),
                    url.getParameter(DYNAMIC_KEY, true));  // dynamic=true → 临时节点
}

// ② 订阅
public void doSubscribe(final URL url, final NotifyListener listener) {
    // ① 创建 providers 持久节点目录
    // ② 注册 ChildListener（监听子节点变化）
    zkClient.addChildListener(path, childListener);
    // ③ 拉取当前子节点（Provider 列表）
    List<String> children = zkClient.addChildListener(...);
    // ④ 转成 URL 列表，通知 Listener（即 RegistryDirectory）
    List<URL> urls = toUrlsWithoutEmpty(url, children);
    notify(url, listener, urls);
}

// ③ 通知（子节点变化时）
private ChildListener childListener = (parentPath, currentChildren) -> {
    // ZK 回调 → 转成 URL → notify(listener, urls)
    // → 更新 Directory 的本地缓存
};
```
```
// RegistryDirectory：Consumer 端的"服务目录"
// ① 持有最新的 Provider URL 列表
// ② 收到 notify → 更新列表 → 触发 Invoker 重建
// ③ ClusterInvoker 从 Directory 拿最新列表（负载均衡用）
```
### Zookeeper vs Nacos
| 维度        | Zookeeper          | Nacos              |
| --------- | ------------------ | ------------------ |
| **一致性模型** | CP（强一致）            | AP（可用优先）+ 可切 CP    |
| **健康检查**  | 心跳 + 会话过期          | 心跳 + TCP 探测 + HTTP |
| **通知**    | Watcher（一次性，需重新注册） | 推送 + 轮询            |
| **存储**    | ZNode 树            | 内存 + 持久化           |
| **性能**    | 一般                 | 较高                 |
| **易用性**   | 一般                 | 好（控制台）             |
CAP分析
```
// Zookeeper = CP：
// ① 主从架构，写要过半数节点同意
// ② 主挂了要重新选举 → 短暂不可用
// ③ 强一致，但可用性差点

// Nacos = AP（默认）：
// ① 节点间异步复制（最终一致）
// ② 任何节点都能读写
// ③ 可用性优先

// 为什么注册中心推荐 AP？
// ① 服务列表短暂不一致可容忍（调用方有本地缓存）
// ② 注册中心不可用 → 全部无法调用（灾难）
// → 可用性比强一致更重要
```
Watcher的一次性问题
```
// ZK 的 Watcher 触发一次后就失效
// → Dubbo 每次 Watcher 触发后重新注册 Watcher
// → 保证持续监听
```
### 缓存与容错
Consumer缓存
```
// Consumer 会把 Provider 列表缓存到：
// ① 内存（RegistryDirectory 的 Map）
// ② 磁盘（~/.dubbo/dubbo-registry-xxx.cache）

// 好处：
// ① 注册中心宕机 → 用内存缓存继续调用
// ② Consumer 重启 → 注册中心也挂 → 用磁盘缓存
// → 高可用
```
注册中心宕机的影响
```
// 注册中心全挂：
// ① 已有服务调用：不受影响（有本地缓存）
// ② 新服务发布/订阅：失败
// ③ 服务变更通知：收不到
// → 尽快恢复注册中心
```
