## EventLoop
EventLoop是什么->核心职责->EventLoopGroup->线程模型->面试高频题
### EventLoop是什么
EventLoop（事件循环）：Netty的核心，一个EventLoop绑定一个线程，负责处理I/O事件和任务。它是Reactor模式实现。
```
// 类比：餐厅服务员
// 一个服务员（EventLoop）负责几桌客人（Channel）
// 循环：看哪桌有需求（事件）→ 处理 → 继续循环
// 一个服务员一个线程（不切换）
```
核心特点
```
// ① 一个 EventLoop 绑一个线程（生命周期内不变）
// ② 一个 EventLoop 处理多个 Channel（1:N）
// ③ 串行执行（无锁，避免并发问题）
// ④ 负责：I/O 事件 + 任务队列
```
### 核心职责
①处理I/O事件
```
// EventLoop 循环处理 Channel 的 I/O 事件：
// ① OP_ACCEPT：接受连接（ServerSocketChannel）
// ② OP_READ：读数据
// ③ OP_WRITE：写数据
// ④ OP_CONNECT：连接完成

// 底层：Selector（多路复用）
// EventLoop 就是"Select 循环 + 事件分发"
```
②执行任务队列
```
// EventLoop 内部有一个任务队列：
// ① 普通任务：eventLoop.execute(task)
// ② 定时任务：eventLoop.schedule(task, delay, unit)

// 为什么需要任务队列：
// 外部线程不能直接操作 Channel（线程安全）
// → 提交任务到 EventLoop 的队列 → 由 EventLoop 线程执行

// 例：从业务线程写数据
channel.writeAndFlush(data);   // 内部包装成任务，提交到 EventLoop
```
源码视角
```
// NioEventLoop.run（核心循环）
protected void run() {
    for (;;) {   // 无限循环
        // ① 轮询 I/O 事件（select）
        select(wakenUp.getAndSet(false));

        // ② 处理 I/O 事件
        processSelectedKeys();

        // ③ 执行任务队列
        runAllTasks();
    }
}
// 这就是"事件循环"——不断 select → 处理 → 执行任务
```
### EventLoopGroup 线程池+事件循环
```
// EventLoopGroup：一组 EventLoop（类似线程池）
// 作用：管理多个 EventLoop，分配 Channel

// 两种：
// ① NioEventLoopGroup：处理 I/O（默认 CPU 核数 * 2 个 EventLoop）
// ② DefaultEventExecutorGroup：处理业务（可选）

// 创建：
EventLoopGroup bossGroup = new NioEventLoopGroup(1);      // 1 个（处理连接）
EventLoopGroup workerGroup = new NioEventLoopGroup();     // 默认（处理读写）
```
Channel怎么分配到EventLoop
```
// ① 顺序分配（轮询）：
// 新 Channel → 按 EventLoop[] 数组顺序分配
// next = (next + 1) % length

// ② 一个 Channel 绑定一个 EventLoop（生命周期内不变）
// ③ 一个 EventLoop 处理多个 Channel

// 效果：
// 同一 Channel 的所有事件由同一线程处理（串行，无锁）
```
### 线程模型
Reactor模式
```
① 单 Reactor 单线程（最简）：
   ┌─────────────┐
   │  Reactor    │  ← 1 个线程：accept + read + 业务
   └─────────────┘
   问题：业务慢 → 阻塞所有 I/O

② 单 Reactor 多线程：
   Reactor（accept + read）
      ↓ 业务 → 线程池
   问题：Reactor 还是单线程（高并发瓶颈）

③ 主从 Reactor 多线程（Netty 默认）：
   MainReactor（bossGroup：只 accept）
      ↓
   SubReactor（workerGroup：read/write）
      ↓ 业务 → 业务线程池
   ✅ 高并发
```
Netty的主从Reactor
```
// Netty 默认"主从 Reactor 多线程"：

// ① bossGroup（主 Reactor）
//    只负责 accept 连接
//    通常 1 个 EventLoop
EventLoopGroup bossGroup = new NioEventLoopGroup(1);

// ② workerGroup（从 Reactor）
//    负责 read/write
//    默认 CPU 核数 * 2 个
EventLoopGroup workerGroup = new NioEventLoopGroup();

// ③ 绑定
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)   // 主从
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        protected void initChannel(SocketChannel ch) {
            ch.pipeline().addLast(new MyHandler());
        }
    });
```
为什么bossGroup只需要1个
```
// accept 操作非常快（建立连接）
// 一个线程足够处理海量连接（select 多路复用）
// 真正的读写由 workerGroup 处理

// 若 accept 有耗时操作（如 SSL 握手），可加多个
```
### 为什么不设更多线程
```
// Netty 线程数不是越多越好：
// ① workerGroup 默认 = CPU 核数 * 2
// ② 为什么是 2 倍：兼顾 IO 等待和计算
// ③ 线程太多 → 上下文切换开销

// 最佳实践：
// ① I/O 线程（workerGroup）尽量少
// ② 业务逻辑（耗时）放"业务线程池"
//    避免阻塞 I/O 线程
```
业务线程池
```
// 场景：Handler 里有耗时操作（DB 查询、RPC）
// ❌ 直接执行会阻塞 I/O 线程

// ✅ 业务线程池
EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(16);

ch.pipeline().addLast(businessGroup,   // 业务组
    new MyBusinessHandler());
// Handler 在业务线程池执行（不阻塞 I/O）
```
### 面试高频题
#### 题目1：EventLoop是什么
```
// 事件循环，绑定一个线程
// 处理 I/O 事件 + 任务队列
```
#### 题目2：EventLoop和EventLoopGroup
```
// EventLoop：单个循环（线程）
// Group：一组循环（线程池 + 分配）
```
#### 题目3：一个channel绑定几个EventLoop
```
// 一个（生命周期内不变）
// 一个 EventLoop 处理多个 Channel
```
#### 题目4：Netty线程模型
```
// 主从 Reactor 多线程
// bossGroup（accept）+ workerGroup（read/write）
```
#### 题目5：为什么串行无锁
```
// 同一 Channel 的事件由同一线程处理
// 串行执行，不用加锁
```
#### 题目6：workerGroup几个线程
```
// 默认 CPU 核数 * 2
// 耗时业务放业务线程池
```

