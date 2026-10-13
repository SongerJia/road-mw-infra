## 核心功能
反向代理->动静分离->缓存->限流->压缩->重写->面试高频题
### 反向代理 proxy_pass
基础配置
```
location /api/ {
    proxy_pass http://backend;      # 转发到后端
    proxy_set_header Host $host;    # 传 Host
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```
proxy_pass带不带斜杠
```
# ① 不带斜杠：保留 location 前缀
location /api/ {
    proxy_pass http://backend;
}
# 请求 /api/user → 转发到 http://backend/api/user（路径原样带上）

# ② 带斜杠：替换 location 前缀
location /api/ {
    proxy_pass http://backend/;
}
# 请求 /api/user → 转发到 http://backend/user（把 /api/ 替换成 /）

# 记忆：
# 带斜杠 = 替换前缀（去掉 /api）
# 不带斜杠 = 保留前缀（带着 /api）
```
代理头 传真实信息
```nginx
proxy_set_header Host $host;                                # 原 Host
proxy_set_header X-Real-IP $remote_addr;                    # 客户端 IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # 代理链
proxy_set_header X-Forwarded-Proto $scheme;                 # 协议（http/https）
```
```
// 为什么需要这些头？
// ① 后端拿到的 remote_addr 是 Nginx 的 IP
// ② 用 X-Real-IP / X-Forwarded-For 传真实客户端 IP
// ③ 用 X-Forwarded-Proto 判断原始协议（HttpServletRequest.getScheme）
```
超时配置
```
proxy_connect_timeout 10s;   # 连接后端超时
proxy_read_timeout 60s;      # 读后端响应超时
proxy_send_timeout 60s;      # 发送给后端超时
```
### 动静分离
```
server {
    # 静态资源：Nginx 直接返回
    location ~ .*\.(gif|jpg|jpeg|png|css|js|html)$ {
        root /var/www/static;
        expires 7d;                 # 缓存 7 天
        add_header Cache-Control "public";
    }

    # 动态请求：转发给后端
    location /api/ {
        proxy_pass http://backend/;
    }
}
```
expires的作用
```
expires 7d;    # 告诉浏览器缓存 7 天，期间不再请求
```
```
// 优势：
// ① 静态资源走 Nginx（快，零拷贝）
// ② 减少 Tomcat 压力
// ③ 浏览器缓存（expires）减少请求
```
### 缓存 proxy_cache
```
# ① 定义缓存区（http 块）
proxy_cache_path /var/cache/nginx
                 levels=1:2
                 keys_zone=my_cache:10m
                 max_size=1g
                 inactive=60m;

# ② 使用缓存（location 块）
location /api/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 302 10m;   # 200/302 缓存 10 分钟
    proxy_cache_valid 404 1m;
    proxy_cache_key "$scheme$host$request_uri";  # 缓存 key
    proxy_pass http://backend;
}
```
缓存参数

|参数|含义|
|---|---|
|`levels=1:2`|缓存目录层级|
|`keys_zone`|共享内存区名 + 大小|
|`max_size`|缓存上限|
|`inactive`|多久没访问就删|
|`proxy_cache_valid`|各状态码缓存时长|
```
// 用途：
// ① 缓存后端响应 → 减少后端压力
// ② 适合读多写少的接口
```
### 限流
①限制请求速率 （limit_req，漏桶算法）
```nginx
# 定义限流区（http 块）
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

# 应用（location 块）
location /api/ {
    limit_req zone=req_limit burst=20 nodelay;
    proxy_pass http://backend;
}
```
```
// rate=10r/s：每个 IP 每秒 10 个请求
// burst=20：突发缓冲 20 个
// nodelay：突发请求立即处理（不延迟）
```
②限制并发连接数（limit_conn）
```
# 定义限流区
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

location /download/ {
    limit_conn conn_limit 5;    # 每个 IP 最多 5 个并发连接
    proxy_pass http://backend;
}
```

|方式|限制|算法|场景|
|---|---|---|---|
|**limit_req**|请求速率（每秒几个）|漏桶|防刷接口|
|**limit_conn**|并发连接数|计数器|防下载占满|
```
// 补充：limit_req 底部是"漏桶算法"
// 请求进桶，桶按固定速率漏出（处理）
// 桶满 → 拒绝
```
### 压缩 gzip
```nginx
http {
    gzip on;
    gzip_min_length 1k;              # 小于 1K 不压缩
    gzip_comp_level 6;               # 压缩级别 1~9
    gzip_types text/plain text/css application/json application/javascript;
    gzip_vary on;                    # 加 Vary: Accept-Encoding
    gzip_disable "MSIE [1-6]\.";     # IE6 不压缩
}
```
```
// 作用：
// ① 压缩响应体（文本类），省带宽
// ② 适合 HTML/CSS/JS/JSON
// ③ 缺点：费 CPU（级别越高越慢）
```
### 重写 rewrite
```
# ① rewrite：重写 URL
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 permanent;   # 301 永久重定向
}

# ② return：直接返回状态码
location /admin/ {
    return 403;                              # 返回 403
}

# ③ 域名跳转
server {
    server_name old.com;
    rewrite ^(.*)$ https://new.com$1 permanent;
}
```
```
// rewrite 标志：
// last    ：重写后重新匹配 location
// break   ：重写后不再匹配 location
// redirect：302 临时重定向
// permanent：301 永久重定向
```
### 其他常用功能
防盗链
```
location ~ .*\.(jpg|png|gif)$ {
    valid_referers none blocked server_names *.example.com;
    if ($invalid_referer) {
        return 403;
    }
}
```
跨域 CORS
```
add_header Access-Control-Allow-Origin *;
add_header Access-Control-Allow-Methods "GET,POST,OPTIONS";
add_header Access-Control-Allow-Headers "Content-Type";
```
SSL终结 HTTPS
```
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;
}
```
```
// SSL 终结：HTTPS 在 Nginx 解密，后端用 HTTP
// 优点：① 证书集中管理 ② 后端不用配 SSL ③ Nginx 处理 SSL 更高效
```
### 面试高频题
#### 题目1：rewrite和return的区别
```
// rewrite：重写 URL
// return：直接返回状态码
```
