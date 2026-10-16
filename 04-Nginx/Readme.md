> **模块边界**：Tomcat 等后端容器 → 各自模块；Linux 内核参数与网络基础 → 系统 / 网络模块；OpenResty 的 Lua 编程 → 独立方向。本模块覆盖 Nginx 的架构、配置、代理、负载、限流与部署。

**① Nginx 总览**

1. 是什么、定位
2. 应用场景（静态服务 / 反向代理 / 负载均衡 / API 网关 / 缓存）
3. 与 Apache 对比
4. 版本（开源 / Plus / Tengine / OpenResty）
5. 模块体系（核心 / 标准 HTTP / 第三方）

**② 架构与进程模型** 6. master / worker 进程模型（多进程 + 每 worker 单线程 + 异步非阻塞） 7. 异步非阻塞 8. worker 数量、CPU 亲和性与 worker_connections 9. 平滑重启与平滑升级（信号语义：reload / USR2 / WINCH）

**③ 事件驱动与高性能** 10. epoll 11. 事件驱动模型 12. Nginx 为什么快（epoll + 异步非阻塞 + 多进程无锁） 13. 与 select / poll 对比 14. 请求处理阶段（11 个 phase）

**④ 配置** 15. nginx.conf 结构 16. 指令上下文 17. 变量（预定义 vs set，作用域）与 map 18. 配置继承与覆盖 19. keepalive_timeout

**⑤ HTTP 核心** 20. server 虚拟主机 21. location 匹配规则与优先级（`=` > `^~` > `~` / `~*` > 普通前缀） 22. root / alias 23. try_files、SPA 前端路由 24. return / rewrite、rewrite flag（last / break / redirect / permanent） 25. error_page 26. if 的陷阱与可用指令

**⑥ 反向代理** 27. proxy_pass（带 / 与不带 / 的差异） 28. 代理头（Host / X-Real-IP / X-Forwarded-For / X-Forwarded-Proto） 29. upstream 30. proxy_buffering 与 proxy_cache 的区分 31. 代理缓存 32. WebSocket 代理（Upgrade / Connection ""）

**⑦ 负载均衡与高可用** 33. upstream 策略、一致性哈希（hash $xxx consistent） 34. 健康检查（被动 / 主动） 35. 后端故障处理（max_fails / fail_timeout / backup / down） 36. upstream keepalive 长连接 37. Nginx + Keepalived（VIP）高可用

**⑧ 缓存与灰度** 38. 代理缓存与缓存键 39. 缓存失效与清理 40. 灰度发布（map + header / cookie / IP 分流）

**⑨ 限流与安全** 41. limit_req（burst / nodelay） 42. limit_conn 43. 黑白名单 44. HTTPS / SSL 配置与优化（session_cache / OCSP / 协议与套件） 45. CORS 跨域配置

**⑩ 动静分离、扩展与运维** 46. 静态资源服务 47. gzip 压缩 48. sendfile、open_file_cache、worker_rlimit_nofile 49. 日志格式定制、日志切割 50. stub_status 监控 51. Lua / OpenResty 扩展（边界标注） 52. 常见坑清单（location 优先级 / if 陷阱 / proxy_pass 斜杠 / XFF 伪造 / root-alias 混用 / rewrite 循环） 53. 生产部署模型（Nginx 前置 + 多实例后端 + Keepalived 双机） 54. 边界：Tomcat 等后端容器 → 各自模块 55. 边界：Linux 内核参数与网络基础 → 系统 / 网络模块 56. 边界：OpenResty 的 Lua 编程 → 独立方向