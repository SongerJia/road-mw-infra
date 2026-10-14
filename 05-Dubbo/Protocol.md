## 协议与序列化
协议格式->协议种类->序列化选择->传输层->线程模型->面试高频题
### Dubbo协议格式
Dubbo协议是一个二进制协议，头部固定16字节。
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Magic        | Flag |Status|          Request ID        |
|       (2 bytes)     |(1B)  |(1B)  |          (8 bytes)         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Request ID (continued)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Body Length (4 bytes)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Body (序列化后的数据)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
各字段含义

|字段|长度|说明|
|---|---|---|
|**Magic**|2 字节|魔数 `0xdabb`（协议识别）|
|**Flag**|1 字节|请求/响应标志 + 序列化 ID + 事件标志|
|**Status**|1 字节|响应状态码|
|**Request ID**|8 字节|请求 ID（匹配请求和响应）|
|**Body Length**|4 字节|数据长度|
|**Body**|可变|序列化后的数据|
Flag字段详解
```
Flag（1 字节 = 8 位）：
├─ bit 7   ：Req/Res（请求=1 / 响应=0）
├─ bit 6   ：TwoWay（是否需要响应）
├─ bit 5   ：Event（是否心跳事件）
├─ bit 4-0 ：序列化 ID（hessian2=2 等）
```
```
// 为什么这样设计？
// ① 魔数：快速识别是不是 Dubbo 协议
// ② Flag 里带序列化 ID：接收方知道怎么反序列化
// ③ Request ID：异步请求响应匹配（Consumer 发多个请求，靠 ID 配对）
```
编解码源码
```
// org.apache.dubbo.remoting.exchange.codec.ExchangeCodec
// 编码
protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) {
    // ① 写魔数
    buffer.writeShort(MAGIC);
    // ② 写 flag（请求 + 序列化ID）
    buffer.setByte(2, flag);
    // ③ 写 requestId
    buffer.writeLong(req.getId());
    // ④ 写 body（序列化后）
    ...
}

// 解码
protected Object decode(Channel channel, ChannelBuffer buffer) {
    // ① 读魔数 → 校验
    short magic = buffer.readShort();
    // ② 读 flag / status / requestId / length
    // ③ 读 body（反序列化）
}
```
### 协议种类
| 协议             | 连接    | 说明             | 场景          |
| -------------- | ----- | -------------- | ----------- |
| **dubbo**      | 单一长连接 | 默认，NIO，性能最好    | 高并发小数据      |
| **rmi**        | 多连接   | JDK 自带         | 传统 Java RMI |
| **hessian**    | 多连接   | HTTP + Hessian | 跨语言         |
| **http**       | 短连接   | HTTP 表单        | 提供 REST 服务  |
| **webservice** | 短连接   | SOAP           | 跨系统         |
| **thrift**     | -     | 跨语言 RPC        | 多语言         |
| **rest**       | -     | RESTful        | 微服务         |
| **grpc**       | -     | HTTP/2         | 云原生（3.x 重点） |
```
// 默认是 dubbo 协议：
// ① 单一长连接（一个 Consumer 对一个 Provider 一个连接）
// ② 单连接可能成为瓶颈（大流量时）
// ③ 适合小数据量高并发
```
单一长连接的问题
```
// dubbo 协议默认单连接：
// ① 优点：连接少、省资源
// ② 缺点：大数据量可能阻塞（一个连接串行）

// 解决：配置多连接
// connections="10"  → 建立多个连接
```
为什么默认Dubbo协议
```
// ① 二进制协议（比 HTTP 省）
// ② 长连接（省建连开销）
// ③ 异步（性能好）
// ④ 定制（带 Request ID，支持异步）
```
### 序列化选择
| 序列化          | 优点       | 缺点           | 场景     |
| ------------ | -------- | ------------ | ------ |
| **Hessian2** | 跨语言、默认   | 性能中等         | 默认     |
| **Java**     | 实现简单     | 仅 Java、慢、体积大 | 不推荐    |
| **JSON**     | 可读       | 体积大、慢        | 调试     |
| **Kryo**     | 性能高      | 需配、有安全记录     | 高性能场景  |
| **FST**      | 性能高      | 需配           | 高性能场景  |
| **Protobuf** | 极致性能、跨语言 | 需写 .proto    | 高性能跨语言 |
源码
```
// 序列化接口
@SPI("hessian2")     // 默认 Hessian2
public interface Serialization {
    byte getContentTypeId();
    ObjectOutput serialize(URL url, OutputStream output);
    ObjectInput deserialize(URL url, InputStream input);
}
```
```
// 配置：
<dubbo:protocol name="dubbo" serialization="kryo"/>
// 或
@Reference(serialization = "hessian2")
```
Hessian2为什么是默认
```
// ① 跨语言（
// ② 自描述（不用 .proto / IDL）
// ③ 性能/体积平衡好
// ④ 使用简单（无需额外配置）
```
### 传输层 Netty
```
// Dubbo 底层用 Netty 做网络传输
// ① Client 端：NettyClient
// ② Server 端：NettyServer
// ③ 编解码器：DubboCodec
```
```
// Pipeline 组装（简化）：
ch.pipeline()
  .addLast("decoder", new DubboCountCodec())
  .addLast("encoder", new DubboCodec())
  .addLast("handler", new NettyHandler());
  
// Dubbo = Netty（传输）+ 自定义协议（编解码）+ RPC 语义
```
信息交换层 Exchange
```
// 在 Transport 之上，Dubbo 抽象了 Exchange 层
// ① 请求-响应模型（Request/Response）
// ② 异步请求 → Future 结果
// Request/Response 通过 Request ID 匹配

// ExchangeClient：
//   request(Object) → ResponseFuture
//   ResponseFuture.get() → 阻塞拿结果
```
```
// DubboInvoker.doInvoke 里：
ResponseFuture future = currentClient.request(inv, timeout);
return (Result) future.get();      // 同步等结果
// 也支持异步：future 加回调
```
### 线程模型
```
// Dubbo 的线程分为两类：
// ① IO 线程（Netty EventLoop）：处理网络读写
// ② 业务线程池：处理具体业务逻辑
```
派发策略 Dispatcher

|策略|说明|
|---|---|
|**all**（默认）|所有消息都派发到业务线程池|
|**direct**|所有消息都在 IO 线程处理（不派发）|
|**message**|只有请求消息派发，其他在 IO 线程|
|**execution**|只有请求消息派发（响应/心跳等在 IO 线程）|
|**connection**|连接建立/断开派发，其他在 IO 线程|
```
<dubbo:protocol name="dubbo" dispatcher="all"
                threads="200" threadpool="fixed"/>
```
为什么要线程池
```
// fixed（默认）：固定大小
// cached：可缓存
// limited：可伸缩（但线程数只增不减）
// eager：优先创建（任务来了先建线程）
```
```
// 关键：业务线程池满了怎么办？
// → 拒绝策略（默认 AbortPolicy，抛异常）
// → 可配队列大小
```

