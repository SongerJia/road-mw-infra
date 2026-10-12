## 通道
Channel是什么->Pipeline结构->Handler入站出站->事件传播->责任链关联->面试高频题
### Channel是什么
Channel（通道）：Netty对网络连接的抽象，代表一个连接。所有I/O操作都通过Channel进行。
```
// 类比：一条电话线
// Channel = 连接的通道
// 读写数据都通过这个通道

// 常见实现：
// NioSocketChannel：客户端 TCP 连接
// NioServerSocketChannel：服务端监听连接
// NioDatagramChannel：UDP
```
核心API
```
// ① 读
ChannelFuture read();        // 读数据（触发入站事件）

// ② 写
ChannelFuture write(Object msg);          // 写数据（触发出站事件）
ChannelFuture writeAndFlush(Object msg);  // 写 + 刷新

// ③ 连接
ChannelFuture connect(SocketAddress);     // 连接
ChannelFuture bind(SocketAddress);        // 绑定端口
ChannelFuture close();                    // 关闭

// ④ 属性
ChannelId id();              // 唯一 ID
ChannelConfig config();      // 配置
ChannelPipeline pipeline();  // 处理器链
EventLoop eventLoop();       // 绑定的 EventLoop
```
Channel的状态
```
// 状态：
// ① 已注册（registered）
// ② 已激活（active，连接建立）
// ③ 已断开（disconnected）
// ④ 已关闭（closed）

// 生命周期（ChannelHandler 回调）：
// channelRegistered → channelActive → channelRead → channelInactive → channelUnregistered
```
### ChannelPipeline 处理器链
ChannelPipeline：ChannelHandler的容器。所有事件（读/写）都经过Pipeline上的Handler链处理。
```
// 类比：流水线
// 数据进入 → Handler1 → Handler2 → Handler3 → 业务
// 一层层处理（责任链）

// 结构（双向链表）：
// head ←→ Handler1 ←→ Handler2 ←→ Handler3 ←→ tail
//  入站（Inbound）：head → 尾（读数据方向）
//  出站（Outbound）：尾 → head（写数据方向）
```
Pipeline结构图
```
入站事件（读）：head → A → B → C → tail
出站事件（写）：tail → C → B → A → head

   head
    ↓（入站）
 ┌────────┐
 │Handler1│  ← 入站处理器（如解码器）
 ├────────┤
 │Handler2│  ← 业务处理器（入站+出站）
 ├────────┤
 │Handler3│  ← 出站处理器（如编码器）
 └────────┘
    ↓（出站）
   tail
```
操作Pipeline
```
ChannelPipeline pipeline = channel.pipeline();

// 添加 Handler（尾部）
pipeline.addLast("handler1", new MyHandler());

// 添加 Handler（头部）
pipeline.addFirst("handler2", new AnotherHandler());

// 指定位置添加
pipeline.addBefore("handler1", "newHandler", new Handler());
pipeline.addAfter("handler1", "newHandler", new Handler());

// 移除
pipeline.remove("handler1");

// 获取
ChannelHandler handler = pipeline.get("handler1");
```
### Handler 入站/出站
入站处理器（ChannelInbondHandler）
```
// 处理"进来的"事件（读数据、连接建立等）
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 处理读到的数据
        System.out.println("收到数据：" + msg);
        // 传给下一个入站 Handler（不调用则不传）
        ctx.fireChannelRead(msg);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 连接建立
        System.out.println("连接建立");
    }
}

// 核心方法：
// channelRegistered：注册
// channelActive：激活（连接建立）
// channelRead：读到数据
// channelReadComplete：读完
// channelInactive：断开
```
出站处理器（ChannelOutbondHandler）
```
// 处理"出去的"事件（写数据、连接等）
public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        // 处理要写出的数据
        System.out.println("要写出：" + msg);
        // 传给下一个出站 Handler
        ctx.write(msg, promise);
    }
}

// 核心方法：
// bind：绑定
// connect：连接
// write：写数据
// flush：刷新
// read：触发读
```
双向处理器
```
// 同时处理入站和出站
public class MyDuplexHandler extends ChannelDuplexHandler {
    // 入站方法 + 出站方法都能重写
}
```
入站vs出站

|对比|入站（Inbound）|出站（Outbound）|
|---|---|---|
|**方向**|进入（读）|出去（写）|
|**传播**|head → tail|tail → head|
|**基类**|ChannelInboundHandlerAdapter|ChannelOutboundHandlerAdapter|
|**例子**|解码器、业务处理|编码器|
### 事件传播
传播机制
```
// 入站事件：
// 从 head 开始，逐个 Handler 传播（通过 ctx.fireXxx）
// 不调用 fireXxx → 停止传播（不再往后传）

// 出站事件：
// 从 tail 开始（或当前 handler 往前），逐个传播（ctx.write）
```
触发传播
```
// 入站：调用 fireChannelRead 才会传给下一个
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 处理...
    ctx.fireChannelRead(msg);   // 传给下一个入站 Handler
    // 不调用 → 到此为止
}

// 出站：调用 ctx.write 才会传给下一个
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    // 处理...
    ctx.write(msg, promise);   // 传给下一个出站 Handler
}
```
为什么要传播
```
// 例：解码器 → 业务处理器
// ① 字节流进入 → 解码器（解码成对象）
// ② 解码器 fireChannelRead(对象) → 传给业务处理器
// ③ 业务处理器处理对象

// 如果不 fireChannelRead → 业务处理器收不到
```
### 责任链关联
```
// Netty Pipeline = 责任链模式！

// 对比：
// 责任链：多个处理器链式传递请求
// Pipeline：多个 Handler 链式处理事件

// 相同点：
// ① 链式结构（链表）
// ② 逐个传递（ctx.fireXxx）
// ③ 可中断（不传递则停止）

// 不同点：
// Netty Pipeline 是"双向"（入站 + 出站）
// 传统责任链通常单向
```
其他责任链
```
// ① Spring MVC 拦截器链（HandlerInterceptor）
// ② Servlet 过滤器链（FilterChain）
// ③ Servlet Response 包装（响应过滤）
// ④ Netty Pipeline ← 本节点
// ⑤ Gateway 过滤器链

// 都是"链式处理 + 逐个传递"的思想
```
### 面试高频题
#### 题目1：Channel是什么
```
// 网络连接的抽象（一条连接一个 Channel）
// 所有 I/O 通过 Channel
```
#### 题目2：Pipeline是什么
```
// Handler 的容器（双向链表）
// 事件经过 Handler 链处理
```
#### 题目3：入站和出站区别
```
// 入站：读数据方向（head→tail）
// 出站：写数据方向（tail→head）
```
#### 题目4：事件怎么传播
```
// fireChannelRead（入站）/ write（出站）
// 调用则传下一个，不调用则停止
```
#### 题目5：Pipeline是责任链吗
```
// 是！双向责任链（入站 + 出站）
```
#### 题目6：为什么用Pipeline
```
// 灵活组合 Handler（解耦）
// 编解码 + 业务分离（各司其职）
```
