## Netty vs NIO
原生NIO->原生NIO的痛点->Netty的改进->对比总结->高频面试题
### 原生NIO
```
// JDK NIO 三大核心：
// ① Channel（通道）
// ② Buffer（缓冲区）
// ③ Selector（多路复用器）

// 原生 NIO 服务端示例（简化）：
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.bind(new InetSocketAddress(8080));
Selector selector = Selector.open();
server.register(selector, SelectionKey.OP_ACCEPT);   // 注册

while (true) {
    selector.select();                    // 阻塞等事件
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> it = keys.iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();                      // ⚠️ 必须手动删除
        if (key.isAcceptable()) { /* 处理连接 */ }
        else if (key.isReadable()) { /* 处理读 */ }
    }
}
```
### 原生NIO痛点
```
// ① API 复杂，学习成本高
//    要自己处理 Channel/Buffer/Selector 三者关系

// ② 空轮询 bug（最坑）
//    Linux epoll 在特定情况下 select() 立即返回但无事件
//    → CPU 100% 空转

// ③ 粘包拆包要自己处理
//    没有现成的解码器

// ④ 半包读没读完要自己处理
//    ByteBuffer 读写指针翻转要小心

// ⑤ 多线程编程复杂
//    要自己管理 Selector 线程、业务线程

// ⑥ 资源管理容易泄漏
//    忘记 close / 引用计数

// ⑦ 异常处理麻烦
//    网络异常要自己兜底
```
举例：Buffer的坑
```
// 原生 ByteBuffer：读写共用指针
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.put("data".getBytes());
buf.flip();               // 切换读模式（必须）
while (buf.hasRemaining()) {
    buf.get();            // 忘了 flip 就读不到
}
buf.clear();              // 复位（必须）
// 到处是 flip/clear，容易出错

// Netty ByteBuf：读写双指针
ByteBuf buf = Unpooled.buffer(1024);
buf.writeBytes("data".getBytes());
buf.readByte();           // 读写指针独立，不用 flip
```
### Netty的改进点
| 原生 NIO 痛点   | Netty 改进                         |
| ----------- | -------------------------------- |
| **API 复杂**  | 简洁 API（Bootstrap/ChannelHandler） |
| **空轮询 bug** | 检测到空轮询 → 重建 Selector             |
| **粘包拆包**    | 内置解码器（LengthField 等）             |
| **半包处理**    | ByteToMessageDecoder 累积机制        |
| **多线程复杂**   | EventLoop 模型（主从 Reactor）         |
| **资源泄漏**    | 引用计数 + 泄漏检测                      |
| **异常处理**    | Pipeline 统一异常传播                  |
| **ByteBuf** | 读写双指针，无需 flip                    |
代码对比
```
// ❌ 原生 NIO：几十行样板代码

// ✅ Netty：优雅
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(boss, worker)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {
     protected void initChannel(SocketChannel ch) {
         ch.pipeline()
           .addLast(new LengthFieldBasedFrameDecoder(1024,0,4,0,4))
           .addLast(new StringDecoder())
           .addLast(new MyBusinessHandler());
     }
 });
b.bind(8080).sync();
```
### 更全的对比
| 维度       | 原生 NIO   | Netty                  |
| -------- | -------- | ---------------------- |
| **开发效率** | 低        | 高                      |
| **稳定性**  | 低（bug 多） | 高（久经考验）                |
| **性能**   | 高（但难优化）  | 更高（零拷贝/内存池）            |
| **线程模型** | 自己实现     | 主从 Reactor 现成          |
| **协议支持** | 无        | HTTP/WebSocket/SSL 等   |
| **生态**   | 无        | Dubbo/RocketMQ/gRPC 底层 |
| **内存管理** | JVM GC   | 池化 + 引用计数              |
| **粘包拆包** | 自己写      | 内置                     |
### 高频面试题
#### 题目1：空轮询的bug是什么
```
// Linux epoll 上 select() 立即返回但无事件
// CPU 100%
// Netty 检测并重建 selector
```
#### 题目2：都用Netty代替NIO吗
```
// 不一定：极简场景可直接 NIO
// 但生产级服务几乎都用 Netty
```
