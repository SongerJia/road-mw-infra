## 整体架构
Tomcat是什么->Server->Service->Connector+Container->组件关系->server.xml->面试高频题
### Tomcat是什么
Tomcat是Servlet容器+Web服务器。核心职责：接收HTTP请求，交给Servlet处理，返回响应。
```
// 两个身份：
// ① Web 服务器：处理 HTTP（像 Nginx/Apache）
// ② Servlet 容器：管理 Servlet 生命周期、执行 Servlet

// 核心抽象（顶层）：
// 最外层 Server
//   └─ Service（连接器 + 容器）
//        ├─ Connector（接收请求）
//        └─ Container（处理请求）
```
### Server 最顶层
```
// Server：代表"整个 Tomcat 实例"
// 职责：① 管理所有 Service；② 提供启动/关闭入口

// Lifecycle 接口的实现（Tomcat 所有组件都实现它）
public interface Server extends Lifecycle {
    void addService(Service service);      // 添加 Service
    Service findService(String name);      // 查找 Service
    Service[] findServices();              // 所有 Service
    void removeService(Service service);   // 移除 Service
```
```
<!-- server.xml 里的 Server -->
<Server port="8005" shutdown="SHUTDOWN">
    <!-- port：关闭端口（发 SHUTDOWN 字符串可关闭 Tomcat） -->
    <!-- shutdown：关闭命令字符串 -->
</Server>
```
关键点
```
// ① 一个 JVM 里通常只有一个 Server 实例
// ② Server 通过标准 ServerSocket 监听 8005 端口，发 "SHUTDOWN" 即关闭
// ③ 是 Lifecycle 的顶级容器
```
### Service 中间层
```
// Service：把"连接器"和"容器"组合起来！
// 职责：① 管理多个 Connector；② 绑定一个 Engine

public interface Service extends Lifecycle {
    void addConnector(Connector connector);
    Connector[] findConnectors();
    void removeConnector(Connector connector);
    void setContainer(Container container);   // 设置 Engine
    Container getContainer();
}
```
核心设计
```
// 一个 Service = 多个 Connector + 一个 Engine
//
// Server
//   ├─ Service-A
//   │    ├─ Connector-1（HTTP 8080）
//   │    ├─ Connector-2（AJP 8009）
//   │    └─ Engine（一个）
//   └─ Service-B
//        ├─ Connector-3
//        └─ Engine
//
// 为什么一个 Service 只有一个 Engine？
// → 因为一个 Engine 代表"处理请求的完整逻辑"
// → 多个 Connector 的目的是"多种接入方式"，但处理逻辑共享一个 Engine
```
```
<Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"/>
    <Connector port="8009" protocol="AJP/1.3"/>
    <Engine name="Catalina" defaultHost="localhost">
        ...
    </Engine>
</Service>
```
### Connector+Container
Connector 连接器
```
// 职责：对外，负责"通信"
// ① 接收 TCP 连接
// ② 解析应用层协议（HTTP/AJP）
// ③ 把请求封装成 Request/Response 给 Container
// ④ 把处理结果转成 HTTP 响应发回

// 三个子组件：
// Endpoint：底层网络（Socket/IO）
// Processor：协议解析（HTTP/AJP）
// ProtocolHandler：组合 + 管理上述两者
```
Container 容器
```
// 职责：对内，负责"处理"
// ① 接收 Connector 传来的请求
// ② 逐层路由：Engine → Host → Context → Wrapper
// ③ 最终调用 Servlet

// 四层容器（层层嵌套）：
// Engine   → 引擎（一个 Service 一个）
// Host     → 虚拟主机（对应域名）
// Context  → Web 应用（对应一个 WAR/目录）
// Wrapper  → 单个 Servlet 的包装
```
两者关系
```
请求方向：Connector → Container
响应方向：Container → Connector

Connector（对外通信）
    ↓ 封装成 Request
Container（对内处理）
    ↓ 逐层路由
Servlet
    ↓ 处理结果
Container
    ↓ 交给 Connector
Connector（编码成 HTTP 响应发回）
```
### 组件关系全景
```
Server                          ← 整个 Tomcat 实例
│
├─ Service 1                    ← 一组"连接器 + 容器"
│   ├─ Connector (HTTP:8080)    ← 对外通信
│   ├─ Connector (AJP:8009)     ← 多种接入
│   └─ Engine                   ← 唯一的处理引擎
│       └─ Host (localhost)     ← 虚拟主机
│           └─ Context (/app)   ← Web 应用
│               └─ Wrapper      ← Servlet 包装
│
└─ Service 2                    ← 可以有多组
    ├─ Connector
    └─ Engine
        └─ ...
```
### server.xml结构
```
<!-- Tomcat 的核心配置文件 -->
<Server port="8005" shutdown="SHUTDOWN">
    <!-- Listener：监听生命周期事件 -->
    <Listener className="org.apache.catalina.startup.VersionLoggerListener"/>

    <!-- Service：连接器 + 引擎 -->
    <Service name="Catalina">
        <!-- Connector：对外连接 -->
        <Connector port="8080" protocol="HTTP/1.1"
                   connectionTimeout="20000"
                   redirectPort="8443"/>

        <!-- Engine：处理引擎 -->
        <Engine name="Catalina" defaultHost="localhost">
            <!-- Host：虚拟主机 -->
            <Host name="localhost" appBase="webapps">
                <!-- Context：Web 应用 -->
                <Context path="/app" docBase="app"/>
            </Host>
        </Engine>
    </Service>
</Server>
```
### 面试高频题
#### 题目1：Tomcat顶层架构
```
// Server → Service → Connector + Container
```
#### 题目2：server/service/connector/container关系
```
// Server 管 Service
// Service = 多 Connector + 一 Engine
// Connector 通信，Container 处理
```
#### 题目3：为什么一个service只能有一个Engine
```
// Engine 是处理逻辑的唯一入口
// 多个 Connector 共享同一处理逻辑
```
#### 题目4：Connector的作用
```
// 接收连接 + 协议解析 + 封装 Request
```
#### 题目5：Container的四层结构
```
// Engine → Host → Context → Wrapper
```
#### 题目6：Tomcat怎么关闭
```
// 向 8005 端口发 "SHUTDOWN" 字符串
```

