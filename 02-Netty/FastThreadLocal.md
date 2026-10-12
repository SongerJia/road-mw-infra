ThreadLocal回顾->ThreadLocal的问题->FastThreadLocal优化->实现原理->对比->高频面试题
### ThreadLocal回顾
```
// ThreadLocal：每个线程独立副本，线程隔离
public class ThreadLocalDemo {
    private static final ThreadLocal<User> USER = new ThreadLocal<>();

    public void set(User u) { USER.set(u); }
    public User get()      { return USER.get(); }
}
```
ThreadLocal底层结构
```
// ① 每个 Thread 内有 ThreadLocalMap
class Thread {
    ThreadLocal.ThreadLocalMap threadLocals;   // 每个线程一个 Map
}

// ② ThreadLocalMap 是"开放地址法"的哈希表
// key = ThreadLocal 对象（弱引用）
// value = 真正的值（强引用）

// ③ 存取值：
threadLocal.get()
→ 拿到当前线程的 ThreadLocalMap
→ 以 threadLocal 为 key 查表
```
### ThreadLocal的问题
问题一：哈希冲突
```
// ThreadLocalMap 用"线性探测"解决冲突
// hash 冲突时，往下一个槽位找（开放地址法）

// 问题场景：ThreadLocal 多的时候
static ThreadLocal<A> A = new ThreadLocal<>();  // hash = 1
static ThreadLocal<B> B = new ThreadLocal<>();  // hash = 1（冲突！）
static ThreadLocal<C> C = new ThreadLocal<>();  // hash = 1（又冲突！）

// 查找 C 时：命中 1 是 A → 冲突 → 找 2 是 B → 冲突 → 找 3 才是 C
// → 查找效率 O(n)，冲突越多越慢
```
问题二：Hash计算
```
// ThreadLocal 的 hash：
private static final int HASH_INCREMENT = 0x61c88647;  // 黄金分割
private static AtomicInteger nextHashCode = new AtomicInteger();
private final int threadLocalHashCode = nextHashCode.getAndAdd(HASH_INCREMENT);
// hash 用黄金分割递增，但仍会冲突（数组长度限制）
```
问题三：内存泄漏风险
```
// key 是弱引用，value 是强引用
// key 被 GC 后，value 还在 → 泄漏（需手动 remove）
```
### FastThreadLocal优化
Netty的FastThreadLocal用数组下标直接定位，把O(n)查变成O(1)。
核心思想
```
// ThreadLocal：Map 结构，hash 冲突 → O(n)
// FastThreadLocal：数组结构，每个 FastThreadLocal 分配唯一 index
//                 → 直接 array[index] → O(1)
```
对比图
```
ThreadLocal：
Thread.threadLocals (Map)
    [ThreadLocal_A] → value
    [ThreadLocal_B] → value   ← hash 冲突，线性探测
    [ThreadLocal_C] → value

FastThreadLocal：
FastThreadLocalThread.indexedVariables (数组)
    [0] → value_A     ← 直接下标访问 O(1)
    [1] → value_B
    [2] → value_C
```
### 实现原理
①全局唯一index
```
public class FastThreadLocal<V> {
    // ① 每个 FastThreadLocal 实例分配一个唯一的 index
    private final int index;

    public FastThreadLocal() {
        // 从全局原子计数器获取（0, 1, 2, 3...）
        index = InternalThreadLocalMap.nextVariableIndex();
    }
}
```
②FastThreadLocalThread
```
// 用数组存值（而不是 Map）
public class FastThreadLocalThread extends Thread {
    private InternalThreadLocalMap threadLocalMap;   // 数组包装
}

// InternalThreadLocalMap 内部是 Object 数组
public final class InternalThreadLocalMap {
    private Object[] indexedVariables;   // 直接数组

    public Object indexedVariable(int index) {
        return indexedVariables[index];   // O(1) 直接访问
    }
}
```
③set/get过程
```
public final void set(V value) {
    InternalThreadLocalMap map = InternalThreadLocalMap.get();
    int index = this.index;
    Object v = map.indexedVariable(index);
    // 直接放到 array[index]
    map.setIndexedVariable(index, value);
}

public final V get() {
    InternalThreadLocalMap map = InternalThreadLocalMap.get();
    int index = this.index;
    Object v = map.indexedVariable(index);
    if (v == InternalThreadLocalMap.UNSET) {
        return initialize();   // 初始化默认值
    }
    return (V) v;
}
```
④扩容
```
// 数组不够时扩容（类似 ArrayList）
// 每次扩容翻倍，复制旧数据
// 但 index 不变，只是数组变大
```
### 完整对比
| 维度          | ThreadLocal         | FastThreadLocal          |
| ----------- | ------------------- | ------------------------ |
| **存储结构**    | ThreadLocalMap（哈希表） | Object[] 数组              |
| **查找方式**    | hash + 线性探测         | 数组下标直接访问                 |
| **查找复杂度**   | O(n)（冲突时）           | O(1)                     |
| **hash 冲突** | 有                   | 无（index 唯一）              |
| **前提**      | 任何 Thread           | 必须 FastThreadLocalThread |
| **内存泄漏**    | 需 remove            | 需 remove                 |
| **适用范围**    | 通用                  | Netty 内部                 |
使用前提
```
// FastThreadLocal 只有在"FastThreadLocalThread"里才快
// 普通 Thread 里：仍需用 ThreadLocal 兼容（慢回退）

// Netty 的 EventLoop 线程 = FastThreadLocalThread
// → 所以在 Netty 里 FastThreadLocal 才快
```
代码细节
```
// 获取：如果是 FastThreadLocalThread 用数组；否则回退到普通 ThreadLocal
public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        return fastGet((FastThreadLocalThread) thread);   // 数组，快
    } else {
        return slowGet();      // 回退到 ThreadLocal，慢
    }
}
```
### 面试高频题
#### 题目1：FastThreadLocal和ThreadLocal区别
```
// FastThreadLocal 用数组 + index → O(1)
// ThreadLocal 用 Map → 冲突时 O(n)
```
#### 题目2：FastThreadLocal为什么快
```
// 每个 FastThreadLocal 有唯一 index
// 直接数组下标访问，无 hash 冲突
```
#### 题目3：Netty为什么使用它
```
// EventLoop 线程大量使用线程本地变量
// O(1) 查找对高并发至关重要
```
#### 题目4：为什么要remove
```
// 与 ThreadLocal 一样，避免内存泄漏
// 尤其线程复用的场景（线程池/EventLoop）
```
