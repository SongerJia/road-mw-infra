> **模块边界**：Kubernetes 编排细节 → K8s 模块；Linux 内核机制（Namespace / Cgroups 的底层实现）→ 计算机基础模块。本模块覆盖 Docker 的原理、镜像、容器、网络存储与落地实践。

**① Docker 总览**

1. 是什么、定位（容器化引擎）
2. 与虚拟机的本质区别（共享内核 vs 独立内核）
3. 核心概念（镜像 / 容器 / 仓库）
4. 应用场景
5. 版本与 OCI 标准（镜像规范 / 运行时规范）
6. 与 K8s 的关系（构建运行 vs 编排；dockershim 弃用）

**② 核心原理** 7. Namespace（六种类型：PID / NET / IPC / MNT / UTS / USER） 8. Cgroups（CPU / 内存 / IO 限制） 9. UnionFS 与存储驱动（overlay2 / aufs） 10. 镜像只读层 vs 容器可写层、Copy-on-Write 11. 容器与进程的关系、PID 1 问题（信号处理 / 孤儿进程回收） 12. Docker 进程架构（dockerd → containerd → runc）

**③ 镜像** 13. 镜像分层、共享与缓存 14. 镜像构建 15. tag 与 digest、latest 漂移问题 16. 多阶段构建 17. 镜像体积优化（合并层 / 最小基础镜像 / 清理）

**④ Dockerfile** 18. 全部指令（FROM / RUN / COPY / ADD / CMD / ENTRYPOINT / EXPOSE / ENV / ARG / WORKDIR / VOLUME / USER / HEALTHCHECK） 19. CMD vs ENTRYPOINT 20. COPY vs ADD 21. ARG vs ENV 22. 构建缓存机制（顺序 / .dockerignore） 23. 最佳实践

**⑤ 容器** 24. 生命周期与状态 25. 运行（前台 / 后台） 26. 日志与日志驱动、日志膨胀问题 27. exec 与 attach 28. 资源限制与 OOM 29. 健康检查

**⑥ 网络** 30. 网络模式（bridge / host / none / container / 自定义） 31. 端口映射与 iptables DNAT 32. 容器互联与 DNS 33. 网络实现（veth pair / 网桥）

**⑦ 存储** 34. 数据卷、数据随容器消失的坑 35. 绑定挂载 36. tmpfs 与数据卷的备份迁移 37. 容器时间 / 时区处理

**⑧ Compose** 38. Compose 文件与服务编排 39. 多容器应用实践

**⑨ 镜像仓库** 40. Docker Hub 与镜像推拉 41. Harbor 与私有仓库

**⑩ CI/CD 与运维** 42. 镜像构建流水线 43. 容器监控与日志 44. 清理（prune）与磁盘管理

**⑪ 最佳实践、安全与坑** 45. 容器化最佳实践（一容器一进程 / 无状态 / 配置外置 / 日志到 stdout） 46. 容器化改造检查表 47. 镜像构建优化清单 48. capabilities 与 root 问题、镜像安全扫描 49. 常见坑清单（日志膨胀 / 数据卷未挂 / 镜像过大 / 时区错误 / latest 漂移） 50. 边界：K8s 编排细节 → K8s 模块 51. 边界：Linux 内核机制 → 计算机基础模块