## 无锁设计
无锁是什么->串行话设计->CAS无锁->volatile/内存屏障->其他优化->面试高频题
### 无锁设计是什么
无锁设计：Netty是尽量不用锁，用串行化+CAS+volatile来减少线程竞争的开销。
```
// 锁的问题：
// ① 线程阻塞/唤醒 → 上下文切换开销
// ② 竞争激烈时性能急剧下降
// ③ 可能死锁

// Netty 的思路：
// ① 串行化：一个 Channel 的事件只由一个 EventLoop 线程处理 → 天然无竞争
// ② CAS：必要的原子操作用 CAS（无锁）
// ③ volatile：保证可见性，避免重排序
```
### 串行化设计
一个Channel的所有I/O事件，都是由一个EventLoop线程处理，单线程内是串行的，天然无锁。
为什么能无锁
```
// 关键：一个 Channel 绑定一个 EventLoop 线程
// 该 Channel 的所有事件（读、写、连接）都在这个线程里串行执行

// 后果：
// ① 单个 Channel 的业务处理器（Handler）无需加锁（单线程串行）
// ② 线程之间不共享 Channel 状态 → 无竞争
```
图示
```
EventLoop-1 (线程)
    ├─ Channel-A 的所有事件（串行）
    └─ Channel-B 的所有事件（串行）

EventLoop-2 (线程)
    └─ Channel-C 的所有事件（串行）

// Channel-A 和 Channel-B 在同一线程 → 串行，无锁
// Channel-C 在另一线程 → 不共享状态，无锁
```
代价
```
// ⚠️ 串行化设计的前提：Handler 不能做耗时操作
// 因为一个 EventLoop 管多个 Channel
// 如果某个 Handler 阻塞，同 EventLoop 的其他 Channel 都被拖累

// 解决：耗时操作放业务线程池
```
### CAS无锁
```
// CAS：Compare And Swap
// 比较内存值是否等于预期值，相等则更新
// 由 CPU 指令（cmpxchg）保证原子性，无需锁

// 伪代码：
boolean cas(Object obj, int expect, int update) {
    if (obj.value == expect) {
        obj.value = update;
        return true;
    }
    return false;
}
```
Netty里的CAS
```
// ① ByteBuf 引用计数（无锁）
// AbstractReferenceCountedByteBuf
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater =
    AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");

public boolean release() {
    // CAS 递减引用计数，无锁
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt == 0) return false;
        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - 1)) break;
    }
    return true;
}

// ② 原子域更新（AtomicIntegerFieldUpdater）
// 相比 AtomicInteger 对象，直接更新字段，省内存、快
```
### volatile和内存屏障
```
// volatile：
// ① 保证可见性（一个线程改，其他线程立即可见）
// ② 禁止指令重排序（内存屏障）

// Netty 用途：
// ① 状态标志、开关
// ② 配合 CAS

// 内存屏障：CPU 层面禁止重排的指令
// Netty 通过 volatile 字段实现"轻量级同步"
```
### 其他无锁优化
①MpscQueue 多生产者单消费者队列
```
// Netty 的消息队列：多生产者单消费者，无锁队列
// 场景：多个业务线程向一个 EventLoop 投递任务
// 用 JCTools 的 MpscQueue → 无锁入队/出队
```
②空轮询检测 Selector优化
```
// JDK 的 epoll 空轮询 bug：selector.select() 可能立即返回但无事件
// Netty 检测到空轮询 → 重建 selector 或重新注册

// 也是"无锁"思路：避免无效自旋消耗 CPU
```
③消除锁的替代方案总结

|问题|加锁方案|Netty 无锁方案|
|---|---|---|
|**Channel 状态**|synchronized|串行化（单线程）|
|**引用计数**|synchronized|CAS|
|**多线程传任务**|BlockingQueue|MpscQueue|
|**标志位**|synchronized|volatile|
### 面试高频题
#### 题目1：Netty为什么能无锁
```
// ① 串行化设计（Channel 绑定单线程）
// ② CAS（引用计数等）
// ③ volatile
```
#### 题目2：串行化设计是什么
```
// 一个 Channel 的事件由同一个 EventLoop 线程处理
// 单线程串行 → 无竞争 → 无锁
```
#### 题目3：串行化的前提/代价
```
// Handler 不能阻塞
// 否则拖累同 EventLoop 的其他 Channel
// 耗时操作放业务线程池
```
#### 题目4：引用计数怎么实现无锁
```
// AtomicIntegerFieldUpdater + CAS 自旋
```
