## 服务治理
服务治理是什么->路由规则->动态配置->限流降级->超时重试->Mock->分组版本->面试高频题
### 服务治理是什么
服务治理：运行时管理服务的能力，不改代码，动态调整服务的行为。
```
// 实现机制：
// ① 配置存 ZK 的 routers / configurators 节点（URL 形式）
// ② Registry 推送给 Consumer/Provider
// ③ Dubbo 内部组件（Router/Filter）实时生效
// → 配置驱动，动态生效
```
### 路由规则 Router
条件路由
```
# 规则示例（存 ZK 的 routers 节点）：
# ① 排除：消费方 A 不调用这个服务
scope: service
force: true
conditions:
  - host != 10.20.1.1 =>        # 禁止 10.20.1.1 访问

# ② 分流：某接口的流量只走指定机器
  - host = 10.20.* => host = 10.20.2.*

# ③ 读写分离
  - method = find,list,get => host = 10.20.1.*
  - method != find,list,get => host = 10.20.3.*
```
```
// 语法：
// 消费方 => 提供方
// 左边：匹配 Consumer；右边：匹配 Provider
// =  匹配；!= 不匹配；* 通配
```
标签路由 3.x常用
```
// 给 Provider 打标签（如灰度机器 tag=gray）
<dubbo:provider tag="gray"/>

// Consumer 带标签调用 → 优先调用同标签的 Provider
// → 灰度发布！
RpcContext.getContext().setAttachment("dubbo.tag", "gray");
userService.getUser(1L);   // 请求打到 gray 机器
```
```
// 源码：Router 接口 + Chain 责任链
public interface Router extends Comparable<Router> {
    URL getUrl();
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation);
}
// ClusterInvoker 调用前，先过 Router 链筛选 Invoker
```
### 动态配置 Override
```
<!-- 修改超时（不改代码，动态生效） -->
override://com.example.UserService?timeout=5000

<!-- 调整权重 -->
override://192.168.1.10:20880?weight=200

<!-- 下线某台 -->
override://192.168.1.10:20880?disabled=true
```
```
// 原理：
// ① 管理端改配置 → 写 ZK configurators 节点
// ② ZK 通知 Provider/Consumer
// ③ Dubbo 收到 override URL → 合并到现有配置
// → 动态生效（不重启）
```
### 限流
①Provider端限流 executes
```
<!-- 服务端并发限流：同一时间最多 10 个线程处理 -->
<dubbo:service executes="10"/>
<!-- 方法级 -->
<dubbo:method name="getUser" executes="5"/>
```
②Consumer端限流 actives
```
<!-- 客户端并发限流：最多同时发起 5 个并发请求 -->
<dubbo:reference actives="5"/>
```
```
// 源码：ExecuteLimitFilter（Provider 端）
public Result invoke(Invoker<?> invoker, Invocation invocation) {
    int max = invoker.getUrl().getMethodParameter(method, "executes", 0);
    if (max > 0) {
        // RpcStatus 记录当前并发数
        if (active >= max) {
            throw new RpcException("超过最大并发数");   // 快速失败
        }
    }
    return invoker.invoke(invocation);
}
```
```
// 等价于信号量限流（Semaphore）
// 超了就抛 RpcException（不是排队等待）
```
### 服务降级 Mock
```
<!-- force：强制 Mock（直接返回假数据，不走远程） -->
<dubbo:reference mock="force:return null"/>

<!-- fail：失败后 Mock（调用失败才返回假数据） -->
<dubbo:reference mock="fail:return null"/>

<!-- 自定义 Mock 类 -->
<dubbo:reference mock="com.example.UserServiceMock"/>
```
```
// 自定义 Mock（实现接口）
public class UserServiceMock implements UserService {
    public User getUser(Long id) {
        return new User(id, "默认用户");   // 降级数据
    }
}
```
```
// 常见用法：
// ① 服务启动检查关闭 + fail Mock → 消费端启动不依赖生产端
// ② Provider 挂了 → Consumer 返回兜底数据 → 系统可用
```
### 超时与重试
超时 timeout
```
// 优先级：方法级 > 接口级 > 全局
// Consumer > Provider（两边都配时，Consumer 优先）

<dubbo:method name="getUser" timeout="3000"/>
```
重试 retries
```
// 只对 Failover 集群有效（前面讲过）
<dubbo:method name="queryUser" retries="2"/>
<dubbo:method name="createOrder" retries="0"/>   // 写操作不重试！
```
```
// 超时 + 重试的组合问题：
// ① 超时 3s + 重试 2 次 → 最坏 9s
// ② 重试导致重复提交（非幂等）
// → 写操作 retries=0 或用 Failfast
```
### 直连、分组、版本、隐式传参
直连提供者 绕过注册中心
```
<dubbo:reference url="dubbo://192.168.1.10:20880"/>
<!-- 调试时常用，生产不用 -->
```
服务分组 group
```
<!-- 同一接口，多套实现（如不同机房/不同版本） -->
<dubbo:service group="pay_a"/>
<dubbo:reference group="pay_a"/>
```
版本 version
```
<!-- 灰度/升级：新旧版本并存 -->
<dubbo:service version="1.0.0"/>
<dubbo:service version="2.0.0"/>

<dubbo:reference version="1.0.0"/>   <!-- 消费指定版本 -->
```
```
// group 和 version 的区别：
// version：同一服务的版本演进（灰度升级）
// group：多个并行实现（逻辑隔离）
```
隐式传参 RpcContext
```
// 链路传递上下文（如 TraceId、登录态）
RpcContext.getContext().setAttachment("traceId", "abc123");
userService.getUser(1L);      // attachment 随请求传到 Provider

// Provider 端获取
String traceId = RpcContext.getContext().getAttachment("traceId");
```
### 面试高频题
#### 题目1：服务治理都包含什么
```
// 路由、动态配置、限流、降级、超时重试、分组版本
```
#### 题目2：路由规则怎么实现动态生效
```
// 规则存 ZK routers 节点 → 推送 → Router 链筛选 Invoker
```
#### 题目3：怎么实现灰度发布
```
// 标签路由（dubbo.tag）
```
#### 题目4：限流怎么实现
```
// Provider: executes；Consumer: actives
// 信号量方式，超了抛异常
```
#### 题目5：Mock降级有哪几种
```
// force（强制）、fail（失败后）、自定义 Mock 类
```
#### 题目6：超时和重试怎么配
```
// 方法级 > 接口级 > 全局；Consumer > Provider
// 写操作 retries=0（幂等）
```
#### 题目7：group和version区别
```
// version：版本演进；group：并行实现
```


