## 字节缓冲区
ByteBuf是什么->ByteBuf vs ByteBuffer->读写指针->内存类型->零拷贝->引用计数->面试高频题
### ByteBuf是什么
ByteBuf：Netty的字节缓冲区，替代JDK的ByteBuffer。读写指针分离、自动扩容、支持池化和零拷贝，性能更好且更好用。
```
// JDK ByteBuffer 的痛点：
// ① 读写共用一个 position（切换麻烦：flip）
// ② 容量固定（不能自动扩容）
// ③ 功能少（缺很多实用方法）

// ByteBuf 的改进：
// ① 读写指针分离（readerIndex / writerIndex）
// ② 自动扩容
// ③ 方法丰富
// ④ 池化（复用）
// ⑤ 零拷贝
```
### ByteBuf vs ByteBuffer
读写指针分离
```
// ByteBuffer（一个指针，要 flip 切换）：
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put("hello".getBytes());  // 写模式
buffer.flip();                   // 切换读模式（麻烦！）
buffer.get();                    // 读

// ByteBuf（读写指针分离）：
ByteBuf buf = Unpooled.buffer(10);
buf.writeBytes("hello".getBytes());  // 写：writerIndex++
buf.readBytes(5);                    // 读：readerIndex++
// 不需要 flip！写和读用不同指针
```
三个指针
```
capacity：总容量
┌─────────────────────────────────────────────────────┐
│     已丢弃    │     可读     │     可写      │ 可扩容  │
└─────────────────────────────────────────────────────┘
        ↑              ↑
   readerIndex    writerIndex
   （已读丢弃）    （已写入）

推导：
① readableBytes = writerIndex - readerIndex（可读字节数）
② writableBytes = capacity - writerIndex（可写字节数）
③ discardableBytes = readerIndex（已读可丢弃）
```
对比表

|对比|ByteBuffer|ByteBuf|
|---|---|---|
|**读写指针**|共用一个（要 flip）|分离（不用 flip）|
|**扩容**|❌ 固定|✅ 自动|
|**池化**|❌|✅|
|**零拷贝**|限制多|✅ 丰富|
|**引用计数**|❌|✅|
|**API**|少|丰富|
### 内存类型 堆内/堆外/池化
堆内 vs 堆外
```
// ① 堆内（Heap）：分配在 JVM 堆（byte[]）
ByteBuf heapBuf = Unpooled.buffer(10);         // 堆内
// 优点：分配快、易 GC
// 缺点：Socket IO 要拷贝到堆外（多一次拷贝）

// ② 堆外（Direct）：分配在直接内存
ByteBuf directBuf = Unpooled.directBuffer(10); // 堆外
// 优点：Socket IO 直接读写（零拷贝）
// 缺点：分配慢、GC 不直接管理
```
池化 vs 非池化
```
// ① 池化（Pooled）：从内存池取（复用）
ByteBuf pooled = PooledByteBufAllocator.DEFAULT.buffer(10);
// 优点：复用（减少分配/GC）
// 缺点：实现复杂

// ② 非池化（Unpooled）：每次新建
ByteBuf unpooled = Unpooled.buffer(10);
// 优点：简单
// 缺点：频繁分配（GC 压力）

// Netty 默认：
// 堆外 + 池化（PooledByteBufAllocator.DEFAULT.directBuffer()）
// 性能最好
```
四种组合

