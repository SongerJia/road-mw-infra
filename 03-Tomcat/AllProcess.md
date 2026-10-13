## 请求处理全流程
全流程总览->阶段分解->CoyoteAdapter桥接->对象转换->线程流转->面试高频题
### 全流程总览
一个HTTP请求从进来到响应回去，要经过网络接收、协议解析、容器路由、Filter、Servlet、响应编码和网络发送。
```
客户端
    ↓ ① TCP 连接 + HTTP 字节流
Connector
    ├─ Endpoint（Socket 接收 → 读出字节）
    ├─ Processor（解析 HTTP → 生成 Coyote Request/Response）
    └─ ProtocolHandler（交给 CoyoteAdapter）
    ↓ ② 桥接
CoyoteAdapter（Coyote ↔ Catalina 适配）
    ↓ ③ 转成 Servlet 规范对象
Engine → Host → Context → Wrapper（Mapper 路由）
    ↓ ④ Filter 链
Filter.doFilter()（前置）
    ↓ ⑤ 执行业务
Servlet.service()
    ↓ ⑥ Filter 链
Filter.doFilter()（后置）
    ↓ ⑦ 逐层返回
Connector
    ├─ Processor（编码 HTTP 响应）
    └─ Endpoint（写回 Socket）
    ↓ ⑧ HTTP 响应
客户端
```
### 阶段分解
阶段一：网络接收 EndPoint
```
// ① Acceptor 线程：accept() 新连接
// ② 注册到 Poller
// ③ Poller 检测到可读 → 交给 Worker 线程
// ④ 从 Socket 读出字节

// 关键类：NioEndpoint
```
阶段二：协议解析 Processor
```
// Processor 解析 HTTP 报文：
// ① 请求行：GET /app/user HTTP/1.1
// ② 请求头：Host、User-Agent、Cookie...
// ③ 请求体：POST 数据
// 
// 生成 Coyote 层的 Request/Response
// 关键类：Http11Processor
```
阶段三：桥接 CoyoteAdapter
```
// CoyoteAdapter：连接 Coyote（Connector 层）和 Catalina（Container 层）

public class CoyoteAdapter implements Adapter {
    @Override
    public void service(org.apache.coyote.Request req, 
                        org.apache.coyote.Response res) {
        // ① 获取 Catalina 层的 Request/Response（从池里拿）
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        if (request == null) {
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            req.setNote(ADAPTER_NOTES, request);
        }

        // ② 映射路由（Mapper：URL → Host/Context/Wrapper）
        connector.getService().getMapper().map(...);

        // ③ 交给 Engine 处理
        connector.getService().getContainer().getPipeline()
                 .getFirst().invoke(request, response);
    }
}

// 为什么需要 CoyoteAdapter？
// Cobote 的 Request 是高效率的"轻量对象"（非 Servlet 规范）
// Catalina 的 Request 是标准 HttpServletRequest
// 两者需要适配
```
阶段四：容器路由 Mapper+四层Value
```
// ① Mapper.map()：URL → Host/Context/Wrapper
// ② Engine.Pipeline → StandardEngineValve → Host
// ③ Host.Pipeline → StandardHostValve → Context
// ④ Context.Pipeline → StandardContextValve → Wrapper
// ⑤ Wrapper.Pipeline → StandardWrapperValve → 创建 FilterChain
```
阶段五：Filter链
```
// StandardWrapperValve 创建 ApplicationFilterChain
// 依次执行 Filter
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    // 前置处理
    chain.doFilter(req, res);   // 传给下个 Filter 或 Servlet
    // 后置处理
}
```
阶段六：Servlet执行
```
// 最终调用 Servlet
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // 业务处理
        resp.getWriter().write("hello");
    }
}
```
阶段七和八：响应返回
```
// ① Servlet 写出响应 → Response Buffer
// ② Filter 后置处理
// ③ 逐层 Valve 返回
// ④ Processor 编码成 HTTP 响应
// ⑤ Endpoint 写回 Socket
// ⑥ 客户端收到
```
### CoyoteAdapter桥接
```
Coyote 世界（Connector 层）
    ├─ org.apache.coyote.Request（轻量，字节级）
    └─ org.apache.coyote.Response
    ↓
CoyoteAdapter（适配器：桥接）
    ↓
Catalina 世界（Container 层）
    ├─ org.apache.catalina.connector.Request（Servlet 规范）
    └─ org.apache.catalina.connector.Response
```
转换关系
```
// Coyote Request → Catalina Request
// 通过 setCoyoteRequest() 关联
request.setCoyoteRequest(coyoteRequest);

// Catalina Request 实现 HttpServletRequest
// 读取参数时，实际委托给 Coyote Request（懒解析）
public String getParameter(String name) {
    // 解析 query string / body，缓存结果
}
```
这样设计的目的
```
// ① Coyote Request 轻量、高效（面向性能）
// ② Catalina Request 标准、完整（面向规范）
// ③ 适配器隔离两层，可独立演进
// ④ 复用对象（从对象池取，减少 GC）
```
### 对象池
```
// Tomcat 大量复用对象，避免频繁创建：
// ① Request/Response 对象池
// ② Coyote Request/Response 对象池

// 请求结束 → 回收对象（reset）→ 放回池中
// 下次请求 → 从池里取

// 好处：
// ① 减少 GC
// ② 减少对象创建开销
// ③ 高并发下性能稳定
```
### 线程流转
```
// 整个流程涉及两类线程：

// ① Tomcat 的线程（Connector 的 Worker）
//    - 接收 socket、解析协议
//    - 调用到 Servlet

// ② Servlet 里的业务代码
//    - 默认在 Tomcat Worker 线程执行
//    - 如果异步（AsyncContext），可切到业务线程

// 默认模型：
// Worker 线程：解析 → 路由 → Filter → Servlet → 响应编码
// 全流程在一个 Worker 线程内完成（同步阻塞）
```
异步Servlet
```
// 同步：Worker 线程全程占用（Servlet 耗时则阻塞）
// 异步：Servlet 启动异步 → 释放 Worker 线程 → 业务线程处理 → 完成再回写

// 配置：
@WebServlet(asyncSupported = true)
AsyncContext ctx = request.startAsync();
```
### 面试高频题
#### 题目1：Filter和Value执行的顺序
```
// Valve（容器级）先于 Filter（应用级）
// Engine→Host→Context→Wrapper(BasicValve) → FilterChain → Servlet
```
