## 零拷贝
零拷贝是什么->OS层面的零拷贝->Netty的零拷贝->具体实现->对比辨析->面试高频题
### 零拷贝是什么
零拷贝：减少数据在用户态与内核态之间的拷贝次数，以及CPU参与的数据搬运。
传统的数据拷贝
```
// 场景：从磁盘读文件，通过 Socket 发出去
// 传统方式（4 次拷贝 + 4 次上下文切换）：

read(file, buf);              // ① 内核缓冲区 → 用户缓冲区（拷贝1）
write(socket, buf);           // ② 用户缓冲区 → Socket 缓冲区（拷贝2）
                              // ③ DMA：磁盘 → 内核缓冲区
                              // ④ DMA：Socket 缓冲区 → 网卡

// 数据经历了：
// 磁盘 →[DMA]→ 内核缓冲 →[CPU]→ 用户缓冲 →[CPU]→ Socket缓冲 →[DMA]→ 网卡
//               拷贝③       拷贝①          拷贝②          拷贝④
// 其中 ① ② 是 CPU 拷贝（浪费 CPU）
```
核心问题
```
// ① CPU 参与数据搬运（浪费算力）
// ② 用户态 ↔ 内核态 频繁切换（开销大）
// ③ 内存带宽浪费（数据来回搬）
```
### OS层面的零拷贝
①sendfile (Linux 2.1+)
```
// 数据不经过用户态，内核直接搬
FileChannel.transferTo(position, count, targetChannel);
// 磁盘 →[DMA]→ 内核缓冲 →[DMA]→ Socket缓冲 →[DMA]→ 网卡
// 省掉了"内核 ↔ 用户"两次 CPU 拷贝
// 拷贝次数：3 → 2（不含 DMA 的话，CPU 拷贝 0 次）
```
②mmap+write
```
// mmap：把内核缓冲区映射到用户空间，避免一次拷贝
// 磁盘 →[DMA]→ 内核缓冲（用户态直接访问映射）→[CPU]→ Socket缓冲 →[DMA]→ 网卡
// 适合"需要修改数据"的场景
```
③splice (Linux 2.6+)
```
// 通过管道在两个 fd 之间搬数据，完全在内核态
// 无需 CPU 拷贝（要求一端是管道）
```
Kafka/Nginx
```
// Kafka：用 sendfile 发消息给消费者
// Nginx：用 sendfile 发静态文件
// RocketMQ：mmap（CommitLog 用 MappedByteBuffer）
```
### Netty的零拷贝
Netty的零拷贝≠OS的零拷贝。Netty的零拷贝是应用层减少拷贝对象，主要指数据在Netty在内部流转时减少复制。
Netty零拷贝的五种体现
```
// ① CompositeByteBuf：逻辑组合多个 Buffer，不实际拷贝
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponents(true, header, body);
// header 和 body 逻辑上拼成一个，物理上没合并拷贝

// ② slice()：切片，共享底层内存
ByteBuf slice = buf.slice(0, 10);   // 不拷贝，共享内存

// ③ duplicate()：复制引用，共享内存
ByteBuf dup = buf.duplicate();       // 读写指针独立，数据共享

// ④ wrap()：包装字节数组，不拷贝
ByteBuf wrapped = Unpooled.wrappedBuffer(byteArray);
// 直接用原数组，不复制

// ⑤ FileRegion / transferTo：真正用 OS 的 sendfile
FileRegion region = new DefaultFileRegion(file.getChannel(), 0, length);
ctx.writeAndFlush(region);
// 这个才是调用 OS 零拷贝
```
核心区别
```
// OS 零拷贝：内核态 → 网卡，不经过用户态
// Netty 零拷贝：应用层内部对象共享，减少 ByteBuf 复制
// 只有 FileRegion 是真正调用 OS 零拷贝
```
### 具体实现
CompositeByteBuf 避免合并拷贝
```
// ❌ 传统方式：合并两个 ByteBuf（拷贝）
ByteBuf merged = Unpooled.buffer(header.readableBytes() + body.readableBytes());
merged.writeBytes(header);   // 拷贝
merged.writeBytes(body);     // 拷贝

// ✅ 零拷贝：逻辑组合
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponent(true, header);
composite.addComponent(true, body);
// 不拷贝，逻辑上是一个整体
```
slice 切分共享
```
ByteBuf buf = Unpooled.wrappedBuffer("hello world".getBytes());
ByteBuf hello = buf.slice(0, 5);   // 共享底层数组
ByteBuf world = buf.slice(6, 5);
// 修改 hello 会影响 buf 和 world（共享内存）
```
FileRegion 真零拷贝
```
// 大文件发送
File file = new File("big.zip");
FileInputStream fis = new FileInputStream(file);
FileRegion region = new DefaultFileRegion(fis.getChannel(), 0, file.length());
ctx.writeAndFlush(region);
// 底层调用 sendfile，数据不经过用户态
// 注意：要配合 ChunkedWriteHandler 或直接写
```
### 对比辨析
| 维度       | OS 零拷贝               | Netty 零拷贝            |
| -------- | -------------------- | -------------------- |
| **层面**   | 内核态↔网卡               | 应用层内部                |
| **手段**   | sendfile/mmap/splice | Composite/slice/wrap |
| **减少的是** | 内核↔用户态拷贝             | ByteBuf 复制           |
| **谁受益**  | 文件传输                 | 协议处理/组合              |
| **真零拷贝** | ✅                    | 仅 FileRegion         |
### 面试高频题
#### 题目1：什么是零拷贝
```
// 减少数据拷贝次数、CPU 参与、上下文切换
```
#### 题目2：OS零拷贝有哪几种
```
// sendfile / mmap+write / splice
```
#### 题目3：Netty零拷贝是什么
```
// 应用层减少对象复制
// Composite/slice/wrap/duplicate/FileRegion
```
#### 题目4：Netty零拷贝和OS零拷贝的区别
```
// Netty 主要在应用层；FileRegion 才是 OS 零拷贝
```
#### 题目5：CompositeByteBuf为什么快
```
// 逻辑组合，物理不拷贝
// 避免合并大 Buffer 的内存开销
```
#### 题目6：哪些中间件用了零拷贝
```
// Kafka（sendfile）、Nginx（sendfile）
// RocketMQ（mmap）、Netty（FileRegion）
```
