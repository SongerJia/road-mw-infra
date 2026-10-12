## 编解码机制
为什么需要编解码->编解码器体系->解码器->编码器->实战->面试高频题
### 为什么需要编解码
```
// 网络传输的是"字节流"（ByteBuf）
// 应用处理的是"Java 对象"
// 需要：字节流 ↔ Java 对象 的转换

// 解码（Decode）：字节流 → Java 对象（入站）
// 编码（Encode）：Java 对象 → 字节流（出站）

// 例：
// 收到 "hello" 字节 → 解码 → String 对象
// 发送 String "world" → 编码 → "world" 字节
```
编解码在Pipeline的位置
```
入站：字节流 → [解码器] → Java对象 → [业务Handler]
出站：[业务Handler] → Java对象 → [编码器] → 字节流
```
### 编解码体系
解码器（Decoder）
```
// ① ByteToMessageDecoder：字节 → 消息（最常用）
public class MyDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 从 in 读字节，解码成对象放入 out
        if (in.readableBytes() < 4) return;   // 数据不够，等下次
        int length = in.readInt();
        // ...
        out.add(decodedObject);
    }
}

// ② MessageToMessageDecoder：消息 → 消息（消息转换）
public class MyDecoder extends MessageToMessageDecoder<ByteBuf> {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) {
        // 一种消息转另一种
    }
}

// ③ ReplayingDecoder：简化版（不用判断可读字节）
```
解码器（Encoder）
```
// ① MessageToByteEncoder：消息 → 字节（最常用）
public class MyEncoder extends MessageToByteEncoder<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        // Java 对象 → 字节
        out.writeInt(msg.getType());
        out.writeBytes(msg.getContent().getBytes());
    }
}

// ② MessageToMessageEncoder：消息 → 消息
// ③ 也有 ByteToMessageEncoder（少用）
```
编解码器（Codec）
```
// 同时实现编解码（入站 + 出站）
public class MyCodec extends CombinedChannelDuplexHandler<Decoder, Encoder> {
    // 组合解码器 + 编码器
}

// 或分别定义（推荐，清晰）：
pipeline.addLast(new MyDecoder());   // 入站
pipeline.addLast(new MyEncoder());   // 出站
```
### 解码器详解
ByteToMessageDecoder原理
```
// 核心：累积 + 解码
public abstract class ByteToMessageDecoder extends ChannelInboundHandlerAdapter {
    // 累积缓冲区（处理半包）
    ByteBuf cumulation;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // ① 累积数据到 cumulation
        cumulation = cumulator.cumulate(ctx.alloc(), cumulation, (ByteBuf) msg);

        // ② 循环解码（可能一次收到多个消息）
        while (true) {
            int outSize = out.size();
            int oldInputLength = cumulation.readableBytes();
            // 调用子类 decode
            decodeRemovalReentryProtection(ctx, cumulation, out);

            // 判断退出：
            // ① 没有产生输出 且 没有消费字节 → 数据不够，等下次
            if (out.size() == outSize && cumulation.readableBytes() == oldInputLength) break;
            // ② 没有可读数据 → 退出
            if (cumulation.readableBytes() == 0) break;
        }
    }
}
```
关键点
```
// ① 累积：网络可能"半包"（一次收到一部分）
//    → 累积到够一个完整消息才解码
// ② 循环：可能"粘包"（一次收到多个消息）
//    → 循环解码多个
// ③ 数据不够：不消费字节，等下次 channelRead
//    → 这就是处理粘包拆包的核心
```
关键方法约定
```
// 子类 decode 方法：
// ① 数据不够 → 不读，直接 return（等下次）
//    if (in.readableBytes() < neededLength) return;
// ② 够 → 读数据，解码，out.add(对象)
```
### 编码器详解
```
public class MyEncoder extends MessageToByteEncoder<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        // ① 判断消息类型（不符合的直接透传）
        // ② 编码到 out（ByteBuf）
        out.writeInt(msg.getLength());
        out.writeBytes(msg.getData());
    }
}

// 注意：
// ① 编码器是出站处理器
// ② 只处理指定类型的消息（泛型）
// ③ 写完后 Netty 自动 flush（或显式 flush）
```
编码器选择

|需求|选择|
|---|---|
|**字节 → 消息**|ByteToMessageDecoder|
|**消息 → 字节**|MessageToByteEncoder|
|**消息 → 消息**|MessageToMessageDecoder/Encoder|
|**附加帧处理**|继承 LengthFieldBasedFrameDecoder|
### 实战（自定义协议）
```
// 协议设计：
// +-------+--------+-------+--------+----------+
// | 魔数   | 版本   | 类型   | 长度   |  数据     |
// | 4字节  | 1字节  | 1字节  | 4字节  |  N字节    |
// +-------+--------+-------+--------+----------+

// ① 解码器
public class ProtocolDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // ① 至少 10 字节（头部）才解码
        if (in.readableBytes() < 10) return;

        // ② 标记读指针（防止数据不够时回退）
        in.markReaderIndex();

        // ③ 读头部
        int magic = in.readInt();       // 魔数
        byte version = in.readByte();   // 版本
        byte type = in.readByte();      // 类型
        int length = in.readInt();      // 数据长度

        // ④ 数据不够 → 回退，等下次
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }

        // ⑤ 读数据
        byte[] data = new byte[length];
        in.readBytes(data);

        // ⑥ 封装成消息
        out.add(new ProtocolMessage(magic, version, type, data));
    }
}

// ② 编码器
public class ProtocolEncoder extends MessageToByteEncoder<ProtocolMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, ProtocolMessage msg, ByteBuf out) {
        out.writeInt(msg.getMagic());
        out.writeByte(msg.getVersion());
        out.writeByte(msg.getType());
        out.writeInt(msg.getData().length);
        out.writeBytes(msg.getData());
    }
}

// ③ Pipeline 组装
pipeline.addLast(new ProtocolDecoder());
pipeline.addLast(new ProtocolEncoder());
pipeline.addLast(new BusinessHandler());
```
### 面试高频题
#### 题目1：为什么需要编解码
```
// 网络传字节流，应用处理对象
// 编解码做转换
```
#### 题目2：解码器基类
```
// ByteToMessageDecoder（字节→消息）
// MessageToMessageDecoder（消息→消息）
```
#### 题目3：编码器基类
```
// MessageToByteEncoder（消息→字节）
```
#### 题目4：ByteToMessageDecoder怎么处理半包
```
// 累积数据（cumulation）
// 数据不够 → 不消费，等下次
```
#### 题目5：怎么处理粘包
```
// while 循环解码（一次读多个消息）
```
#### 题目6：编解码器在Pipeline位置
```
// 最前面（入站：先解码）
// 最前面（出站：最后编码）
```