|类型|堆内|堆外|
|---|---|---|
|**池化**|PooledHeapByteBuf|PooledDirectByteBuf|
|**非池化**|UnpooledHeapByteBuf|UnpooledDirectByteBuf|
### 零拷贝
Netty零拷贝的三种
```
// ① ByteBuf 内部：读写指针分离
// 读和写不冲突，不用拷贝（不是真正意义上的"零拷贝"）

// ② CompositeByteBuf：组合多个 ByteBuf（逻辑合并）
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponent(true, header);
composite.addComponent(true, body);
// 效果：逻辑上是一个 ByteBuf，物理上多个（无拷贝）

// ③ FileRegion：文件传输零拷贝
// 文件 → 直接发到网络（sendfile）
FileRegion region = new DefaultFileRegion(fileChannel, 0, fileLength);
channel.writeAndFlush(region);
// 不走应用层（零拷贝，下载场景）
```
对比JDK零拷贝
```
// JDK：FileChannel.transferTo（sendfile）
// Netty：FileRegion（也是 sendfile）+ CompositeByteBuf

// 最大的零拷贝：
// CompositeByteBuf（逻辑合并，避免多次拷贝）
```
slice（切片）和copy（复制）
```
// slice：共享底层内存（零拷贝）
ByteBuf slice = buf.slice(0, 10);   // 与原 buf 共享内存（不拷贝）
// 修改 slice → 影响原 buf

// copy：真正拷贝（独立）
ByteBuf copy = buf.copy(0, 10);     // 独立副本（拷贝）
// 修改 copy → 不影响原 buf

// duplicate：复制引用（共享内存）
ByteBuf dup = buf.duplicate();       // 新读写指针，共享内存
```
### 引用计数（内存管理）
```
// ByteBuf 用"引用计数"管理内存（类似 C++ 智能指针）
// 每个 ByteBuf 有 refCnt（引用计数）

// 规则：
// ① 创建时 refCnt = 1
// ② retain()：refCnt++
// ③ release()：refCnt--
// ④ refCnt = 0 → 回收（归还池/释放内存）

ByteBuf buf = Unpooled.buffer();  // refCnt = 1
buf.retain();                     // refCnt = 2
buf.release();                    // refCnt = 1
buf.release();                    // refCnt = 0 → 回收
```
谁负责释放
```
// 原则：谁最后使用，谁释放

// ① 入站消息（读）：Handler 负责释放
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    try {
        // 处理
    } finally {
        buf.release();   // 入站消息要释放
    }
}

// ② SimpleChannelInboundHandler 自动释放
// ③ 出站消息：Netty 释放（不用管）
// ④ 传下去的（fireChannelRead）：下一个 Handler 释放

// 内存泄漏排查：
// JVM 参数：-Dio.netty.leakDetection.level=paranoid
// 检测未释放的 ByteBuf
```
### 常用API
```
ByteBuf buf = Unpooled.buffer(10);

// 写
buf.writeByte(1);
buf.writeInt(100);
buf.writeBytes("hello".getBytes());
buf.writeCharSequence("world", CharsetUtil.UTF_8);

// 读
buf.readByte();
buf.readInt();
buf.readBytes(5);

// get/set（不移动指针）
buf.getByte(0);      // 读下标 0（readerIndex 不变）
buf.setByte(0, 2);   // 设置下标 0

// 索引
buf.readerIndex();   // 读指针位置
buf.writerIndex();   // 写指针位置
buf.readableBytes(); // 可读字节数
buf.writableBytes(); // 可写字节数
buf.capacity();      // 容量

// 查找
buf.indexOf(0, 5, (byte) 'a');   // 查找字节

// 标记
buf.markReaderIndex();    // 标记读指针
buf.resetReaderIndex();   // 回到标记
```
### 面试高频题
#### 题目1：ByteBuf和ByteBuffer区别
```
// 读写指针分离（不用 flip）
// 自动扩容、池化、零拷贝、引用计数
```
#### 题目2：ByteBuf的指针
```
// readerIndex（读）、writerIndex（写）
// readableBytes = writer - reader
```
#### 题目3：堆外 vs 堆内
```
// 堆内：分配快，IO 要拷贝到堆外
// 堆外：IO 直接读写（零拷贝），分配慢
```
#### 题目4：Netty零拷贝
```
// ① 读写指针分离
// ② CompositeByteBuf（逻辑合并）
// ③ FileRegion（文件传输 sendfile）
// ④ slice（共享内存）
```
#### 题目5：引用计数是什么
```
// refCnt 管理内存（retain/release）
// 归 0 回收
// 入站消息要 release
```
#### 题目6：怎么排查内存泄漏
```
// io.netty.leakDetection.level=paranoid
// 检测未释放的 ByteBuf
```
