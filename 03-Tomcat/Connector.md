## 连接器
Connector是什么->三大组件->协议支持->IO模型->处理流程->面试高频题
### Connector是什么
Connector：Tomcat对外的大门。负责接收TCP连接，解析应用层协议，把请求封装成Servlet能处理的Request/Response。
```
// 位置：Service 里
// 职责：
// ① 监听端口，接收 TCP 连接
// ② 按协议解析字节流（HTTP/AJP）
// ③ 封装成 HttpServletRequest / HttpServletResponse
// ④ 交给 Container（Engine）处理
// ⑤ 把结果编码成 HTTP 响应发回

// 一句话：连接器 = "通信专家"，容器 = "业务专家"
```
一个service可以有多个Connector
```
<Service name="Catalina">
    <!-- HTTP 连接器（给浏览器用） -->
    <Connector port="8080" protocol="HTTP/1.1"/>
    <!-- AJP 连接器（给前端代理 Nginx/Apache 用） -->
    <Connector port="8009" protocol="AJP/1.3"/>
</Service>
// 作用：一个 Engine 可以有多种接入方式
// 但底层处理逻辑（Engine）共享
```
## 三大组件
Connector内部由三个组件构成：ProtocolHandler、EndPoint、Processor。
```
Connector
    └─ ProtocolHandler（协议处理器：组合并协调下面两个）
           ├─ Endpoint（端点：网络通信/IO）
           └─ Processor（处理器：协议解析）
```
ProtocolHandler 协议处理器
```
// 顶层抽象：负责"如何处理连接"
// 具体实现：
//   Http11Protocol    → HTTP/1.1（BIO，已淘汰）
//   Http11NioProtocol → HTTP/1.1 + NIO（默认）
//   Http11AprProtocol → HTTP/1.1 + APR
//   AjpNioProtocol    → AJP + NIO

public interface ProtocolHandler {
    void init() throws Exception;
    void start() throws Exception;
    void pause();
    void resume();
    void stop();
    void destroy();
    void setAdapter(Adapter adapter);   // 设置到 Container 的桥接
}
```
EndPoint 端点
```
// 职责：底层网络通信（Socket 的接收/发送）
// 抽象类：AbstractEndpoint
// 核心子类：
//   NioEndpoint  → NIO（同步非阻塞，默认）
//   Nio2Endpoint → NIO2（异步非阻塞）
//   AprEndpoint  → APR（本地库，高性能）

// Endpoint 内部管理：
// ① Acceptor  → 接收连接
// ② Poller    → 事件轮询（IO 多路复用）
// ③ SocketProcessor → 处理 socket 读写
```
Processor 处理器
```
// 职责：把 Endpoint 收到的"字节流"解析成"请求对象"
// 实现：
//   Http11Processor → 解析 HTTP/1.1
//   AjpProcessor    → 解析 AJP
//   Http2Processor  → 解析 HTTP/2

// 流程：
// Endpoint 收到数据 → Processor 解析 HTTP → 生成 Request/Response
```
三者协作
```
请求到达
    ↓
Endpoint（Acceptor 接收连接 → Poller 监听可读 → 读出字节）
    ↓ 字节
Processor（解析 HTTP 协议 → 生成 Request/Response）
    ↓ Request
ProtocolHandler（协调：把 Request 交给 Adapter/CoyoteAdapter）
    ↓
Engine（Container 处理）
```
### 协议支持
| 协议            | 说明                    | 场景                 |
| ------------- | --------------------- | ------------------ |
| **HTTP/1.1**  | 主协议                   | 浏览器直连              |
| **HTTP/2**    | 多路复用                  | 现代浏览器（Tomcat 8.5+） |
| **AJP**       | Apache JServ Protocol | Nginx/Apache 反代转发  |
| **WebSocket** | 全双工                   | 实时通信               |
```
// AJP 是什么？
// 二进制协议，比 HTTP 更紧凑（省掉 HTTP 头开销）
// Nginx/Apache 用 mod_jk/mod_proxy_ajp 转发到 Tomcat:8009
// 现在推荐直接 HTTP 反代（AJP 有安全隐患，CVE-2020-1938 幽灵猫）
```
### IO模型
Tomcat支持4种IO模型，通过< Connector protocol="..."> 或protocolClass指定。
①BIO
```
// JDK 1.4 之前
// 一个连接一个线程
// protocol="HTTP/1.1"（Tomcat 8 之前默认）
// 缺点：并发高时线程爆炸
```
②NIO 同步非阻塞 默认
```
// JDK 1.4 起
// 多路复用（Selector）
// 一个线程管多个连接
// protocol="org.apache.coyote.http11.Http11NioProtocol"
// Tomcat 8+ 默认
```
③NIO2 异步非阻塞
```
// JDK 7 起（AIO）
// 真正的异步 IO（Proactor 模型）
// protocol="org.apache.coyote.http11.Http11Nio2Protocol"
// 适合长连接
```
④APR Apache Portable Runtime
```
// 调用本地 C 库（openssl/apr）
// 性能最高（接近 Nginx）
// protocol="org.apache.coyote.http11.Http11AprProtocol"
// 需要安装本地库
```

|模型|原理|线程模型|性能|
|---|---|---|---|
|**BIO**|阻塞|一连接一线程|低|
|**NIO**|多路复用|Selector 轮询|高（默认）|
|**NIO2**|异步|回调|高|
|**APR**|本地库|事件驱动|最高|
```
// 配置示例
<Connector port="8080"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="200"
           acceptCount="100"/>
```
### 处理流程
```
① 客户端 TCP 连接到达
    ↓
② Acceptor 线程 accept → 注册到 Poller
    ↓
③ Poller 检测到可读事件 → 交给 SocketProcessor
    ↓
④ SocketProcessor → Processor 解析 HTTP
    ↓
⑤ 生成 Request/Response → CoyoteAdapter
    ↓
⑥ 交给 Engine（Container 处理）
    ↓
⑦ 处理完 → Processor 编码响应 → 写回客户端
```
关键类
```
// Endpoint 内的三个线程角色：
// Acceptor：接收新连接（数量少）
// Poller：监听 IO 事件（数量少）
// Worker：执行业务处理（数量多，用线程池）

// 这是典型的"多路复用 + 线程池"模型
```
### 面试高频题
#### 题目1：Connector三大组件
```
// ProtocolHandler + Endpoint + Processor
```

