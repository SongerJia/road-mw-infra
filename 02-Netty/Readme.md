> **模块边界**：TCP/IP 协议栈本体 → 网络基础模块；Java NIO 的 API 细节 → 基础模块（此处只讲 Netty 的封装）；Dubbo 的 RPC 协议与治理 → 框架模块。 

**① Netty 总览**

1. 是什么、定位（网络传输底座）
2. 为什么用 Netty、与 Dubbo / gRPC 的关系
3. 应用场景（RPC / 网关 / IM / 游戏 / 中间件底层）
4. 与 NIO / BIO / AIO 对比
5. 版本演进（Netty 5 废弃，主线 4.1 / 4.2）

**② IO 模型与 NIO 基础** 6. BIO / NIO / AIO 7. IO 多路复用（select / poll / epoll） 8. Java NIO 三件套（Channel / Buffer / Selector） 9. 原生 NIO 的痛点、epoll 空轮询 bug 与规避 10. 零拷贝基础

**③ 核心组件** 11. Channel 与生命周期、closeFuture 12. ChannelHandler / ChannelPipeline 13. ChannelHandlerContext 14. ChannelFuture / Promise 15. Bootstrap / ServerBootstrap 与启动流程 16. ChannelOption 全景（SO_BACKLOG / TCP_NODELAY / SO_REUSEADDR / SO_KEEPALIVE）

**④ 线程模型** 17. Reactor 模式（单线程 / 多线程 / 主从） 18. Netty 的线程模型（bossGroup / workerGroup）与 EventLoopGroup 线程数配置（2 × CPU） 19. EventLoop / NioEventLoop 的 run 循环（IO + 任务队列 + 定时任务） 20. EventLoop 与 Channel 绑定（一 Channel 一 EventLoop，一 EventLoop 多 Channel） 21. 任务队列、MpscQueue 与异步 22. 同一 Channel 的 Handler 单线程执行与线程安全含义 23. 业务线程池的引入（避免阻塞 IO 线程）

**⑤ Pipeline 与 Handler** 24. ChannelPipeline 责任链 25. Inbound / Outbound Handler 26. Handler 传播方向与添加顺序 27. @Sharable 与线程安全 28. 编解码器层次（ByteToMessageDecoder vs MessageToMessageCodec） 29. ReplayingDecoder 与半包处理 30. pipeline 的热插拔

**⑥ ByteBuf** 31. 与 ByteBuffer 的区别 32. 读写指针（readerIndex / writerIndex） 33. 三种类型（Heap / Direct / Composite） 34. 池化（PooledByteBufAllocator） 35. 引用计数与内存泄漏（ResourceLeakDetector）、释放时机

**⑦ 编解码与粘包** 36. 粘包拆包（TCP 流式本质） 37. 解决方式（定长 / 分隔符 / 长度字段） 38. LengthFieldBasedFrameDecoder 39. 序列化（protobuf / JSON / JDK）

**⑧ 高性能机制** 40. 零拷贝三种实现（切片 / CompositeByteBuf / transferTo·FileRegion） 41. 内存池（PoolArena） 42. 无锁化设计 43. FastThreadLocal 44. Native Transport（Epoll / KQueue）

**⑨ 可靠性与安全** 45. 心跳检测（IdleStateHandler） 46. 断线重连 47. 优雅关闭 48. 流量整形（WriteBufferWaterMark） 49. SSL/TLS（SslHandler 的位置）

**⑩ TCP、协议与调优** 50. TCP 参数详解与调优取舍 51. 与 HTTP 集成 52. WebSocket 协议升级握手 53. 自定义协议 54. Write / flush 与 ChannelOutboundBuffer 55. Netty 为什么快（收口：线程模型 + 零拷贝 + 内存池 + 无锁）与常见坑清单（业务阻塞 IO 线程 / ByteBuf 泄漏 / @Sharable 误用 / 粘包未处理 / 水位线未设 / 心跳缺失）