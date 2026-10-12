## 心跳与重连机制
为什么需要心跳->心跳实现->心跳设计->断线重连->幂等与重试->面试高频题
### 为什么需要心跳
```
// 长连接的问题：
// ① 连接可能"假死"（TCP 连接还在，但对端已崩溃/网络中断）
// ② 中间设备（NAT/防火墙）会回收空闲连接
// ③ 无法感知对端存活

// TCP 自带 KeepAlive 的问题：
// ① 默认 2 小时才探测（太慢）
// ② 是 TCP 层，应用层不感知
// ③ 操作系统可配置但改系统参数影响大
// → 应用层自己实现心跳
```
心跳的作用
```
// ① 保活：维持连接不被中间设备回收
// ② 探活：检测对端是否存活
// ③ 及时清理死连接（释放资源）
```
### IdleStateHandler
Netty内置的空闲检测处理器，检测多久没读写，触发事件。
```
pipeline.addLast(new IdleStateHandler(
    60,    // readerIdleTime：读空闲（60秒没读 → 触发 READER_IDLE）
    30,    // writerIdleTime：写空闲（30秒没写 → 触发 WRITER_IDLE）
    0      // allIdleTime：读写都空闲（0=禁用）
));
```
触发的事件
```
// 空闲后触发 userEventTriggered 事件：
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            switch (event.state()) {
                case READER_IDLE:
                    // 读空闲 → 可能对端挂了
                    break;
                case WRITER_IDLE:
                    // 写空闲 → 主动发心跳
                    ctx.writeAndFlush(HEARTBEAT);
                    break;
                case ALL_IDLE:
                    break;
            }
        }
    }
}
```
### 心跳设计
典型心跳方案
```
// 客户端：每 30 秒发一次心跳
// 服务端：每 60 秒没收到心跳 → 关闭连接

// 客户端配置：
pipeline.addLast(new IdleStateHandler(0, 30, 0));
// 30 秒没写 → 触发 WRITER_IDLE → 发心跳

// 服务端配置：
pipeline.addLast(new IdleStateHandler(60, 0, 0));
// 60 秒没读 → 触发 READER_IDLE → 关闭连接
```
完整心跳Handler
```
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    private static final ByteBuf HEARTBEAT = ...;

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            if (e.state() == IdleState.WRITER_IDLE) {
                // 发心跳
                ctx.writeAndFlush(HEARTBEAT.duplicate())
                   .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else if (e.state() == IdleState.READER_IDLE) {
                // 超时未收到 → 关闭
                ctx.close();
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}
```
心跳设计要点
```
// ① 心跳间隔：太短浪费流量，太长发现慢
//    推荐：客户端 30s，服务端超时 60s（2 倍）
// ② 心跳内容：尽量小（心跳包）
// ③ 心跳失败：连续 N 次无响应 → 判定断线
```
### 断线重连
客户端重连机制
```
// 连接失败 / 断开 → 定时重连（指数退避）

public class Client {
    private EventLoopGroup group;
    private Bootstrap bootstrap;
    private int retryCount = 0;

    public void connect() {
        ChannelFuture future = bootstrap.connect(host, port);
        future.addListener((ChannelFutureListener) f -> {
            if (f.isSuccess()) {
                retryCount = 0;   // 重置重试次数
                System.out.println("连接成功");
            } else {
                // 重连（指数退避）
                long delay = Math.min(1 << retryCount++, 60);  // 1,2,4,...60秒
                f.channel().eventLoop().schedule(this::connect, delay, TimeUnit.SECONDS);
            }
        });
    }
}
```
断开监听 重连触发
```
pipeline.addLast(new ChannelInboundHandlerAdapter() {
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 连接断开 → 触发重连
        client.connect();
    }
});
```
重连策略对比

|策略|说明|适用|
|---|---|---|
|**固定间隔**|每 5 秒重连|简单场景|
|**指数退避**|1,2,4,8...秒|推荐，避免风暴|
|**带随机抖动**|退避 + 随机|防止同时重连|
```
// 指数退避 + 抖动（防惊群）
long delay = Math.min(1 << retryCount, 60);
long jitter = ThreadLocalRandom.current().nextInt(1000);
eventLoop.schedule(this::connect, delay * 1000 + jitter, MILLISECONDS);
```
### 幂等与重试
```
// ⚠️ 重连后的隐患：
// ① 消息重发 → 重复消费（需幂等）
// ② 请求超时重试 → 重复提交

// 解决：
// ① 消息带唯一 ID（去重）
// ② 服务端幂等处理（如 Redis 记录已处理 ID）
// ③ 请求序列号（对应响应）

// Netty 层面：重连后要重建 Pipeline、重发未完成请求
```
### 面试高频题
#### 题目1：为什么需要心跳
```
// 检测假死 + 保活 + 清理死连接
// TCP KeepAlive 太慢
```
#### 题目2：Netty怎么实现心跳
```
// IdleStateHandler + userEventTriggered
```
#### 题目3：IdleStateHandler三个参数
```
// readerIdleTime / writerIdleTime / allIdleTime
```
#### 题目4：心跳间隔怎么定
```
// 客户端 30s 发，服务端 60s 超时
// 服务端超时 = 客户端间隔的 2 倍
```
#### 题目5：断线怎么重连
```
// channelInactive 监听 + 定时重连
// 指数退避 + 抖动
```
#### 题目6：重连后要注意什么
```
// 幂等（防重复）+ 重建 Pipeline + 重发请求
```
