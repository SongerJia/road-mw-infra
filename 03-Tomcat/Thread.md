## 线程模型
线程模式是什么->三大角色->线程池->各IO模型对比->调优参数->面试高频题
### 线程模式是什么
Tomcat的NIO线程模型，少量专用线程负责接收连接和IO事件，大量Worker线程负责业务处理，形成多路复用+线程池结构。
```
// 核心思想：职责分离
// ① 接收连接：Acceptor（1~2 个线程）
// ② IO 事件轮询：Poller（1~2 个线程）
// ③ 业务处理：Worker（线程池，几百个）

// 为什么这样设计？
// 接收/轮询是"轻量高频"操作，少量线程即可
// 业务处理是"重量耗时"操作，需要线程池
```
### 三大角色 NIO模型
①Acceptor 接收线程
```
// 职责：接收新连接
// 数量：1~2 个
// 工作：
//   while (true) {
//       SocketChannel socket = serverSocket.accept();  // 阻塞接收
//       // 注册到 Poller
//       poller.register(socket);
//   }

// 只在"建立连接"时工作，成本低
```
②Poller 轮询线程
```
// 职责：监听所有已连接 socket 的 IO 事件
// 数量：通常 = CPU 核数（或 1~2 个）
// 工作：
//   while (true) {
//       selector.select();                    // 多路复用
//       Set<SelectionKey> keys = ...;
//       for (SelectionKey key : keys) {
//           if (key.isReadable()) {
//               // 交给 Worker 处理
//               executor.execute(new SocketProcessor(key));
//           }
//       }
//   }

// 一个 Poller 管成千上万个连接（IO 多路复用）
```
③Worker 工作线程
```
// 职责：执行业务处理（解析请求 + 调用 Servlet + 编码响应）
// 数量：线程池，默认 200（maxThreads）
// 工作：
//   // 从 Poller 拿到的 SocketProcessor
//   processor.process(socket);   // 解析 + 处理 + 响应

// 这是"重"操作，跑在业务线程池里
```
三者协作
```
        ┌─────────────────────────────────────┐
        │            Acceptor 线程             │
        │   接收新连接 → 注册到 Poller           │
        └──────────────┬──────────────────────┘
                       ↓
        ┌─────────────────────────────────────┐
        │            Poller 线程               │
        │   selector.select() 监听 IO 事件     │
        │   有可读事件 → 投递给 Worker 线程池    │
        └──────────────┬──────────────────────┘
                       ↓ 任务队列
        ┌─────────────────────────────────────┐
        │          Worker 线程池               │
        │   解析 HTTP → Engine → Servlet       │
        │   编码响应 → 写回                     │
        └─────────────────────────────────────┘
```
### 线程池
Worker线程由Executor管理，可自定义。
默认线程池
```
// 不配置 <Executor> 时，Tomcat 内部用默认线程池
// 关键参数：
// maxThreads：最大工作线程数（默认 200）
// minSpareThreads：最小空闲线程（默认 10）
// maxIdleTime：空闲线程存活时间（默认 60s）
// maxQueueSize：任务队列大小
```
自定义线程池
```
<!-- server.xml：定义共享线程池 -->
<Executor name="tomcatThreadPool"
          namePrefix="catalina-exec-"
          maxThreads="500"
          minSpareThreads="50"
          maxIdleTime="60000"
          prestartminSpareThreads="true"/>

<!-- Connector 引用线程池 -->
<Connector port="8080"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           executor="tomcatThreadPool"/>
```
线程池本质
```
// Tomcat 的线程池 = JDK ThreadPoolExecutor 的封装
// ① 任务队列：Poller 提交的任务
// ② 拒绝策略：队列满时拒绝
// ③ 核心思想：复用线程，避免频繁创建

// 具体实现：StandardThreadExecutor（Tomcat 8+）
// 内部用 TaskQueue + ThreadPoolExecutor
```
### 各IO模型的线程模型对比
BIO模型 已淘汰
```
// 一个连接一个线程
// Acceptor 接收 → 直接创建线程处理
// 连接数 = 线程数
// 问题：并发高时线程爆炸
```
NIO模型 默认
```
// 多路复用
// Acceptor（接收）+ Poller（轮询）+ Worker（处理）
// 少量 Poller 管大量连接
```
NIO2模型
```
// 异步 IO（AIO）
// Acceptor + Poller + Worker（类似 NIO）
// 但 IO 是内核完成的（Proactor）
```
APR模型
```
// 本地库（epoll/sendfile）
// Acceptor + Poller + Worker
// 性能最高
```

|模型|接连接|轮询|处理|特点|
|---|---|---|---|---|
|**BIO**|Acceptor|无|每连接 1 线程|线程爆炸|
|**NIO**|Acceptor|Poller|Worker 池|多路复用|
|**NIO2**|Acceptor|Poller|Worker 池|异步 IO|
|**APR**|Acceptor|Poller|Worker 池|本地库|
### 调优参数
Connector关键参数
```
<Connector port="8080"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="500"          <!-- 最大工作线程 -->
           minSpareThreads="50"      <!-- 最小空闲线程 -->
           acceptCount="200"         <!-- 等待队列长度（连接满了排队）-->
           maxConnections="10000"    <!-- 最大连接数 -->
           connectionTimeout="20000" <!-- 连接超时(ms) -->
           maxKeepAliveRequests="100"<!-- 长连接最大请求数 -->
           compression="on"          <!-- 开启压缩 -->
           compressionMinSize="2048" <!-- 压缩阈值 -->
           URIEncoding="UTF-8"/>
```
参数含义

|参数|含义|建议|
|---|---|---|
|**maxThreads**|最大工作线程数|200~500（看 CPU/IO 密集度）|
|**acceptCount**|连接队列长度|100~200（满了拒连）|
|**maxConnections**|最大连接数|NIO 默认 10000|
|**connectionTimeout**|连接超时|20000ms|
|**minSpareThreads**|最小空闲线程|预热，避免冷启动|
调优思路
```
// ① CPU 密集型：maxThreads ≈ CPU 核数 + 1
// ② IO 密集型：maxThreads 大一些（如 500）
// ③ acceptCount 不能太大（会堆积）
// ④ 开启 compression（省带宽，费 CPU）
// ⑤ 用 APR 或 NIO2（追求极致性能）
// ⑥ 共享线程池（多 Connector 复用）
```
常见问题排查
```
// ① 线程数飙高 → 看是不是有阻塞操作
// ② 请求堆积 → acceptCount 太小 / 处理太慢
// ③ 内存泄漏 → WebappClassLoader 未释放
// ④ CPU 打满 → 线程空转 / 大量计算
```
