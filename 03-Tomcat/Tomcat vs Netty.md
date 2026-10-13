## Tomcat vs Netty
定位对比->协议对比->线程模式对比->设计理念->使用场景->面试高频题
### 定位对比
```
// Tomcat：Servlet 容器 + Web 服务器
//   → 面向"HTTP Web 应用"
//   → 遵循 Servlet 规范
//   → "开箱即用"的 Web 服务器

// Netty：通用网络通信框架
//   → 面向"任意协议的通信"
//   → 不绑定任何规范
//   → "造轮子的工具"

// 一句话：
// Tomcat = 精装好的 Web 应用服务器
// Netty  = 万能网络编程积木
```

|维度|Tomcat|Netty|
|---|---|---|
|**定位**|Web 服务器 + Servlet 容器|通用网络框架|
|**层次**|应用服务器|底层通信框架|
|**规范**|遵循 Servlet 规范|无规范约束|
|**开箱即用**|✅（部署 WAR 即用）|❌（要自己搭）|
### 协议对比
```
// Tomcat：主要支持 Web 相关协议
// ① HTTP/1.1（核心）
// ② HTTP/2
// ③ AJP
// ④ WebSocket
// ⑤ JSP

// Netty：支持任意协议
// ① HTTP/HTTPS
// ② WebSocket
// ③ TCP/UDP 自定义协议
// ④ 各种二进制协议（Protobuf/Thrift）
// ⑤ 只要你能写编解码器，任何协议都行
```
```
// Tomcat 的协议是"内置写好的"
// Netty 的协议是"自己组装的"（提供组件）

// 举例：要在 Netty 里支持 HTTP
pipeline.addLast(new HttpServerCodec());          // 自己加
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new MyHttpHandler());

// 举例：Tomcat 里支持 HTTP
// 配 <Connector protocol="HTTP/1.1"/> 即可（内置）
```
### 线程模型对比
```
// Tomcat（NIO）：Acceptor + Poller + Worker
// ① Acceptor：接收连接
// ② Poller：多路复用监听
// ③ Worker：业务处理（线程池）

// Netty：主从 Reactor
// ① BossGroup：接收连接（对应 Acceptor）
// ② WorkerGroup：IO 处理 + 业务（EventLoop）
// ③ 业务可再丢业务线程池
```

|维度|Tomcat|Netty|
|---|---|---|
|**接收层**|Acceptor|BossGroup|
|**IO 层**|Poller|WorkerGroup|
|**业务层**|Worker 线程池|EventLoop / 业务线程池|
|**模型**|多路复用 + 线程池|主从 Reactor|
|**共同点**|都是"多路复用 + 线程池"思想|同左|
```
// 核心相同：
// ① 都用 IO 多路复用（Selector）
// ② 都是少量线程管大量连接
// ③ 业务处理都交给线程池
// → Tomcat 的模型其实参考了 Netty 的思想（或说同源）
```
### 设计理念对比
| 维度       | Tomcat           | Netty           |
| -------- | ---------------- | --------------- |
| **核心抽象** | Servlet / Filter | ChannelHandler  |
| **责任链**  | Pipeline + Valve | ChannelPipeline |
| **编解码**  | 内置（HTTP/AJP）     | 提供组件（自己组装）      |
| **内存管理** | JVM GC           | 池化 + 引用计数       |
| **零拷贝**  | sendfile（部分）     | 全面优化            |
| **事件模型** | 轮询               | 异步事件驱动          |
| **生命周期** | Lifecycle 接口     | EventLoop 管理    |
| **配置**   | XML（server.xml）  | 代码（Bootstrap）   |
### 使用场景
Tomcat
```
// ① 传统 Web 应用（SSM/Spring Boot Web）
// ② 需要 Servlet/JSP
// ③ 快速部署（WAR 包）
// ④ 不需要极致性能
```
Netty
```
// ① RPC 框架（Dubbo/gRPC）
// ② IM 即时通讯
// ③ 游戏服务器
// ④ API 网关
// ⑤ 需要自定义协议
// ⑥ 极致性能要求
```
特殊：SpringBoot
```
// Spring Boot 默认内嵌 Tomcat
// 但可以换成 Netty（WebFlux + Reactor Netty）
//   spring-boot-starter-webflux

// 区别：
// Spring MVC（Tomcat）= 同步阻塞（一请求一线程）
// Spring WebFlux（Netty）= 异步非阻塞（事件循环）
```
### 面试高频题
#### 题目1：为什么中间件用Netty不用Tomcat
```
// Netty 更灵活（自定义协议）+ 性能更高 + 更轻量
// Tomcat 太重（Servlet 规范开销）
```