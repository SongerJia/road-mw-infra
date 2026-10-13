## 容器
Container是什么->四层容器->Pipeline+value->Mapper路由->请求流转->面试高频题
### Container是什么
Container：Tomcat对内的处理核心。接收Connector传来的请求，逐层路由到具体的Servlet，执行并返回结果。
```
// 位置：Service 里（与 Connector 并列）
// 职责：处理请求（业务侧）
// 核心接口：
public interface Container extends Lifecycle {
    void addChild(Container child);       // 添加子容器
    Container findChild(String name);     // 查找子容器
    Container[] findChildren();
    void removeChild(Container child);
    Pipeline getPipeline();               // 获取责任链
    void invoke(Request request, Response response);  // 处理请求
}
```
### 四层容器
Container是接口，有四个实现类，层层嵌套，各司其职。

|层级|实现类|职责|对应关系|
|---|---|---|---|
|**Engine**|StandardEngine|引擎，处理请求入口|一个 Service 一个|
|**Host**|StandardHost|虚拟主机|一个域名|
|**Context**|StandardContext|Web 应用|一个 WAR/目录|
|**Wrapper**|StandardWrapper|Servlet 包装|一个 Servlet|
①Engine引擎
```
// 一个 Service 只有一个 Engine
// 职责：
// ① 接收 Connector 传来的请求
// ② 根据 Host 名找到对应虚拟主机
// ③ 转发给 Host

// 配置：
<Engine name="Catalina" defaultHost="localhost">
```
②Host 虚拟主机
```
// 代表一个"虚拟主机"（域名）
// 职责：
// ① 根据域名匹配（如 localhost、www.example.com）
// ② 找到对应的 Context（Web 应用）

// 配置：
<Host name="localhost" appBase="webapps">
    <Context path="/app" docBase="app"/>
</Host>
```
③Context web应用
```
// 代表一个 Web 应用（一个 WAR 包 / 目录）
// 职责：
// ① 管理 Servlet、Filter、Listener
// ② 根据 URL 路径找到对应 Wrapper
// ③ 管理应用的类加载器

// 配置：
<Context path="/app" docBase="app"/>
// path=/app → 访问 http://localhost:8080/app/...
```
④Wrapper servlet包装
```
// 最底层，包装单个 Servlet
// 职责：
// ① 管理 Servlet 生命周期（init/service/destroy）
// ② 调用 Servlet 的 service() 方法
// ③ 管理 Servlet 的单例

// 每个 <servlet> 对应一个 Wrapper
```
嵌套关系
```
Engine（一个 Service 一个）
    ↓ 按 Host 名
Host（虚拟主机，一个域名）
    ↓ 按 Context 路径
Context（Web 应用）
    ↓ 按 Servlet 映射
Wrapper（Servlet）
    ↓ 调用
Servlet.service()
```
### Pipeline+value
每个容器都有一个Pipeline，里面装着一串Value。请求像水流一样，逐个穿过Value，最终到达基础阀。
结构
```
Container
    └─ Pipeline（管道）
           ├─ Valve-1（自定义，可选）
           ├─ Valve-2（自定义，可选）
           └─ BasicValve（基础阀，必有一个，负责处理）
```
各层的基础阀

|容器|基础阀（BasicValve）|作用|
|---|---|---|
|**Engine**|StandardEngineValve|转发给 Host|
|**Host**|StandardHostValve|转发给 Context|
|**Context**|StandardContextValve|转发给 Wrapper|
|**Wrapper**|StandardWrapperValve|执行 Servlet（过滤链 + service）|
责任链执行
```
// Valve 接口
public interface Valve {
    void invoke(Request request, Response response);

    // Valve 之间形成链，调用 next 继续
}
```
```
// 请求穿越 Valve 链
Engine.Pipeline
    → Valve1.invoke() → getNext().invoke() → ...BasicValve
        → 转发到 Host.Pipeline
            → ...BasicValve → 转发到 Context.Pipeline
                → ...BasicValve → 转发到 Wrapper.Pipeline
                    → StandardWrapperValve → 执行 Filter 链 → Servlet
```
自定义Value 拦截
```
// 类似 Filter，但作用在容器级别（更底层）
public class MyValve extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException {
        // 前置处理
        System.out.println("valve before");
        // 继续下个 valve
        getNext().invoke(request, response);
        // 后置处理
        System.out.println("valve after");
    }
}

// 注册到 Context
context.getPipeline().addValve(new MyValve());
```
Value vs Filter

|维度|Valve|Filter|
|---|---|---|
|**所属**|Tomcat（容器级）|Servlet 规范（应用级）|
|**可移植性**|只能 Tomcat|所有 Web 容器|
|**层面**|容器内部|Servlet 前后|
|**作用**|日志/访问控制|业务过滤|
### Mapper 请求路由
Mapper负责请求URL到具体处理类的映射。
映射层级
```
// 请求：http://localhost:8080/app/user?id=1
// 
// ① 域名 localhost → 找 Host
// ② 路径 /app     → 找 Context
// ③ 路径 /user    → 找 Wrapper（Servlet）
```
Mapper规则
```
// ① Host 映射：按域名（含泛域名 *.example.com）
// ② Context 映射：按路径前缀（最长匹配）
// ③ Wrapper 映射：三种规则
//    - 精确匹配：/user
//    - 前缀匹配：/user/*
//    - 扩展名匹配：*.jsp
//    - 默认：/（DefaultServlet）

// Mapper 内部用 Map 缓存映射结果，提升性能
```
### 请求流转全景
```
Connector 收到请求
    ↓ CoyoteAdapter（适配器：Coyote ↔ Catalina）
Engine
    ↓ Engine.Pipeline → StandardEngineValve
Host（按域名匹配）
    ↓ Host.Pipeline → StandardHostValve
Context（按路径匹配）
    ↓ Context.Pipeline → StandardContextValve
Wrapper（按 Servlet 映射）
    ↓ Wrapper.Pipeline → StandardWrapperValve
    ↓ 创建 FilterChain
Filter 链（Filter.doFilter 前后）
    ↓
Servlet.service()
    ↓ 处理完
逐层返回（Valve 后置处理）
    ↓
Connector 编码响应
    ↓
返回客户端
```

