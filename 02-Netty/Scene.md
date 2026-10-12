## 应用场景
Netty定位->四大场景->各场景技术要点->中间件底层->实战类比->面试高频题
### Netty定位
Netty是高性能异步事件驱动的网络通信框架。几乎所有Java高性能中间件/框架的网络层都以它为基础。
```
// Netty 不是业务框架，是"造轮子的轮子"
// 上层：业务框架（Dubbo/gRPC/Spring WebFlux）
// 中层：Netty（网络通信）
// 下层：Java NIO / epoll
```
### 四大典型场景
场景一：RPC框架
```
// 服务间调用，需要高性能通信
// Netty 负责：编解码 + 传输 + 异步

// 典型：Dubbo
// ① Consumer 用 Netty 发请求
// ② Provider 用 Netty 收请求
// ③ 自定义协议（Dubbo 协议头 + 数据）
```
Dubbo协议结构
```
+--------+--------+--------+--------+--------+
| 魔数    | 标志位  | 状态位 | 请求ID |  数据   |
| 0xdabb | 1字节   | 1字节  | 8字节  |  N字节  |
+--------+--------+--------+--------+--------+
```
Netty在RPC中的作用
```
// ① 客户端：连接复用、异步请求、超时
// ② 服务端：解码请求 → 业务处理 → 编码响应
// ③ 心跳保活（长连接）
```
场景二：IM即时通信
```
// 海量长连接 + 消息推送
// 特点：连接数多（百万级）、消息实时

// Netty 负责：
// ① 维持海量长连接（EventLoop 多路复用）
// ② 消息推送（双向通信）
// ③ 心跳检测（连接存活）
// ④ 断线重连
```
技术要点
```
// ① 用 WebSocket 协议（Web 端）
pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));

// ② 用 Protobuf 编解码（省流量）
pipeline.addLast(new ProtobufDecoder(Message.getDefaultInstance()));

// ③ 用 ChannelGroup 群发
ChannelGroup group = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
group.writeAndFlush(msg);   // 给所有连接群发
```
场景三：API网关
```
// 流量入口，做路由、限流、鉴权
// Netty 负责：高性能 HTTPServer

// 典型：Spring Cloud Gateway（底层 WebFlux + Netty）
//      Zuul 2.x、Soul 网关
```
技术要点
```
// ① HTTP 编解码
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));

// ② 请求转发（反向代理）
// ③ 限流（令牌桶）
```
场景四：游戏服务器
```
// 高并发 + 低延迟 + 长连接
// Netty 负责：实时通信

// 技术要点：
// ① 自定义二进制协议（省流量、快）
// ② UDP 支持（部分游戏，Netty 支持 NioDatagramChannel）
// ③ 帧同步 / 状态同步
```
### 中间件底层
| 中间件                  | 用 Netty 做什么             |
| -------------------- | ----------------------- |
| **Dubbo**            | RPC 网络通信层               |
| **RocketMQ**         | 消息收发网络层                 |
| **gRPC-Java**        | HTTP/2 传输层              |
| **Elasticsearch**    | 节点间通信、HTTP 服务           |
| **Apache Cassandra** | 节点间通信                   |
| **Hadoop/HBase**     | RPC 通信                  |
| **Spring WebFlux**   | HTTP 服务器（Reactor Netty） |
| **Zookeeper**        | 不用 Netty（用原生 NIO）       |
### 实战类比
一个平台，如果使用Netty可以做什么
```
// ① 实时判题推送
// 用户提交代码 → 后台判题 → Netty 推结果（而不是轮询）

// ② 在线编程协同
// 多人同时编辑代码 → WebSocket 实时同步

// ③ 竞赛排行榜实时更新
// 提交 → 判题 → 排行榜变化 → 推送

// ④ 高并发提交接口
// 大量提交请求 → Netty 异步处理 → 削峰

// 技术方案：
// Netty + WebSocket + Protobuf
// 网关层用 Netty 承接高并发
```
### 高频面试题
#### 题目1：Netty和Tomcat的区别
```
// Tomcat：Servlet 容器，HTTP 为主
// Netty：通用网络框架，支持任意协议
// 场景不同：Web 应用 vs 高并发通信
```