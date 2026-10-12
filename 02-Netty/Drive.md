## 事件驱动与异步
事件驱动是什么->ChannelFuture->异步回调->Future vs Promise->异步实践->面试高频题
### 事件驱动是什么
事件驱动：Netty是异步事件渠道框架。所有的I/O操作（连接/读/写）都是异步的，结果同事件或者Future通知。
```
// 传统同步（阻塞）：
Socket socket = server.accept();   // 阻塞等连接
InputStream in = socket.getInputStream();
in.read();                         // 阻塞等数据

// Netty 异步（非阻塞）：
ChannelFuture future = bootstrap.connect(host, port);  // 立即返回
future.addListener(f -> {          // 完成后回调
    if (f.isSuccess()) { /* 连接成功 */ }
});
// 发起后立刻返回，不阻塞
```
异步的价值
```
// ① 不阻塞线程（高并发）
// ② 一个线程管理多个连接（EventLoop + Selector）
// ③ 结果通过回调/Future 通知
```
### ChannelFuture
```
// ChannelFuture：Netty 异步操作的结果
// 任何 I/O 操作都返回 ChannelFuture

ChannelFuture future = channel.writeAndFlush(data);
// future 代表"这个写操作的状态"（未完成/成功/失败）
```
两种获取结果的方式
```
// ① 同步等待（阻塞，少用）
ChannelFuture future = channel.writeAndFlush(data);
future.sync();                    // 阻塞直到完成
if (future.isSuccess()) { ... }   // 判断结果

// ② 异步监听（推荐，不阻塞）
ChannelFuture future = channel.writeAndFlush(data);
future.addListener((ChannelFutureListener) f -> {
    if (f.isSuccess()) {
        // 成功回调
    } else {
        // 失败回调
        f.cause().printStackTrace();
    }
});
```
状态判断
```
future.isDone();       // 是否完成
future.isSuccess();    // 是否成功
future.isCancelled();  // 是否取消
future.cause();        // 失败原因
future.getNow();       // 立即获取结果（未完成返回 null）
```
### 异步回调实践
连接
```
// 客户端连接（异步）
ChannelFuture future = bootstrap.connect(host, port);

future.addListener((ChannelFutureListener) f -> {
    if (f.isSuccess()) {
        System.out.println("连接成功");
    } else {
        System.out.println("连接失败");
        // 重连
        f.channel().eventLoop().schedule(this::reconnect, 5, TimeUnit.SECONDS);
    }
});
```
服务端启动
```
// 服务端绑定端口
ChannelFuture future = bootstrap.bind(port).sync();
// 或异步：
bootstrap.bind(port).addListener((ChannelFutureListener) f -> {
    if (f.isSuccess()) {
        System.out.println("服务端启动成功，端口：" + port);
    }
});

// 等待关闭（阻塞）
future.channel().closeFuture().sync();
```
写数据
```
// 写数据（异步）
channel.writeAndFlush(data).addListener((ChannelFutureListener) f -> {
    if (!f.isSuccess()) {
        log.error("写失败", f.cause());
    }
});
// 写数据是异步的，避免阻塞 I/O 线程
```
### Future vs Promise
```
// Netty 的 Future 体系（比 JDK Future 强）

// ① Future（只读）：异步结果
public interface Future<V> extends java.util.concurrent.Future<V> {
    boolean isSuccess();
    Throwable cause();
    Future<V> addListener(GenericFutureListener<?> listener);
    Future<V> sync() throws InterruptedException;   // 同步等待
    // ...
}
// 只能"读"结果、加监听器

// ② Promise（可写）：可主动设置结果
public interface Promise<V> extends Future<V> {
    Promise<V> setSuccess(V result);   // 主动设置成功
    Promise<V> setFailure(Throwable cause);  // 主动设置失败
    boolean trySuccess(V result);
    boolean tryFailure(Throwable cause);
}
// 能"写"结果

// 关系：
// Promise 是"可写的 Future"
// ChannelFuture extends Future（只读）
// ChannelPromise extends ChannelFuture, Promise（可写）
```
为什么需要Promise
```
// 场景：自己实现异步操作
// 创建一个 Promise → 异步处理 → 完成后 setSuccess/setFailure

public Promise<String> doAsync() {
    EventLoop eventLoop = channel.eventLoop();
    Promise<String> promise = eventLoop.newPromise();

    // 异步执行
    eventLoop.execute(() -> {
        try {
            String result = compute();
            promise.setSuccess(result);   // 设置成功
        } catch (Exception e) {
            promise.setFailure(e);        // 设置失败
        }
    });

    return promise;   // 立即返回
}
```
### 异步的注意点
①不要在I/O线程做耗时操作
```
// ❌ 错误：Handler 里阻塞 I/O 线程
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    Thread.sleep(5000);        // 阻塞 I/O 线程！
    orderService.createOrder(); // 慢操作
}
// 后果：I/O 线程被占用 → 其他 Channel 的事件延迟

// ✅ 正确：耗时操作放业务线程池
// 方式一：业务线程池
EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(16);
pipeline.addLast(businessGroup, new MyBusinessHandler());

// 方式二：异步提交
ctx.executor().execute(() -> { /* 快操作 */ });
```
②忽略ChannelFuture的结果
```
// ❌ 忽略结果
channel.writeAndFlush(data);   // 失败也不知道

// ✅ 加监听器
channel.writeAndFlush(data).addListener(f -> {
    if (!f.isSuccess()) log.error("写失败", f.cause());
});
```
③sync()死锁
```
// ❌ 在 I/O 线程里 sync()
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // I/O 线程
    ctx.writeAndFlush(msg).sync();  // ❌ 死锁！等自己完成
}
// 因为 sync 等 Future 完成，但 Future 需要 I/O 线程执行，而 I/O 线程被 sync 阻塞

// ✅ 用 addListener（异步）
ctx.writeAndFlush(msg).addListener(...);
```
### 面试高频题
#### 题目1：Netty是异步还是同步
```
// 异步事件驱动
// I/O 操作异步（Future/回调）
```
#### 题目2：ChannelFuture是什么
```
// 异步操作的结果
// addListener 回调 / sync 等待
```
#### 题目3：Future vs Promise
```
// Future：只读结果
// Promise：可设置结果（可写）
```
#### 题目4：为什么不用sync
```
// I/O 线程里 sync 会死锁
// 推荐 addListener（异步）
```
#### 题目5：耗时操作怎么办
```
// 放业务线程池（DefaultEventExecutorGroup）
// 不阻塞 I/O 线程
```
#### 题目6：写数据失败怎么感知
```
// addListener 监听
// f.isSuccess() / f.cause()
```
