## 类加载机制
双亲委派->Tomcat类加载器层次->为什么打破->WebappClassLoader逻辑->应用隔离->面试高频题
### 双亲委派
```
// JDK 默认三个类加载器：
// Bootstrap ClassLoader（C++ 实现，加载 rt.jar / java.*）
//     ↑
// Extension ClassLoader（加载 ext 目录）
//     ↑
// Application ClassLoader（加载 classpath）
//     ↑
// 自定义 ClassLoader

// 双亲委派机制：
// ① 收到加载请求 → 先委派给父加载器
// ② 父加载器再委派给它的父...
// ③ 直到 Bootstrap
// ④ 父加载器加载不了 → 子加载器自己加载

protected Class<?> loadClass(String name, boolean resolve) {
    // ① 检查是否已加载
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            // ② 委派父加载器
            if (parent != null) c = parent.loadClass(name, false);
            else c = findBootstrapClassOrNull(name);
        } catch (ClassNotFoundException e) {}
        // ③ 父加载不了 → 自己加载
        if (c == null) c = findClass(name);
    }
    return c;
}
```
为什么要双亲委派
```
// ① 安全：防止核心类被篡改（自己写 java.lang.String 也加载不了）
// ② 唯一：同一个类只被加载一次（父加载器优先）
```
### Tomcat的类加载器层次
Tomcat在JDK的基础上，扩展出多个类加载器，形成树状结构。
```
Bootstrap ClassLoader（JDK 启动类加载器）
    ↑
System ClassLoader（JDK 应用类加载器）
    ↑
Common ClassLoader（Tomcat 通用，加载 tomcat/lib）
    ├──→ Catalina ClassLoader（加载 Tomcat 自身类）
    └──→ Shared ClassLoader（加载所有 Web 应用共享类）
               ↑
         Webapp ClassLoader（每个 Web 应用一个！）
               ↑
         JasperLoader（每个 JSP 一个，支持 JSP 热更新）
```
各加载器职责

|加载器|加载内容|说明|
|---|---|---|
|**Bootstrap**|JDK 核心类|根|
|**System**|CLASSPATH|Tomcat 启动脚本设置|
|**Common**|`tomcat/lib`|公共类（Connector/Container）|
|**Catalina**|Tomcat 自身类|通常与 Common 合并|
|**Shared**|所有 Web 应用共享|通常与 Common 合并|
|**Webapp**|单个 Web 应用 `WEB-INF/classes`、`WEB-INF/lib`|**每个应用一个**|
|**JasperLoader**|JSP 编译的类|**每个 JSP 一个**|
```
// 默认配置下（conf/catalina.properties）：
// common.loader = Common 加载
// server.loader = Catalina 加载（默认空，含在 common）
// shared.loader = Shared 加载（默认空，含在 common）

// 所以实际常见是三层：
// Bootstrap → System → Common → Webapp → JasperLoader
```
### 为什么要打破双亲委派
双亲委派的父优先策略，无法满足Wen容器的需求。
原因一：Web应用隔离
```
// 场景：两个 Web 应用 A、B
// A 依赖 Spring 4.x，B 依赖 Spring 5.x
// 如果按双亲委派（父优先），只能加载一个版本的 Spring
// → 冲突！

// 解决：每个 Web 应用有自己的 WebappClassLoader
// 各加载各的 Spring，互不干扰
```
原因二：支持热部署
```
// 场景：重新部署 Web 应用（不重启 Tomcat）
// ① 丢弃旧的 WebappClassLoader
// ② 创建新的 WebappClassLoader
// ③ 重新加载应用类

// 如果按双亲委派，类被父加载器加载后无法卸载
// → 必须让 WebappClassLoader 优先自己加载，才能整体卸载
```
原因三：避免应用类污染
```
// Web 应用里的类，不应该被 Tomcat 的其他部分看到
// 各应用之间也要隔离
```
原因四：JSP热更新
```
// JSP 修改后要立即生效
// JasperLoader 每个 JSP 一个 → 改了 JSP 就换类加载器
```
### WebappClassLoader的加载逻辑
打破父优先，改为自己优先（除了少数例外）
```
// WebappClassLoaderBase.loadClass() 简化逻辑：
public Class<?> loadClass(String name) {
    // ① 检查是否已加载
    Class<?> clazz = findLoadedClass(name);
    if (clazz != null) return clazz;

    // ② 特殊类：JVM 核心类，委派给父（不能自己加载）
    if (name.startsWith("java.") || name.startsWith("javax.")
        || name.startsWith("sun.") || ...) {
        return parent.loadClass(name);   // 委派父
    }

    // ③ 先尝试自己加载（打破双亲委派的关键！）
    try {
        clazz = findClass(name);   // 从 WEB-INF/classes、WEB-INF/lib 加载
        if (clazz != null) return clazz;
    } catch (ClassNotFoundException e) {}

    // ④ 自己加载不了 → 委派父加载器
    return parent.loadClass(name);
}
```
加载顺序
```
WebappClassLoader 加载一个类：
1. 检查本地缓存（findLoadedClass）
2. 检查 JVM 核心类（java.* 等）→ 委派父
3. 从 WEB-INF/classes 加载（自己）
4. 从 WEB-INF/lib 加载（自己）
5. 从 System/Common 加载（父）
```
### 应用隔离与热部署
应用隔离
```
// 每个 Web 应用一个 WebappClassLoader
// A 应用的类，B 应用看不到
// → 类层面的隔离

// 但：java.* 等核心类是共享的（委派父）
// → 基础类唯一，应用类隔离
```
热部署 Reload
```
// 应用重新加载：
// ① 停止应用（Context.stop）
// ② 销毁旧的 WebappClassLoader
// ③ 创建新的 WebappClassLoader
// ④ 重新加载应用类
// ⑤ 启动应用

// 为什么能"卸载"类？
// 因为类由 WebappClassLoader 加载
// 没有其他引用了 → 类加载器 + 类一起被 GC
```
类的卸载条件
```
// 一个类能被卸载，需同时满足：
// ① 类的所有实例都被回收
// ② 类的 ClassLoader 被回收
// ③ 类的 Class 对象没有被引用

// 所以：只要丢弃 WebappClassLoader，它的类就能卸载
```


