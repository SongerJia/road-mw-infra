## 集群容错
集群容错是什么->6种策略->AbstractClusterInvoker源码->Failover源码->面试高频题
### 集群容错是什么
集群容错：当调用失败（Provider宕机、网络异常、超时）时，Dubbo进行处理。
```
// 位置：Cluster 集群层
// 职责：
// ① 从 Directory 拿 Provider 列表
// ② 用 LoadBalance 选一台
// ③ 调用，失败则按"容错策略"处理

// 核心接口：
@SPI(FailoverCluster.NAME)
public interface Cluster {
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory);
}
```
```
// 关键：Cluster.join() 返回一个 ClusterInvoker
// 业务调用时，实际调的是 ClusterInvoker（内含容错逻辑）
```
### 6种容错策略
| 策略            | 名称     | 行为                 | 场景         |
| ------------- | ------ | ------------------ | ---------- |
| **Failover**  | 失败自动切换 | 失败重试其他服务器（默认 2 次）  | 读操作        |
| **Failfast**  | 快速失败   | 只调一次，失败立即抛异常       | 非幂等写操作     |
| **Failsafe**  | 失败安全   | 失败忽略，返回空结果         | 日志等无关紧要    |
| **Failback**  | 失败自动恢复 | 失败后台记录，定时重发        | 消息通知       |
| **Forking**   | 并行调用   | 并行调多个，一个成功即返回      | 实时性高、可容忍浪费 |
| **Broadcast** | 广播     | 广播所有 Provider，逐个调用 | 通知所有节点更新缓存 |
①Failover 默认
```
// 失败自动切换，重试其他服务器
// retries=2（默认），总共调用 3 次（1 次 + 2 次重试）

// 注意：
// ① 重试会带来幂等问题（非幂等操作慎用）
// ② 重试要在"不同"的 Provider 上（避免重试到同一台）
```
②Failfast
```
// 只调用一次，失败立即报错
// 适合：非幂等操作（如新增订单）
```
③Failsafe
```
// 失败忽略，记日志，返回空
// 适合：审计日志、埋点等
```
④Failback
```
// 失败后放入失败队列，后台定时重试
// 适合：消息通知、异步补偿
```
⑤Forking
```
// 并行调用 N 个 Provider（forks=2）
// 只要一个成功就返回
// 适合：实时性要求高（读操作）
```
⑥Broadcast
```
// 广播给所有 Provider，逐个调用
// 一个失败则整体失败
// 适合：通知所有节点刷新缓存、更新本地资源
```
### AbstractClusterInvoker源码
```
// org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker
public abstract class AbstractClusterInvoker<T> implements Invoker<T> {

    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        // ① 检查是否被销毁
        checkWhetherDestroyed();

        // ② 从 Directory 拿到所有 Provider（经过 Router 过滤）
        List<Invoker<T>> invokers = list(invocation);

        // ③ 初始化负载均衡（从 URL 的 loadbalance 参数选）
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);

        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        // ④ 调用子类的 doInvoke（不同容错策略不同实现）
        return doInvoke(invocation, invokers, loadbalance);
    }

    protected abstract Result doInvoke(Invocation invocation,
                                       List<Invoker<T>> invokers,
                                       LoadBalance loadbalance) throws RpcException;

    // ⑤ select：选一个 Invoker（用负载均衡，避开已失败的）
    protected Invoker<T> select(LoadBalance loadbalance,
                                Invocation invocation,
                                List<Invoker<T>> invokers,
                                List<Invoker<T>> selected) {
        Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
        // 如果选中的已在 selected（之前失败过），重新选
        if (selected != null && selected.contains(invoker)) {
            invoker = reselect(loadbalance, invocation, invokers, selected, ...);
        }
        return invoker;
    }
}
// 模板方法模式：invoke 定义骨架，doInvoke 交给子类
```
### FailoverClusterInvoker
```
// org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker
public Result doInvoke(Invocation invocation,
                       final List<Invoker<T>> invokers,
                       LoadBalance loadbalance) throws RpcException {

    // ① 拿到重试次数（默认 2）
    int len = getUrl().getMethodParameter(invocation.getMethodName(), "retries", 2) + 1;
    if (len <= 0) len = 1;      // 至少调 1 次

    // ② 记录已调用的 Invoker（避免重试到同一台）
    List<Invoker<T>> invoked = new ArrayList<>(invokers.size());

    RpcException le = null;
    for (int i = 0; i < len; i++) {
        // ③ 第一次调用，或重试时（i>0）
        if (i > 0) {
            // 重试前重新检查 + 从淘汰的 Provider 中排除已失败的
            checkWhetherDestroyed();
            copyInvokers = list(invocation);
            // 重新进行负载均衡
        }

        // ④ 选一个 Invoker（避开已失败的）
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        invoked.add(invoker);

        try {
            // ⑤ 调用
            Result result = invoker.invoke(invocation);
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) throw e;      // 业务异常直接抛
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        }
    }
    // ⑥ 全部失败 → 抛异常
    throw new RpcException(...);
}
```
```
// 关键点：
// ① 默认 retries=2 → 总共 3 次调用
// ② 用 invoked 记录已调用的，避免重复
// ③ 每次重试都重新 list + 负载均衡
// ④ 业务异常（isBiz）不重试，直接抛
//    → 幂等保障：只有"系统异常"才重试
```
### 各策略源码对比
```
// FailfastClusterInvoker
public Result doInvoke(...) {
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        throw new RpcException("Failfast invoke providers ... ");
        // 只调一次，失败直接抛
    }
}

// FailsafeClusterInvoker
public Result doInvoke(...) {
    try {
        Invoker<T> invoker = select(...);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failsafe ignore exception: " + e.getMessage(), e);
        return new RpcResult();     // 失败返回空结果
    }
}

// FailbackClusterInvoker
// 失败 → 放入 ConcurrentMap 失败队列 → 定时任务重发

// ForkingClusterInvoker
// 并发调用 N 个（forks），用 CountDownLatch 等，一个成功返回

// BroadcastClusterInvoker
// 循环所有 invoker 调用，一个失败记异常，最后统一抛
```
### 配置与使用
```
<!-- Failover（默认）-->
<dubbo:reference cluster="failover" retries="2"/>

<!-- Failfast -->
<dubbo:reference cluster="failfast"/>

<!-- Forking -->
<dubbo:reference cluster="forking" forks="2"/>
```
```
// 注解
@Reference(cluster = "failover", retries = 2)
private UserService userService;
```
### 面试高频题
#### 题目1：Failover怎么避免重试同一台
```
// 用 invoked 列表记录已调用的，select 时避开
```
#### 题目2：业务异常会重试吗
```
// 不会！isBiz() 判断，业务异常直接抛
// 只有系统异常才重试
```