## 内存池
内存池是什么
内存池：预先申请一大块内存，应用按需分配/归还，避免频繁向系统申请/释放内存。Netty的实现类是PooledByteBufAllocator。
```
// ❌ 非池化（UnpooledByteBufAllocator）
ByteBuf buf = Unpooled.buffer(1024);
// 每次 new → 向 JVM 申请内存 → 用完等 GC 回收

// ✅ 池化（PooledByteBufAllocator，默认）
ByteBuf buf = PooledByteBufAllocator.DEFAULT.buffer(1024);
// 从池里拿 → 用完归还池 → 复用，不用 GC
```
### 为什么需要
```
// ① 减少内存分配开销（向 OS/JVM 申请内存是慢操作）
// ② 减少 GC 压力（高频分配/释放在高并发下会打爆 GC）
// ③ 内存复用（Netty 处理海量连接，内存抖动大）
// ④ 减少内存碎片
```
性能对比
```
// 高并发场景：每秒百万请求
// 非池化：每秒百万次分配 → 严重 GC → STW
// 池化：从池里复用 → 几乎无 GC → 低延迟
```
### 内存规格 jemalloc思想
Netty借鉴jemalloc思想，把内存按大小分为4档，减少内部碎片。

|规格|大小范围|说明|
|---|---|---|
|**Tiny**|0 ~ 512B|最小档，用 Subpage 管理|
|**Small**|512B ~ 8KB|用 Subpage 管理|
|**Normal**|8KB ~ 16MB|用 Page 管理|
|**Huge**|> 16MB|单独分配，不池化|
```
// 为什么要分档？
// ① 小内存（Tiny/Small）：一个 Page 切成多份（Subpage）
// ② 大内存（Normal）：直接按 Page 分配
// ③ 超大内存（Huge）：直接向系统申请，不使用池
// → 不同大小用不同策略，减少浪费
```
关键常量
```
// Page：Netty 内存管理的基本单位（默认 8KB）
// Chunk：一组 Page（默认 16MB = 2048 个 Page）
// Arena：一个内存竞技场（负责分配）
// Subpage：Page 的细分（Tiny/Small 用）
```
### 核心结构
层级关系
```
PooledByteBufAllocator
    └─ Arena[]（多个竞技场，按线程绑定）
           └─ PoolChunk（默认 16MB）
                  └─ PoolPage（默认 8KB）
                         └─ PoolSubpage（Tiny/Small 细分）
```
Arena 竞技场
```
// 为避免多线程竞争，Netty 用"多竞技场"分散压力
// 默认：CPU 核数 * 2 个 Arena
// 15 个线程用一个 Arena，减少锁竞争
// 线程绑定 Arena → 类似 ThreadLocal 的思想
```
PoolChunk 内存块
```
// 默认 16MB，是分配的基本大块
// 内部用"完全二叉树"管理 Page 的分配状态
// 每个叶子节点 = 一个 Page，标记是否已分配
// 分配时从树中找到足够的连续 Page
```
PoolSubpage 子页
```
// Tiny/Small 内存（< 8KB）用 Subpage 管理
// 一个 Page（8KB）切成 N 份（如 512B 切 16 份）
// 用 bitmap 记录每份是否已分配
```
### 分配与回收
分配流程
```
// ① 请求 1024 字节
// ② 判断规格：Small（512~8KB）
// ③ 从当前线程的 Arena 找
// ④ 找到合适的 Chunk → Page → Subpage
// ⑤ 分配一小块，返回
```
回收流程
```
// ① ByteBuf 引用计数归零（release()）
// ② 归还到 Subpage/Page/Chunk
// ③ 标记 bitmap / 二叉树状态为"未使用"
// ④ 可被下次分配复用
```
引用计数
```
// Pooled ByteBuf 用引用计数管理生命周期（不用 GC）
ByteBuf buf = PooledByteBufAllocator.DEFAULT.buffer(1024);
System.out.println(buf.refCnt());  // 1
buf.release();
System.out.println(buf.refCnt());  // 0 → 归还池
buf.release();                     // 再 release 报错（已释放）

// ⚠️ 谁最后持有谁负责 release（否则内存泄漏）
// ⚠️ 引用计数是 Netty 内存泄露检测的基础
```
使用建议
```
// ① 默认就是池化（PooledByteBufAllocator.DEFAULT）
// ② 高并发用池化，低并发/短连接可用非池化
// ③ 用完必须 release（引用计数）
// ④ 可用 -Dio.netty.allocator.type=pooled/unpooled 切换
// ⑤ 监控内存泄露：-Dio.netty.leakDetection.level=paranoid
```
### 面试高频题
#### 题目1：什么是内存池
```
// 预分配大块，按需分配/回收，复用内存
// PooledByteBufAllocator
```
#### 题目2：为什么用内存池
```
// 减少分配开销 + 减少 GC + 复用 + 减碎片
```
#### 题目3：内存规格有几档
```
// Tiny(≤512B)/Small(≤8KB)/Normal(≤16MB)/Huge(>16MB)
```
题目4：Arena/Chunk/Page/SubPage关系
```
// Arena > Chunk(16MB) > Page(8KB) > Subpage
```
