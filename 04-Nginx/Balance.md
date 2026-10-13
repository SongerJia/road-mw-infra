## 负载均衡
upstream是什么->内置策略->第三方策略->后端参数->健康检查->面试高频题
### upstream是什么
upstream：Nginx定义的一组后端服务器。负载均衡就是请求按策略分发到upstream里的某台服务器。
```
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    location / {
        proxy_pass http://backend;   # 引用 upstream
    }
}
```
### 内置策略 5种
①轮询 round-robin 默认
```
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
# 默认就是这个，请求轮流分发：10 → 11 → 10 → 11 ...

// 特点：默认，简单，均等
// 适合：后端性能相近
```
②权重 weight
```
upstream backend {
    server 192.168.1.10:8080 weight=3;   # 分 3/4 流量
    server 192.168.1.11:8080 weight=1;   # 分 1/4 流量
}

// 特点：按权重分配（weight 越大分的越多）
// 适合：服务器性能不同（好机器多分）
```
③ip_hash
```
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

// 特点：同一 IP 始终打到同一台服务器
// 原理：hash(客户端 IP) % 服务器数
// 用途：保持会话（Session 粘性）
// 缺点：① 分配不均（NAT 后大量用户同 IP）
//       ② 一台挂了，该 IP 的会话丢失
```
④least_conn 最少连接
```
upstream backend {
    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

// 特点：发给当前连接数最少的服务器
// 适合：请求处理时间差异大（谁闲给谁）
```
⑤hash 自定义key
```
upstream backend {
    hash $request_uri;      # 按 URI hash
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

// 特点：按指定 key（如 URI、参数）hash
// 用途：同一资源打到同一台（利于缓存）
```
### 第三方策略
fair 按响应时间
```
upstream backend {
    fair;                  # 需要 nginx-upstream-fair 模块
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

// 特点：按后端响应时间分配（响应快的多分）
// 缺点：需要额外编译模块
```
url_hash
```
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    hash $request_uri consistent;   # 一致性 hash
}

// 特点：同一 URL 打到同一台（利于缓存命中）
// consistent：一致性 hash（减少节点变动影响）
```
### 策略对比与选型
| 策略             | 均衡度   | 会话保持     | 适用        |
| -------------- | ----- | -------- | --------- |
| **轮询**         | 均等    | ❌        | 后端性能相近    |
| **权重**         | 按比例   | ❌        | 性能不同      |
| **ip_hash**    | 可能不均  | ✅        | 需要会话粘性    |
| **least_conn** | 动态    | ❌        | 请求耗时差异大   |
| **fair**       | 动态    | ❌        | 追求最优（需模块） |
| **hash**       | 按 key | ✅（同 key） | 缓存命中      |
选型建议
```
// ① 默认用轮询（或用权重）
// ② 需要 Session 保持 → ip_hash
//    （更推荐：不用 ip_hash，改用 Redis 共享 Session）
// ③ 请求耗时差异大 → least_conn
// ④ 缓存优化 → hash $request_uri
```
### 后端服务器参数
```
upstream backend {
    server 192.168.1.10:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 down;      # 手动下线
    server 192.168.1.12:8080 backup;    # 备用（主全部挂了才用）
    keepalive 32;                       # 长连接池
}
```
参数说明

|参数|含义|
|---|---|
|**weight**|权重|
|**down**|手动标记下线|
|**backup**|备用服务器|
|**max_fails**|最大失败次数（超过则标记不可用）|
|**fail_timeout**|失败超时（间隔多久重试）|
|**keepalive**|与后端的长连接数|
```
// max_fails + fail_timeout 组合：
// 30s 内失败 3 次 → 标记不可用 → 30s 后再试
// 这是 Nginx 自带的"被动健康检查"
```
### 健康检查
被动检查
```
// 通过 max_fails / fail_timeout 实现
// 请求失败时计数，超过阈值标记下线
// 缺点：被动（要等请求失败才发现）
```
主动检查 需模块
```
# 需要 nginx_upstream_check_module
upstream backend {
    server 192.168.1.10:8080;
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```
```
// 主动定期探活（发 HTTP 请求）
// interval：检查间隔
// rise：连续成功几次标记健康
// fall：连续失败几次标记不健康
```
Sunginx/OpenResty
```
// OpenResty（基于 Nginx + Lua）可以实现更灵活的健康检查
```
### Session问题
```
// 用 ip_hash 保会话的缺点：
// ① 分配不均
// ② 服务器挂 → 会话丢失
// ③ 扩展困难

// 更好的方案：Session 共享
// ① Session 存 Redis（所有后端共享）
// ② Token/JWT（无状态）
// ③ 一致性 hash（尽量落到同一台）
```
```
// 面试回答：
// "负载均衡本身不解决 Session，
//  用 ip_hash 是权宜之计，
//  生产推荐 Redis 集中式 Session 或 JWT 无状态"
```
