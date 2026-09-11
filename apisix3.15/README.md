# Apache APISIX 3.15 架构核心与微服务网关实战手册

> 基于官方文档（[Apache APISIX 3.15 Documentation](https://apisix.apache.org/docs/apisix/3.15/getting-started/README/)）与生产级架构实战提炼。深入剖析 APISIX 3.15 的核心架构、Radixtree 路由匹配引擎、全动态配置机制、热加载插件生态、Nacos 微服务注册发现体系，并与 Java 生态主流网关 **Spring Cloud Gateway + Nacos** 进行全方位深度对比与演进迁移分析。

---

## 📑 章节目录导航

| 章节文件 | 对应知识领域 | 核心内容提要 |
| :--- | :--- | :--- |
| **[00. 核心架构与核心概念](./00_APISIX3.15_Architecture_and_Concepts.md)** | 系统架构 / 核心模型 | 数据面与控制面彻底解耦、etcd 毫秒级 Watch 机制、Route/Upstream/Service/Consumer/Plugin/PluginConfig/Secret 模型体系 |
| **[01. 快速上手与部署安装](./01_Quick_Start_and_Installation.md)** | 安装部署 / Admin API | 快速启动脚本、Docker Compose 单机/高可用编排、Standalone 无状态模式、Admin API 认证与配置初始化 |
| **[02. 路由与流量调度实战](./02_Routing_and_Traffic_Management.md)** | 流量治理 / 路由引擎 | Radixtree 路由匹配规则、Upstream 负载均衡（Roundrobin/Consistent Hash 等）、健康检查探针、灰度发布（Traffic Split）、gRPC/WebSocket/TCP/UDP 代理 |
| **[03. 核心插件与安全防护实战](./03_Plugins_and_Security_Ecosystem.md)** | 插件体系 / API 安全 | 认证鉴权（Key-Auth、JWT、OpenID Connect）、流量防护（限流 limit-req、限频 limit-count、限并发 limit-conn）、协议改写、CORS/IP 过滤 |
| **[04. 服务发现与 Nacos 深度集成](./04_Service_Discovery_Nacos_and_Microservices.md)** | 微服务治理 / 服务发现 | APISIX 对接 Nacos 2.x/1.x 注册中心、多租户/命名空间/分组支持、动态拉取与实时同步、Eureka/Consul/K8s 联动 |
| **[05. 多语言插件扩展与开发指南](./05_Plugin_Development_and_Extensibility.md)** | 插件扩展 / 多语言生态 | Lua 原生插件生命周期、**Java Plugin Runner 深度实践**、Go/Python Plugin Runner、WebAssembly (Wasm) 跨语言插件 |
| **[06. 可观测性与生产运维治理](./06_Observability_and_Ops.md)** | 生产运维 / 监控链路 | Prometheus 指标监控、SkyWalking / OpenTelemetry 分布式链路追踪、APISIX Dashboard 管理、高可用容灾与性能调优 |
| **[07. APISIX vs Spring Gateway & Nacos 全维度对比与演进迁移](./07_APISIX_vs_Spring_Gateway_and_Nacos_Comprehensive_Comparison.md)** | 架构选型 / 迁移指南 | **十大维度全景对比矩阵**、性能/GC/内存对比、毫秒级热更 vs 动态重载、多语言与多协议、Spring Cloud 体系平滑迁移至 APISIX 方案 |

---

## 🎯 本手册核心特色

1. **对齐 APISIX 3.15 最新特性**：覆盖 3.15 版本的核心更新（全局规则 Global Rule 行为约束、Secret 密钥管理器、Wasm 增强、Control API 升级等）。
2. **生产实战导向**：提供完整的 `curl`、`Docker Compose`、`config.yaml`、`Java Plugin` 代码与配置，杜绝纸上谈兵。
3. **深入微服务生态**：重点针对国内主流的 **Nacos** 注册中心与微服务架构，提供开箱即用的配置方案与高可用实践。
4. **全方位对比 Java 网关**：专设深度对比章节，从底层模型（Nginx/LuaJIT/Epoll vs Netty/Reactor/JVM）、性能与内存消耗、运维热更能力到架构迁移，为技术选型与改造提供详实依据。
