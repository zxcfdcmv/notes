---
tags:
    - 运维/prometheus
---
---
# 核心组件
## Prometheus
- **作用**：定义 Prometheus Server 实例。
- **特性**：可以直接在 YAML 中声明副本数（实现高可用）、持久化存储（PVC）、数据保留时间等。Operator 会根据该定义自动创建并管理背后的 `StatefulSet`。

## ServiceMonitor
- **作用**：**最核心、最常用的组件**，用来定义**如何发现并抓取**一组 Kubernetes Service 的监控指标。
- **原理**：它通过 `Label Selector`（标签选择器）去匹配集群中的 Service。一旦匹配成功，Prometheus 就会自动将该 Service 后端的 Pod 加入到抓取目标中（Target）。

## PodMonitor
**作用**：定义 Alertmanager 实例（负责告警的去重、分组和分发）。Operator 会自动将其部署为 `StatefulSet` 并在 Prometheus 中做好关联配置。

## Alertmanager
**作用**：定义 Alertmanager 实例（负责告警的去重、分组和分发）。Operator 会自动将其部署为 `StatefulSet` 并在 Prometheus 中做好关联配置

## PrometheusRule
- **作用**：定义 Prometheus 的告警规则（Alerting Rules）和记录规则（Recording Rules）。
- **优势**：直接用 Kubernetes YAML 编写 PromQL 语句，Operator 会自动将这些规则打包注入到 Prometheus 的配置中。

## Probe
**作用**：用于定义黑盒监控（Blackbox Monitoring），如通过 HTTP/TCP 定期探测某个 URL 或 Ingress 的连通性和响应时间。