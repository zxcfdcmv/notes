---
tags:
    - 运维/prometheus
---
> [!note]
> 基于**时序列数据库（TSDB）**，核心采用**拉取（Pull）模式**从被监控目标中获取监控指标数据
---
# 核心
## 1. 拉取模型(pull)为主
**Prometheus Server** 会根据配置的采样间隔（`scrape_interval`），主动向被监控目标发起 HTTP 请求来拉取指标（Metrics）。这降低了被监控端的压力，避免了因目标端故障或配置错误导致的大量数据拥堵。

## 2. 多维数据模型（Time Series）
Prometheus 将所有数据存储为**时间序列**。每条数据由**指标名称（Metric Name）**和一组**标签（Labels，键值对）**唯一标识。这种多维标签机制使得数据筛选、聚合极其灵活。

## 3. 查询语言（PromQL）
内置专用的 PromQL 语言，支持对多维时间序列数据进行实时的加减乘除、聚合、趋势分析以及切片操作。

## 4. 动态服务发现（Service Discovery）
针对云原生（如 Kubernetes）动态多变的环境，Prometheus 支持**动态发现监控目标**。它能实时自动对接 Kubernetes API、Consul、公有云平台等，动态更新需要抓取的机器、Pod 列表，无需手动频繁修改静态配置。

---
# 流程
## 1. 指标暴露或中转（Metrics Exposure）
- **Exporters（采集器）**：针对不支持直接输出 Prometheus 格式的第三方软件（如 MySQL、Redis、Linux 系统等），部署专门的 Exporter 充当代理，收集原生指标并转化为 HTTP 格式暴露出来（默认路径通常是 `/metrics`）。
- **Client Libraries（客户端库）**：在业务代码中直接集成 Prometheus SDK，自定义埋点输出业务指标。
- **Pushgateway（推送网关）**：针对生命周期短的定时任务（Ephemeral Jobs），任务结束后便无法被 Pull。因此这类任务会在运行时将数据推送到 Pushgateway，再由 Prometheus Server 从中转站统一 Pull。
## 2. 目标发现与过滤（Service Discovery & Scrape）
- Prometheus Server 通过**静态配置**（`static_configs`）或**服务发现**（`Service Discovery`）机制，动态获取当前存活的监控目标列表。
- 在拉取之前，利用**重新打标签（Relabeling）**机制，对抓取的目标、标签进行过滤或重构。

## 3. 数据拉取与存储（Data Scraping & Storage）
- Prometheus Server 依据设定的时间间隔，向目标组件发送 HTTP GET 请求拉取数据。
- 抓取到的时间序列数据会实时写入本地的 **TSDB（时序数据库）** 块中。默认情况下在本地保留 15 天，也可以配置接入远程存储（Remote Storage）实现数据的长期归档。

## 4. 规则评估与告警触发（Rules Evaluation）
- Prometheus 内部的**规则引擎**会周期性地执行用户定义好的 PromQL 告警规则（Alerting Rules）。
- 一旦计算结果触发了阈值（例如：CPU 使用率 > 90% 持续 5 分钟），Prometheus Server 并不会直接发邮件或短信，而是将生成的告警消息**推送给 Alertmanager**。
## 5. 告警处理与可视化展示（Alerting & Visualization）
- **告警投递**：Alertmanager 接收到告警后，进行**去重、分组、静默、抑制**等降噪处理，随后通过路由规则，将告警发送到指定的接收媒介（如 邮件、钉钉、企业微信、PagerDuty 等）。
- **数据可视化**：用户可以使用 Prometheus 自带的 Web UI 进行简单的 PromQL 查询和图表展示，但生产环境中通常会将 Prometheus 作为数据源接入 **Grafana**，配置丰富、炫酷的监控大屏。