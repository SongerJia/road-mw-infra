> **模块边界**：Docker 的镜像构建 → Docker 模块；Linux 内核机制（Cgroups / Namespace）→ 计算机基础模块；具体 CNI / CSI 插件实现 → 各插件文档。本模块覆盖 K8s 的架构、对象、网络存储与运维。

**① K8s 总览**

1. 是什么、定位（容器编排平台）
2. 与 Docker Swarm 对比
3. 核心能力（服务发现 / 负载均衡 / 自愈 / 扩缩容 / 滚动更新 / 存储编排）
4. 版本演进
5. 应用场景
6. 与 Docker 的关系收口（造镜像 vs 编排；镜像经 OCI 标准流通）

**② 架构** 7. 控制平面组件（kube-apiserver 唯一入口 / etcd 唯一数据源 / scheduler / controller-manager） 8. 工作节点组件（kubelet / kube-proxy / 容器运行时） 9. 容器运行时（containerd / CRI-O，dockershim 已移除） 10. 声明式 API 与控制器模式（期望状态 vs 实际状态、reconcile 调谐循环） 11. 一次部署的完整流程 12. 集群搭建（kubeadm）

**③ 核心对象** 13. Pod 14. ReplicaSet / Deployment 15. StatefulSet 16. DaemonSet 17. Job / CronJob 18. Service 19. Ingress 20. ConfigMap / Secret

**④ Pod** 21. 生命周期与状态、pause 容器（沙箱） 22. Init Container、多容器模式（Sidecar / Ambassador / Adapter） 23. 探针三种（liveness / readiness / startup）及失败后果 24. 资源 requests / limits 25. QoS 三等级与驱逐顺序 26. 调度（nodeSelector / affinity / taint） 27. Pod 优先级与抢占 28. 驱逐、优雅终止（preStop + terminationGracePeriodSeconds）

**⑤ 控制器与配额** 29. Deployment 滚动更新（maxSurge / maxUnavailable） 30. StatefulSet（有状态 / 稳定网络标识 / 有序） 31. DaemonSet 32. ResourceQuota / LimitRange

**⑥ 调度原理** 33. 调度流程（过滤 → 打分 → 绑定）

**⑦ Service 与网络** 34. Service 类型（ClusterIP / NodePort / LoadBalancer / ExternalName）、Headless Service 35. kube-proxy（iptables vs ipvs） 36. Endpoints 与 EndpointSlice 37. Service 负载均衡原理 38. Ingress 与 Ingress Controller 39. CNI 网络插件（Calico / Flannel） 40. Pod 扁平网络模型（每 Pod 一 IP、跨节点直连）

**⑧ 存储** 41. Volume（emptyDir / hostPath） 42. PV / PVC 与绑定 43. StorageClass 与动态供给、CSI 44. ConfigMap / Secret、Secret 类型

**⑨ 自动扩缩容** 45. HPA 与指标来源（CPU / 内存 / 自定义指标） 46. VPA 47. Cluster Autoscaler

**⑩ 运维与发布** 48. kubectl 常用命令 49. 滚动更新与回滚 50. 蓝绿 / 金丝雀发布 51. 排障（describe / logs / exec / events）

**⑪ 扩展与生态** 52. CRD 53. Operator（把运维知识编码进控制器） 54. Helm

**⑫ 安全与坑** 55. RBAC 56. ServiceAccount 57. NetworkPolicy 58. 生产排障清单（Pod 起不来 / 服务不通的定位路径） 59. 常见坑清单（资源未设 limit / 探针配置错误 / 优雅终止未配 / 镜像拉取失败 / 存储未挂载） 60. 边界：Docker 镜像构建 → Docker 模块 61. 边界：Linux 内核机制 → 计算机基础模块 62. 边界：CNI / CSI 插件实现 → 各插件文档