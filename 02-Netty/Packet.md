## 粘包拆包
粘包拆包是什么->为什么发生->解决方案->解码器详解->选型对比->面试高频题
### 粘包拆包是什么
粘包：多条消息被合并成一个字节块发过来，接收方分不清边界。
拆包：一条消息被拆成多次发过来，接收方一次读不完整。
```
发送方发两条消息：[msg1][msg2]

接收方可能收到：
① 正常：  [msg1]  [msg2]        （两次读到）
② 粘包：  [msg1|msg2]           （一次读到两条）
③ 拆包：  [ms]  [g1] ...        （一条被拆成多次）
④ 半包+粘包混合：[msg1|ms][g2]   （最复杂）
```
关键理解
```
// TCP 是"字节流"协议，没有消息边界概念
// UDP 有边界（但不可靠）
// Netty 基于 TCP → 必须自己定义消息边界
```
### 为什么发生
①发送方原因 粘包
```
// TCP 有 Nagle 算法：小包合并发送
// 应用层多次 write，TCP 攒够一定量才发
socket.write("msg1");   // 可能不发
socket.write("msg2");   // 攒到"msg1msg2"一起发
```
②接收方原因 粘包
```
// 接收方读得慢：缓冲区攒了多条消息，一次读到
byte[] buf = new byte[1024];
socket.read(buf);   // 一次读到 "msg1msg2"
```
③MSS/MTU限制 拆包
```
// 数据超过 MSS（最大报文段长度）→ TCP 分片
// 接收方收到多个 TCP 段 → 一次读不全一条消息
```
本质
```
// 根本原因：TCP 面向"字节流"，消息边界由应用层自己保证
// 不是"bug"，是 TCP 的固有特性
```
### 解决方案
方案一：固定长度（FixedLengthFrameDecoder）
```
// 每条消息固定长度
pipeline.addLast(new FixedLengthFrameDecoder(100));
// 每条消息 100 字节，不够就等，多了就切
// 缺点：消息长度变化时不灵活、浪费带宽
```
方案二：分隔符（DelimiterBasedFrameDecoder）
```
// 用特殊字符分隔消息（如 \r\n、\0）
pipeline.addLast(new DelimiterBasedFrameDecoder(
    1024,                              // 最大长度
    Delimiters.lineDelimiter()         // 分隔符 \r\n
));
// 适合文本协议（HTTP、Redis RESP）
// 缺点：消息内容不能含分隔符
```
方案三：长度字段（LengthFieldBasedFrameDecoder）
```
// 头部带"长度字段"，先读长度，再读内容
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    1024,    // maxFrameLength 最大帧长
    0,       // lengthFieldOffset 长度字段偏移
    4,       // lengthFieldLength 长度字段字节数
    0,       // lengthAdjustment 长度调整
    4        // initialBytesToStrip 解码后剥离的字节数
));
// 适合二进制协议，最常用
```
方案四：自定义解码器
```
// 最灵活，自己解析协议头
// （如 自定义的的 ProtocolDecoder：魔数+版本+类型+长度）
```
### LengthFieldBasedFrameDecoder详解
参数图解
```
// 协议结构：
// +----------+----------+----------+
// | 长度字段  | 实际数据              |
// +----------+----------+----------+
// ↑ offset=0  ↑ length=4
// |<-- 长度字段本身也占字节 -->|

new LengthFieldBasedFrameDecoder(
    maxFrameLength,          // ① 最大帧长度（超了抛异常，防攻击）
    lengthFieldOffset,       // ② 长度字段相对帧头偏移
    lengthFieldLength,       // ③ 长度字段占几个字节（通常 2 或 4）
    lengthAdjustment,        // ④ 长度值需要调整多少（校正）
    initialBytesToStrip      // ⑤ 解码后剥离前几个字节（通常剥离长度字段本身）
);
```
常见配置
```
// 配置①：长度字段在开头，只含数据长度
// [长度][数据]   长度=数据长度
new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 0);
// → 解码后：[长度][数据]（都保留）

new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4);
// → 解码后：[数据]（剥离长度字段）

// 配置②：长度字段含"自身+数据"的总长度
// [长度][数据]   长度=4+数据长度（含长度字段自己）
new LengthFieldBasedFrameDecoder(1024, 0, 4, -4, 4);
// → lengthAdjustment = -4（减去长度字段自身）

// 配置③：长度字段在头部之后
// [魔数][版本][长度][数据]
new LengthFieldBasedFrameDecoder(1024, 6, 4, 0, 10);
// → offset=6（跳过魔数4+版本2），strip=10（剥离全部头）
```
### 实战
```
// 场景：协议如下
// +--------+--------+--------+
// | type   | length |  data  |
// | 1字节   | 2字节  | N字节   |
// +--------+--------+--------+
// length = data 的长度（不含头部）

// 问：LengthFieldBasedFrameDecoder 怎么配？
new LengthFieldBasedFrameDecoder(
    65535,   // maxFrameLength（length 是 2 字节，最大 65535）
    1,       // lengthFieldOffset：type 占 1 字节 → 偏移 1
    2,       // lengthFieldLength：length 占 2 字节
    0,       // lengthAdjustment：length 就是 data 长度，不调整
    3        // initialBytesToStrip：剥掉 type(1) + length(2) = 3
);

// 如果要保留头部（自己再读 type）：
new LengthFieldBasedFrameDecoder(65535, 1, 2, 0, 0);
```
### 面试高频题
#### 题目1：什么是粘包拆包
```
// TCP 字节流无边界
// 粘包：多条合一起
// 拆包：一条被拆开
```
#### 题目2：为什么发生
```
// TCP 是字节流协议，无消息边界
// Nagle 算法、接收缓冲、MSS 分片
```
#### 题目3：为什么说TCP没有粘包问题
```
// TCP 面向字节流，本来就无边界
// "粘包"是应用层的问题，不是 TCP 的问题
```
#### 题目4：怎么选择方案
```
// 文本协议 → 分隔符
// 二进制协议 → 长度字段
// 定长协议 → 固定长度
```