## 通道处理器
Handler是什么->@Sharable线程安全->生命周期->异常处理->内置Handler->面试高频题
### ChannelHandler是什么
ChannelHandler：Netty中处理I/O事件和数据的组件。所有业务逻辑、编解码都在Handler中实现，通过Pipeline串联。
```
// Handler 分类（：
// ① ChannelInboundHandlerAdapter：入站处理（读）
// ② ChannelOutboundHandlerAdapter：出站处理（写）
// ③ ChannelDuplexHandler：双向
// ④ SimpleChannelInboundHandler：简化版入站（自动释放）

// 类比：流水线上的工人
// 每个 Handler 负责一道工序（解码/校验/业务/编码）
```
### @Sharable 线程安全
Handler能被多个Channel共享吗
```
//不能，需要加注解
// Channel 各自有 Pipeline，但 Handler 可以复用
// 默认：一个 Handler 实例只能被一个 Channel 用（非 @Sharable）

// ❌ 错误：把非 @Sharable 的 Handler 共享给多个 Channel
ChannelHandler handler = new MyHandler();
// channel1.pipeline().addLast(handler);
// channel2.pipeline().addLast(handler);  // ❌ 抛异常！
```
@Sharable注解
```
// 标记 Handler 可以被多个 Channel 共享
// 前提：Handler 必须"无状态"（线程安全）

@Sharable
public class MySharedHandler extends ChannelInboundHandlerAdapter {
    // ❌ 不能有可变状态（多个 Channel 共享会冲突）
    // private int counter = 0;   // 危险！

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 无状态处理（安全）
    }
}
```
为什么默认不共享
```
// ① 多数 Handler 有状态（如解码器的缓冲区）
// ② 多个 Channel 共享有状态 Handler → 数据错乱
// ③ 所以默认限制共享，@Sharable 显式声明"我无状态"

// 什么时候用 @Sharable：
// ① 无状态 Handler（如统计、日志）
// ② 编码器通常无状态（可 @Sharable）
```
什么时候不能@Sharable
```
// ① 编解码器（有解码状态，如半包缓冲）
// ② 业务 Handler 有成员变量
// → 每个 Channel 新建一个 Handler 实例

// 例：ProtoBuf 的 ProtobufDecoder 无状态 → @Sharable
//     LengthFieldBasedFrameDecoder 有状态 → 不 @Sharable
```
### Handler生命周期
```
// Handler 的通知方法（Pipeline 调用）：

// ① handlerAdded：Handler 被添加到 Pipeline
// ② channelRegistered：Channel 注册到 EventLoop
// ③ channelActive：Channel 激活（连接建立）
// ④ channelRead：读到数据
// ⑤ channelReadComplete：一批数据读完
// ⑥ channelInactive：Channel 断开
// ⑦ channelUnregistered：Channel 注销
// ⑧ handlerRemoved：Handler 从 Pipeline 移除

// 顺序：
// handlerAdded → channelRegistered → channelActive
// → channelRead → channelReadComplete → channelInactive
// → channelUnregistered → handlerRemoved
```
典型用法
```
public class MyHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        // Handler 加入 Pipeline（初始化资源）
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 连接建立（可主动发数据）
        ctx.writeAndFlush("welcome");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 读数据
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 断开（清理资源）
    }
}
```
### 异常处理
executionCaught
```
// Handler 处理异常的方法
public class MyHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // 处理异常
        cause.printStackTrace();
        // 关闭连接（推荐）
        ctx.close();
        // 或传给下一个 Handler
        // ctx.fireExceptionCaught(cause);
    }
}
```
异常传播
```
// 异常沿入站链传播（head → tail）
// 若某个 Handler 处理了（不 fire），则停止
// 若都处理不了 → 尾部的 DefaultChannelPipeline 记录日志 + 关闭

// 注意：
// ① 入站异常从当前 Handler 向后传
// ② 出站异常特定（如 write 的异常通过 ChannelFuture 感知）
```
为什么异常一定要处理
```
// ❌ 不处理 exceptionCaught：
// 异常被记录但连接可能不关闭 → 资源泄漏

// ✅ 处理：
// ① 记录日志
// ② 关闭连接（ctx.close）
// ③ 或标记故障
```
### 内置常用Handler
| Handler                           | 作用           | 方向     |
| --------------------------------- | ------------ | ------ |
| **LengthFieldBasedFrameDecoder**  | 长度字段拆包       | 入站（解码） |
| **LineBasedFrameDecoder**         | 换行符拆包        | 入站     |
| **DelimiterBasedFrameDecoder**    | 分隔符拆包        | 入站     |
| **StringDecoder / StringEncoder** | 字符串编解码       | 入站/出站  |
| **IdleStateHandler**              | 空闲检测（心跳）     | 双向     |
| **LoggingHandler**                | 日志           | 双向     |
| **SSLHandler**                    | SSL/TLS 加密   | 双向     |
| **HttpServerCodec**               | HTTP 编解码     | 双向     |
| **ProtobufDecoder / Encoder**     | ProtoBuf 编解码 | 入站/出站  |
SimpleChannelInBondHandler 自动释放
```
// 简化版入站 Handler（自动释放 msg 的引用计数）
public class MyHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        // 处理 msg
        // 方法返回后自动 release（不用手动释放）
    }
    // 泛型指定消息类型（自动类型过滤）
}
// 对比 ChannelInboundHandlerAdapter：需手动 ReferenceCountUtil.release
```
### 面试高频题
#### 题目1：@Sharable是什么
```
// 标记 Handler 可被多 Channel 共享
// 前提：无状态（线程安全）
```
#### 题目2：为什么不默认共享
```
// 多数 Handler 有状态（编解码缓冲）
// 共享会数据错乱
```
#### 题目3：哪些Handler能@Sharable
```
// 无状态的（如 ProtobufDecoder、LoggingHandler）
// 有状态的（如 LengthFieldBasedFrameDecoder）不能
```
#### 题目4：Handler生命周期
```
// handlerAdded → channelRegistered → channelActive
// → channelRead → channelInactive → handlerRemoved
```
#### 题目5：异常怎么处理
```
// exceptionCaught
// 记录日志 + 关闭连接（防泄漏）
```


