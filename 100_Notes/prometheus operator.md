---
tags:
    - 运维/prometheus
---
---
> [!note]
> **Prometheus Operator** 是由 CoreOS（现为 Red Hat）开源的项目，它基于 Kubernetes 的 **Operator 模式**，使用 **Custom Resource Definition（CRD）** 将 Prometheus 及其相关组件的部署、配置和管理工作封装成 Kubernetes 原生资源。


> [!tip] 简单来说
> **用 Kubernetes 的 YAML 文件来声明式地管理 Prometheus 集群，而不用手动维护配置文件和服务。**
# 核心组件
## Prometheus
- **作用**：定义 Prometheus Server 实例。
- **特性**：可以直接在 YAML 中声明副本数（实现高可用）、持久化存储（PVC）、数据保留时间等。Operator 会根据该定义自动创建并管理背后的 `StatefulSet`。

## ServiceMonitor
- **作用**：**最核心、最常用的组件**，用来定义**如何发现并抓取**一组 Kubernetes Service 的监控指标。
- **原理**：它通过 `Label Selector`（标签选择器）去匹配集群中的 Service。一旦匹配成功，Prometheus 就会自动将该 Service 后端的 Pod 加入到抓取目标中（Target）。

## PodMonitor
- **作用**：定义 Alertmanager 实例（负责告警的去重、分组和分发）。Operator 会自动将其部署为 `StatefulSet` 并在 Prometheus 中做好关联配置。

## Alertmanager
- **作用**：定义 Alertmanager 实例（负责告警的去重、分组和分发）。Operator 会自动将其部署为 `StatefulSet` 并在 Prometheus 中做好关联配置

## PrometheusRule
- **作用**：定义 Prometheus 的告警规则（Alerting Rules）和记录规则（Recording Rules）。
- **优势**：直接用 Kubernetes YAML 编写 PromQL 语句，Operator 会自动将这些规则打包注入到 Prometheus 的配置中。

## Probe
- **作用**：用于定义黑盒监控（Blackbox Monitoring），如通过 HTTP/TCP 定期探测某个 URL 或 Ingress 的连通性和响应时间。

# 工作原理
> [!tip]
> Prometheus Operator 的核心工作是一个**持续调和（Reconciliation Loop）**的过程

- **配置监听**：Prometheus Operator 服务在后台持续监听 Kubernetes API，关注 `Prometheus`、`ServiceMonitor`、`PrometheusRule` 等资源的变化。
- **配置转换**：当用户创建或修改了一个 `ServiceMonitor` 时，Operator 会捕获该事件，并将多份 `ServiceMonitor` 和 `PrometheusRule` 的内容**拼接、转换为原生 `prometheus.yml` 格式**，最后保存到一个 Kubernetes `Secret` 对象中。
- **无缝热加载**：在 Prometheus Pod 内，运行着一个名为 `config-reloader` 的 Sidecar 容器。它负责监控这个 Secret。一旦发现配置改变，它会立刻向 Prometheus 发送一个 HTTP `POST /-/reload` 请求。**整个过程业务无感知，不需要重启 Prometheus 服务**。