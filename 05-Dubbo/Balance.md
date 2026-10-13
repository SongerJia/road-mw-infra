## 负载均衡
负载均衡是什么->4种策略->源码实现->服务预热->面试高频题
### Dubbo负载均衡是什么
```
// 位置：Cluster 集群层内部
// 作用：从 Directory 提供的 Provider 列表中，选一台来调用

// 核心接口：
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation);
}
// @SPI(RandomLoadBalance.NAME) → 默认"随机"
// @Adaptive → 可用 URL 参数 loadbalance 指定实现
```
### 4种策略
①Random 随机 默认
```
// 加权随机
// ① 无权重：随机选一个
// ② 有权重：按权重大小决定概率
// 适合：后端性能相近，简单高效
```
②RoundRobin（轮询）
```
// 加权轮询（平滑加权轮询算法）
// ① 无权重：轮流选
// ② 有权重：按权重比例轮询（平滑，避免连续打同一台）
// 适合：后端性能相近
```
③LeastActive 最少活跃调用数
```
// 选"当前活跃调用数最少"的 Provider
// 活跃数 = 正在处理的请求数
// ① 活跃数相同 → 随机
// ② 让处理快的机器接更多请求（自适应）
// 适合：请求耗时差异大
```
④ConsistentHash 一致性hash
```
// 相同参数的请求 → 固定打到同一台
// ① 默认按第一个参数 hash
// ② 可用 hash.arguments 指定参数
// ③ 用 160 个虚拟节点（减少数据倾斜）
// 适合：有状态服务（如缓存）
```
### 源码实现
抽象基类 AbstractLoadBalance
```
// org.apache.dubbo.rpc.cluster.loadbalance.AbstractLoadBalance
public abstract class AbstractLoadBalance implements LoadBalance {

    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (invokers == null || invokers.isEmpty()) return null;
        if (invokers.size() == 1) return invokers.get(0);   // 只有一个直接返回
        return doSelect(invokers, url, invocation);          // 子类实现
    }

    protected abstract <T> Invoker<T> doSelect(...);

    // 计算权重（含服务预热逻辑）
    protected int getWeight(Invoker<?> invoker, Invocation invocation) {
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), "weight", 100);
        if (weight > 0) {
            long timestamp = invoker.getUrl().getParameter("timestamp", 0);
            if (timestamp > 0) {
                int uptime = (int) (System.currentTimeMillis() - timestamp);
                int warmup = invoker.getUrl().getParameter("warmup", 600000);  // 默认 10 分钟
                if (uptime > 0 && uptime < warmup) {
                    // 服务预热：启动初期降低权重
                    weight = calculateWarmupWeight(uptime, warmup, weight);
                }
            }
        }
        return weight;
    }
}
```
①RandomLoadBalance源码
```
public class RandomLoadBalance extends AbstractLoadBalance {
    public <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();
        boolean sameWeight = true;
        int[] weights = new int[length];
        int totalWeight = 0;

        // ① 累加权重
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight;
            weights[i] = totalWeight;
            if (sameWeight && i > 0 && weight != weights[i-1] - weights[i-2... ]) sameWeight = false;
        }

        // ② 权重相同 → 直接随机
        if (totalWeight > 0 && !sameWeight) {
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // ③ 按权重区间选择
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) return invokers.get(i);
            }
        }
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
}
// 核心：加权随机（累积权重区间 + 随机 offset）
```
②RoundRobinLoadBalance
```
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    // 平滑加权轮询（Nginx 同款算法）
    public <T> Invoker<T> doSelect(...) {
        // ① 每个 Invoker 维护：weight（固定）、current（动态）、
        // ② 每次：current += weight
        // ③ 选 current 最大的
        // ④ 被选的 current -= totalWeight
    }
}
// 效果：权重高的被选次数多，但分布平滑
```
③LeastActiveLoadBalance
```
public class LeastActiveLoadBalance extends AbstractLoadBalance {
    public <T> Invoker<T> doSelect(...) {
        int leastActive = -1;
        int leastCount = 0;
        int[] leastIndexes = new int[length];
        int[] weights = new int[length];

        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            int active = RpcStatus.getStatus(invoker.getUrl(), ...).getActive();  // 活跃数
            int weight = getWeight(invoker, invocation);

            if (leastActive == -1 || active < leastActive) {
                leastActive = active;    // 记录最小活跃数
                leastCount = 1;
                leastIndexes[0] = i;
            } else if (active == leastActive) {
                leastIndexes[leastCount++] = i;   // 活跃数相同，收集
            }
        }
        // ② 只有一个最小 → 直接返回
        // ③ 多个最小 → 按权重随机
    }
}
// 核心：活跃数 = 正在处理的请求数（越快处理完，活跃数越低，越容易被选）
```
④ConsistentHashLoadBalance
```
public class ConsistentHashLoadBalance extends AbstractLoadBalance {
    public <T> Invoker<T> doSelect(...) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        ConsistentHashSelector<T> selector = selectors.computeIfAbsent(key, k -> 
            new ConsistentHashSelector<>(invokers, methodName, ...));
        return selector.select(invocation);
    }
}

// ConsistentHashSelector：
// ① 每个 Invoker 用 hash(地址) 生成 160 个虚拟节点
// ② 放入 TreeMap（有序）
// ③ select：hash(参数) 找到 >= 它的第一个节点
// ④ 虚拟节点 → 减少数据倾斜
```
### 服务预热 Warmup
```
// 问题：服务刚启动时，JIT 未优化、缓存未预热
// 此时性能差，如果分很多流量 → 打爆

// 解决：预热期间，权重从低逐渐升到正常
// 公式：
weight = weight * (uptime / warmup)
// 启动初期 uptime 小 → weight 小
// 达到 warmup（默认 10 分钟）→ weight 恢复正常
```
```
// AbstractLoadBalance.calculateWarmupWeight
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
    return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```
### 配置与使用
```
<!-- 方式一：XML -->
<dubbo:reference interface="..." loadbalance="leastactive"/>

<!-- 方式二：注解 -->
@Reference(loadbalance = "roundrobin")
private UserService userService;

<!-- 方式三：URL 参数（自适应扩展） -->
// 服务端/客户端配置，也可动态配置
```
```
// 4 种名称：
// random（默认）/ roundrobin / leastactive / consistenthash
```
### 面试高频题
#### 题目1：轮询为什么平滑
```
// 平滑加权轮询：current += weight，选最大，被选者 -= totalWeight
// （Nginx 同款）
```