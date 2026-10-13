## 生命周期
LifeCycle是什么->四大阶段->状态机->观察者模式->事件传播->面试高频题
### LifeCycle是什么
LifeCycle：Tomcat所有组件（Server/Service/Connector/Container）都实现的接口。统一管理组件的创建、启动、停止和销毁，并用观察者模式对外发布生命周期事件。
```
// 为什么需要它？
// Tomcat 组件层层嵌套（Server 里套 Service，Service 里套 Connector/Engine...）
// 启动时：从上到下依次启动（父亲启动子组件）
// 关闭时：从下到上依次关闭（子组件先关）
// → 需要统一的"生命周期管理"

public interface Lifecycle {
    // ① 添加/移除监听器（观察者）
    void addLifecycleListener(LifecycleListener listener);
    LifecycleListener[] findLifecycleListeners();
    void removeLifecycleListener(LifecycleListener listener);

    // ② 生命周期方法
    void init() throws LifecycleException;
    void start() throws LifecycleException;
    void stop() throws LifecycleException;
    void destroy() throws LifecycleException;

    // ③ 状态
    LifecycleState getState();
    String getStateName();
}
```
### 四大阶段
```
// ① init    ：初始化（创建对象、分配资源）
// ② start   ：启动（开始工作，如监听端口）
// ③ stop    ：停止（停止工作，但保留资源）
// ④ destroy ：销毁（释放资源）

// 通常顺序：init → start → (运行) → stop → destroy
```
调用时机
```
// Tomcat 启动：
// Server.init() → Service.init() → Connector/Engine.init()
// Server.start() → Service.start() → Connector/Engine.start()

// Tomcat 关闭：
// Server.stop() → Service.stop() → Engine/Connector.stop()
// Server.destroy() → ...
```
### 状态机 LifecycleState
```
// 每个组件有明确的状态
public enum LifecycleState {
    NEW(false, null),                          // 刚创建
    INITIALIZING(false, Lifecycle.INIT_EVENT), // 初始化中
    INITIALIZED(false, Lifecycle.AFTER_INIT_EVENT), // 已初始化
    STARTING_PREP(false, Lifecycle.BEFORE_START_EVENT), // 启动准备
    STARTING(true, Lifecycle.START_EVENT),     // 启动中
    STARTED(true, Lifecycle.AFTER_START_EVENT), // 已启动
    STOPPING_PREP(true, Lifecycle.BEFORE_STOP_EVENT),  // 停止准备
    STOPPING(false, Lifecycle.STOP_EVENT),     // 停止中
    STOPPED(false, Lifecycle.AFTER_STOP_EVENT), // 已停止
    DESTROYING(false, Lifecycle.BEFORE_DESTROY_EVENT), // 销毁中
    DESTROYED(false, Lifecycle.AFTER_DESTROY_EVENT),   // 已销毁
    FAILED(false, null);                       // 失败
}
```
状态流转
```
NEW
 ↓ init()
INITIALIZING → INITIALIZED
 ↓ start()
STARTING_PREP → STARTING → STARTED
 ↓ stop()
STOPPING_PREP → STOPPING → STOPPED
 ↓ destroy()
DESTROYING → DESTROYED
```
### 观察者模式 LifecycleListener
```
// Lifecycle 继承自 LifecycleBase（抽象类）
// LifecycleBase 内部维护一个监听器列表
// 状态变化时 → 通知所有监听器

public class LifecycleBase implements Lifecycle {
    // 监听器列表
    private final List<LifecycleListener> lifecycleListeners = new CopyOnWriteArrayList<>();

    // 状态变化 → 发布事件
    protected void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this, type, data);
        for (LifecycleListener listener : lifecycleListeners) {
            listener.lifecycleEvent(event);   // 通知
        }
    }
}
```
自定义监听器
```
// 监听生命周期事件
public class MyLifecycleListener implements LifecycleListener {
    @Override
    public void lifecycleEvent(LifecycleEvent event) {
        // event.getType() → 事件类型
        // event.getLifecycle() → 事件源
        if (Lifecycle.START_EVENT.equals(event.getType())) {
            System.out.println("组件启动：" + event.getLifecycle());
        }
    }
}

// 注册（server.xml）
<Listener className="com.example.MyLifecycleListener"/>
```
### 事件传播
```
// 组件嵌套时，生命周期事件会"级联"
// StandardServer.startInternal()
//    → 循环调用每个 Service.start()
//        → Service.startInternal()
//            → 启动每个 Connector 和 Engine
//                → Engine 启动 Host、Context...

// 关闭时反向：
// Server.stop() → Service.stop() → Engine.stop() → Host.stop() ...
```
关键模版方法
```
// LifecycleBase 用"模板方法模式"
public final synchronized void start() {
    // ① 状态校验
    if (state == STARTED) return;
    // ② 设置状态
    setStateInternal(STARTING_PREP);
    // ③ 触发前置事件
    fireLifecycleEvent(BEFORE_START_EVENT, null);
    // ④ 调用子类实现（关键！）
    startInternal();
    // ⑤ 设置状态 + 触发后置事件
    setStateInternal(STARTED);
    fireLifecycleEvent(AFTER_START_EVENT, null);
}

// 子类只需实现 startInternal()（如 StandardEngine.startInternal）

// 这是"模板方法模式"的经典应用：
// 父类（LifecycleBase）定义流程骨架
// 子类（StandardXXX）实现具体逻辑（startInternal）
```
### 面试高频题
#### 题目1：Liftcycle的作用
```
// 统一管理组件生命周期（init/start/stop/destroy）
// 所有 Tomcat 组件都实现它
```

