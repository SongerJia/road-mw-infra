## 进程模型
进程模型是什么->Master进程->Worker进程->为什么多进程->惊群问题->热重载->面试高频题
### 进程模型是什么
Nginx采用多进程模型：一个Master进程+多个Worker进程。
```
Master 进程（1 个）
    │  管理、调度、热重载
    ├──────────────┬──────────────┐
    ↓              ↓              ↓
Worker 1       Worker 2       Worker 3      ...
    │              │              │
  epoll          epoll          epoll
 处理请求        处理请求        处理请求

// 两个角色分工：
// Master：不处理请求，只负责"管理"
// Worker：不管理，只负责"处理请求"
// → 职责分离，各自专注
```
### Master进程
```
// Master 的职责：
// ① 读取并校验配置文件（nginx.conf）
// ② 启动、管理 Worker 进程
// ③ 接收信号，处理指令（重启、重载、关闭）
// ④ 监控 Worker（Worker 挂了就重启它）
// ⑤ 平滑升级（热部署）
```
关键信号

|信号|含义|
|---|---|
|`nginx -s reload`|重新加载配置（保存旧 Worker，启动新 Worker）|
|`nginx -s stop`|强制停止|
|`nginx -s quit`|优雅停止（处理完当前请求）|
|`nginx -s reopen`|重开日志文件|
|`kill -HUP <master_pid>`|同 reload|
```
# 常用命令
nginx -s reload      # 热重载（不中断服务）
nginx -s quit        # 优雅退出
nginx -t             # 测试配置语法
```
### Worker进程
```
// Worker 的职责：
// ① 接收客户端连接
// ② 处理请求（静态/代理）
// ③ 返回响应
// ④ 每个 Worker 独立，互不干扰

// Worker 数量配置：
worker_processes auto;   # 自动 = CPU 核数
# 或
worker_processes 4;      # 手动指定
```
Worker数量怎么定
```
// ① CPU 密集型：worker_processes = CPU 核数
// ② IO 密集型：可以 = CPU 核数 * 1.5 ~ 2（但一般还是核数）
// ③ 官方推荐：auto（= CPU 核数）

// 为什么 = CPU 核数？
// ① 每个 Worker 是单线程（用 epoll 多路复用）
// ② 避免进程切换开销
// ③ 充分利用多核
```
```nginx
# 配 CPU 亲和绑定（每个 Worker 绑一个核）
worker_processes 4;
worker_cpu_affinity 0001 0010 0100 1000;   # 4 个 Worker 分别绑 4 个核
```
每个Worker的连接数
```nginx
events {
    worker_connections 10240;   # 每个 Worker 最多 10240 连接
}

# 总连接数 = worker_processes × worker_connections
#               4          ×      10240        = 40960
```
### 为什么用多进程
稳定性+无锁+充分利用多核
```
// ① 稳定性（最重要）
//    一个 Worker 崩溃 → 不影响其他 Worker
//    Master 会重新拉起崩溃的 Worker
//    如果是多线程，一个线程崩溃 → 整个进程挂掉

// ② 无锁设计
//    每个 Worker 是独立进程，内存隔离
//    处理请求时不需要加锁（不像多线程共享内存）
//    → 避免锁竞争开销

// ③ 充分利用多核
//    多个 Worker 分散到多个 CPU 核

// ④ 热升级方便
//    可以先启动新 Worker，再优雅关闭旧 Worker
```
多进程 vs 多线程

|维度|多进程（Nginx）|多线程（Apache prefork/worker）|
|---|---|---|
|**稳定性**|高（进程隔离）|低（线程崩溃影响进程）|
|**锁**|无需锁|需锁|
|**内存**|隔离（各自独立）|共享|
|**切换开销**|大（进程切换）|小（线程切换）|
|**适合**|IO 密集型高并发|计算密集|
```
// Nginx 选择多进程的关键理由：
// IO 密集型场景下，"无需锁 + 稳定" 比 "切换开销小" 更重要
```
### 惊群问题
多个Worker同时监听同一个端口，当新连接到来时，多个Worker被同时唤醒，但只有一个accept成功，其他worker被白白唤醒。
```
// 问题：
// Master 创建监听 socket → fork 多个 Worker
// 所有 Worker 都通过 epoll 监听这个 socket
// 新连接到来 → 内核唤醒所有 Worker → 只有一个成功 accept
// → 其他 Worker 白白被唤醒，浪费 CPU

// 解决：accept_mutex（accept 互斥锁）
```
accept_mutex机制
```nginx
events {
    accept_mutex on;              # 开启 accept 互斥（默认 off，新版）
    accept_mutex_delay 500ms;     # 拿不到锁的等待时间
}
```
```
// 原理：
// ① 只有拿到 accept_mutex 锁的 Worker 才去 accept
// ② 避免所有 Worker 同时被唤醒
// ③ 逐个轮转（负载均衡）

// 新版 Nginx（1.11.3+）：
// 默认 accept_mutex off
// 改用 EPOLLEXCLUSIVE 标志（内核层面解决惊群）
```
更细的优化
```
// Nginx 全部用非阻塞 + 多路复用：
// Worker 内：epoll_wait 统一处理所有事件
// 一个新连接不会阻塞 Worker
```
### 热重载 & 平滑升级
热重载 reload
```
nginx -s reload
    ↓
① Master 收到 HUP 信号
② 检查新配置文件的语法
③ 用新配置启动新的 Worker（不删除旧 Worker）
④ 向旧 Worker 发送信号，让它"处理完当前请求后退出"
⑤ 旧 Worker 优雅退出，新 Worker 接管
```
```
// 关键：新旧 Worker 并存一段时间
// 旧 Worker 处理完手头的请求才退出
// → 服务不中断
```
平滑升级 二进制升级
```
// 更换 Nginx 二进制文件（版本升级），不停止服务
// ① 用新二进制启动一个新的 Master
// ② 旧 Master 的 Worker 优雅退出
// ③ 新 Master 接管

// 命令：
// kill -USR2 <old_master_pid>   # 启动新 Master
// kill -WINCH <old_master_pid>  # 优雅关闭旧 Worker
// kill -QUIT <old_master_pid>   # 关闭旧 Master
```
### 面试高频题
#### 题目1：一个连接占几个连接数
```
// 反向代理时：客户端连接 + 后端连接 = 2 个
// 所以 worker_connections 要考虑代理场景
```

