> [!note]
> 基于**时序列数据库（TSDB）**，核心采用**拉取（Pull）模式**从被监控目标中获取监控指标数据
---
# 核心
## 拉取模型(pull)为主
**Prometheus Server** 会根据配置的采样间隔（`scrape_interval`），主动向被监控目标发起 HTTP 请求来拉取指标（Metrics）。这降低了被监控端的压力，避免了因目标端故障或配置错误导致的大量数据拥堵。

## 多维数据模型（Time Series）
Prometheus 将所有数据存储为**时间序列**。每条数据由**指标名称（Metric Name）**和一组**标签（Labels，键值对）**唯一标识。这种多维标签机制使得数据筛选、聚合极其灵活。

## 查询语言（PromQL）
内置专用的 PromQL 语言，支持对多维时间序列数据进行实时的加减乘除、聚合、趋势分析以及切片操作。

## 动态服务发现（Service Discovery）
针对云原生（如 Kubernetes）动态多变的环境，Prometheus 支持**动态发现监控目标**。它能实时自动对接 Kubernetes API、Consul、公有云平台等，动态更新需要抓取的机器、Pod 列表，无需手动频繁修改静态配置。

# 流程
