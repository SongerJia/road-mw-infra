## 高可用与配置
为什么用高可用->Keepalived+VIP->主备/双活->配置结构->location匹配->面试高频题
### 为什么需要高可用
```
// 问题：Nginx 是唯一入口，它挂了 → 整个系统不可用
// 单点故障！
// → 需要多台 Nginx 组成高可用集群

// 方案：Keepalived + VIP（虚拟 IP）
```
架构
```
        客户端
          ↓
      ┌───────────────┐
      │  VIP（虚拟IP） │  192.168.1.100
      └───────────────┘
          ↓ 漂移
   ┌────────────┐    ┌────────────┐
   │ Nginx A    │    │ Nginx B    │
   │ Master     │    │ Backup     │
   │192.168.1.10│    │192.168.1.11│
   └────┬───────┘    └────┬───────┘
        ↓ 心跳检测         ↓
   ┌──────────────────────────┐
   │      后端服务器集群        │
   └──────────────────────────┘
```
Keepalived原理
```
// ① Keepalived 通过 VRRP 协议实现高可用
// ② 两台（或多台）Nginx 共享一个 VIP
// ③ 只有 Master 持有 VIP，对外提供服务
// ④ Master 挂了 → Backup 接管 VIP（漂移）
// ⑤ 客户端无感知（IP 不变）
```
VRRP协议
```
// Virtual Router Redundancy Protocol
// ① 多台机器组成一个虚拟路由器
// ② 通过优先级（priority）选举 Master
// ③ 优先级高的当 Master
// ④ Master 定期发心跳（组播）
// ⑤ Backup 收不到心跳 → 认为 Master 挂了 → 接管
```
配置示例
```
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER                  # 角色（MASTER/BACKUP）
    interface eth0                # 网卡
    virtual_router_id 51          # 虚拟路由 ID（同组一致）
    priority 100                  # 优先级（MASTER 高）
    advert_int 1                  # 心跳间隔（秒）

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        192.168.1.100             # VIP
    }
}
```
检测Nginx是否存活
```
# 脚本：检测 Nginx 进程，挂了就停止 Keepalived（让 VIP 漂移）
#!/bin/bash
if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
    systemctl stop keepalived
fi
```
```
// 关键：Keepalived 默认只检测"自己机器",不检测 Nginx
// 所以要写脚本检测 Nginx，挂了就停止 Keepalived
// → VIP 漂移到 Backup
```
### 主备 vs 双活
主备（Active-Standby）
```
// 一台 Master 提供服务，一台 Backup 待命
// 优点：简单
// 缺点：Backup 浪费（不干活）
```
双活（Active-Active）
```
// 两台都提供服务（如 DNS 轮询 + 双 VIP）
// 优点：资源利用率高
// 缺点：配置复杂
```
```
// 也可以用 LVS + Keepalived 做四层高可用
// Nginx 专注七层
```
### Nginx配置结构（5大块）
```
# ① main：全局配置（最外层）
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;

# ② events：事件配置
events {
    worker_connections 10240;
    use epoll;
}

# ③ http：HTTP 相关（可嵌套 server）
http {
    include mime.types;
    default_type application/octet-stream;

    # ④ server：虚拟主机（可嵌套 location）
    server {
        listen 80;
        server_name example.com;

        # ⑤ location：路由匹配
        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```
层级关系
```
main（全局）
    ├─ events（事件）
    └─ http（HTTP）
         ├─ upstream（后端组）
         ├─ server（虚拟主机）
         │    └─ location（路由）
         └─ server
              └─ location
```
### location匹配规则
```
location [匹配符] uri {
    ...
}
```

|匹配符|含义|优先级|
|---|---|---|
|`=`|精确匹配|最高|
|`^~`|前缀匹配（不再正则）|高|
|`~`|正则匹配（区分大小写）|中|
|`~*`|正则匹配（不区分大小写）|中|
|无|前缀匹配|低|
匹配顺序
```
# ① 先找精确匹配 =（找到就停）
# ② 再找前缀匹配 ^~（找到就停，不再正则）
# ③ 再按顺序找正则匹配 ~ / ~*（第一个匹配就用）
# ④ 最后用普通前缀匹配（最长匹配）

# 记忆：精确 → ^~ → 正则 → 普通前缀
```
示例
```
location = /login {
    # 只匹配 /login（精确）
}
location ^~ /static/ {
    # 匹配 /static/ 开头，命中后不再看正则
}
location ~ \.php$ {
    # 匹配 .php 结尾（正则）
}
location / {
    # 默认匹配（兜底）
}
```
```
// 高频陷阱：
// location = /login 和 location /login 的区别
// = 精确（只匹配 /login）
// 无 = 前缀（匹配 /login、/login/xxx、/loginabc）
```
### 其他常用配置
日志配置
```
http {
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log warn;
}
```
全局常用优化
```
user nginx;
worker_processes auto;

events {
    worker_connections 10240;
    use epoll;
    multi_accept on;             # 一次接受多个新连接
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    client_max_body_size 20m;    # 上传大小限制
    server_tokens off;           # 隐藏版本号
}
```
### 面试高频题
#### 题目1：Nginx怎么实现高可用
```
// Keepalived + VIP
// Master 挂了，VIP 漂移到 Backup
```
#### 题目2：Keepalived原理
```
// VRRP 协议 + 优先级选举 + 心跳检测
```
#### 题目3：只部署Keepalived够吗
```
// 不够！要加脚本检测 Nginx 进程
// Nginx 挂了但 Keepalived 还在 → 需停 Keepalived
```
