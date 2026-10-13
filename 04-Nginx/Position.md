## 定位与角色
Nginx是什么->三大角色->正向代理 vs 反向代理->动静分离->高频面试题
### Nginx是什么
Nginx是高性能的HTTP服务器/反向代理服务器。核心定位：高并发、低内存占用和稳定。
```
// 一个 Nginx 可以同时是：
// ① Web 服务器（提供静态资源）
// ② 反向代理（转发请求到后端）
// ③ 负载均衡器（分发请求到多台后端）
// ④ 邮件代理服务器（较少用）

// 核心特点：
// ① 高并发（C10K、C100K 轻松）
// ② 低内存（1万连接约 2.5MB）
// ③ 异步非阻塞（epoll）
// ④ 热部署（不中断服务重载配置）
```
### 三大角色
角色一：静态web服务器
```
# 直接提供静态文件（HTML/CSS/JS/图片）
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    location / {
        # 找静态文件，找到就直接返回
        try_files $uri $uri/ =404;
    }
}

// 优势：
// ① 处理静态资源极快（零拷贝 sendfile）
// ② 不经过后端 Tomcat（省资源）
```
角色二：反向代理
```
# 把请求转发给后端服务器
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;   # 转发到后端
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

// 作用：
// ① 隐藏后端服务器（客户端只看到 Nginx）
// ② 统一入口（安全、日志、限流）
// ③ 转发时带上真实客户端 IP（X-Forwarded-For）
```
角色三：负载均衡
```
# 把请求分发到多台后端
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    location / {
        proxy_pass http://backend;   # 转发到 upstream 集群
    }
}

// 作用：
// ① 分摊压力（多台服务器）
// ② 高可用（一台挂了，转其他）
// ③ 横向扩展（加机器即扩展）
```
### 正向代理 vs 反向代理
代理的是客户端还是服务端
正向代理 代理客户端
```
客户端 → [代理服务器] → 目标服务器
         ↑ 代表客户端去访问

// 客户端知道自己在用代理，服务器不知道真正的客户端

// 特点：
// ① 代理的是"客户端"
// ② 客户端主动配置代理
// ③ 服务器不知道真实客户端是谁
// 场景：科学上网、VPN、公司内网访问外网
```
反向代理 代理服务器
```
客户端 → [反向代理] → 后端服务器
              ↑ 代表服务器接收请求

// 客户端以为在访问服务器，其实被代理转发了

// 特点：
// ① 代理的是"服务器"
// ② 客户端不知道有代理（以为直接访问服务器）
// ③ 服务器知道真实客户端（通过 X-Forwarded-For）
// 场景：Nginx 转发到 Tomcat、CDN
```

|维度|正向代理|反向代理|
|---|---|---|
|**代理对象**|客户端|服务器|
|**谁配置**|客户端|服务器方|
|**客户端知情**|知道|不知道|
|**服务器知情**|不知道真实客户端|知道（X-Forwarded-For）|
|**典型场景**|翻墙、VPN|Nginx 负载均衡|
|**代表**|客户端代理软件|Nginx / 负载均衡|
```
// 正向代理：站在客户端旁边，帮客户端访问
// 反向代理：站在服务器前面，帮服务器挡请求
//
// 一个"陪你去"，一个"替你挡"
```
### 动静分离
```
# 静态请求 → Nginx 自己处理
# 动态请求 → 转发给 Tomcat
server {
    # 静态资源：Nginx 直接返回
    location ~ .*\.(html|css|js|jpg|png|gif)$ {
        root /var/www/static;
        expires 7d;                    # 浏览器缓存 7 天
    }

    # 动态请求：转发给后端
    location /api/ {
        proxy_pass http://backend;
    }
}

// 动静分离的好处：
// ① 静态资源 Nginx 处理更快（不占用 Tomcat）
// ② Tomcat 专注动态业务
// ③ 减少后端压力
// ④ 静态资源可加缓存（expires）
```
请求流向
```
浏览器
  ↓
Nginx
  ├─ 静态请求（.css/.js/图片）→ Nginx 直接返回
  └─ 动态请求（/api）→ Tomcat 处理
```
### Nginx vs 其他服务器
| 服务器        | 特点           | 场景                |
| ---------- | ------------ | ----------------- |
| **Nginx**  | 高并发、轻量、异步    | 反向代理、静态、负载均衡      |
| **Apache** | 多进程/多线程、模块丰富 | 传统 Web（.htaccess） |
| **Tomcat** | Servlet 容器   | Java Web 应用       |
| **IIS**    | Windows 平台   | .NET              |
### 高频面试题
#### 题目1：Nginx是什么
```
// 高性能 HTTP 服务器 + 反向代理 + 负载均衡
```
#### 题目2：正向代理和反向代理的区别
```
// 正向：代理客户端（客户端知情）
// 反向：代理服务器（客户端不知情）
```
#### 题目3：反向代理怎么传真实IP
```
// X-Real-IP / X-Forwarded-For 头
```
