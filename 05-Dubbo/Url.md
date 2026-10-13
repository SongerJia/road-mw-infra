## Url统一模型
url统一模型是什么->url结构->贯穿的场景->为什么这样设计->与SPI结合->面试高频题
### Url模型是什么
Dubbo用Url（统一资源定位符）来描述一切，例如一个服务、一个配置、一个注册信息、一条路由规则等。
```
// 一句话：
// Dubbo 世界里，所有东西最终都被抽象成一个 URL
// URL = 协议 + 地址 + 服务路径 + 参数

// 例：一个服务的 URL
dubbo://192.168.1.10:20880/com.example.UserService?
    interface=com.example.UserService&
    version=1.0.0&
    group=g1&
    timeout=3000&
    methods=getUser
```
### Url结构
```
protocol://username:password@host:port/path?key1=value1&key2=value2

各段含义：
protocol  ：协议（dubbo/rmi/http/...）
username  : password : 用户名密码（可选）
host:port ：主机端口
path      ：服务接口全限定名（如 com.example.UserService）
parameters：一堆参数（版本、分组、超时、序列化方式...）
```
Dubbo的URL类
```
public final class URL {
    private final String protocol;      // 协议
    private final String username;
    private final String password;
    private final String host;
    private final int port;
    private final String path;          // 服务路径
    private final Map<String, String> parameters;  // 参数
}
```
### Url贯穿的场景
①服务暴露时
```
// Provider 暴露服务 → 生成 URL → 注册到 Registry
// 注册节点的 value 就是一个 URL：
dubbo://192.168.1.10:20880/com.example.UserService?version=1.0.0&...
```
②服务引用时
```
// Consumer 订阅到 URL 列表 → 解析 URL → 生成 Invoker
// 每个 URL 对应一个 Provider
```
③配置时
```
// XML / 注解配置 → 转成 URL
<dubbo:service interface="..." timeout="3000"/>
    ↓
dubbo://...?interface=...&timeout=3000
```
④路由时
```
// 路由规则也是 URL：
condition://0.0.0.0/com.example.UserService?category=routers&rule=...
```
⑤监控/动态配置
```
// 监控数据、动态配置（override），都以 URL 形式存储
```
```
配置（XML/注解）
    ↓ 转成
URL  ←──────────→ 注册中心（Zookeeper 节点 = URL）
    ↓ 解析
服务暴露 / 服务引用
    ↓
路由规则 / 动态配置 / 监控
    ↓
一切都是 URL
```
### 为什么这样设计
```
// ① 统一：用同一种数据结构描述所有概念
//    → 代码简洁，易于理解

// ② 可扩展：新参数直接加在 URL 上
//    → 不需要改结构

// ③ 便于传输与存储：URL 是字符串
//    → 可以存 Zookeeper、可以通过网络传递

// ④ 灵活：字符串拼接，动态改参数方便

// ⑤ 与 SPI 结合：自适应扩展根据 URL 参数选实现（关键！）
```
举例：一个字符串搞定所有
```
// 想加一个配置？直接加参数
dubbo://...?interface=...&timeout=3000&retries=2&loadbalance=random

// 想改协议？换 protocol
rmi://...

// 想本地调用？换协议
injvm://...
```
### URL与SPI的结合
Dubbo的自适应扩展（Adaptive）会根据Url参数，动态选择用什么实现。
```
// 例：负载均衡的实现选择
// URL 里 loadbalance=random
// → SPI 根据这个参数选择 RandomLoadBalance

dubbo://...?loadbalance=random
// → 使用 RandomLoadBalance

dubbo://...?loadbalance=leastactive
// → 使用 LeastActiveLoadBalance
```
```
// 例：序列化方式
// URL 里 serialization=hessian2
// → 用 Hessian2 序列化

// 这就是"配置驱动"：
// 改配置（URL 参数）→ 改行为
// 不用改代码
```
自适应扩展原理
```
// @Adaptive 注解 + URL 参数 → 动态生成实现
// 运行时根据 URL 里的参数，决定用哪个 SPI 实现
// → 这是 Dubbo "微内核 + 插件式"设计的体现
```

