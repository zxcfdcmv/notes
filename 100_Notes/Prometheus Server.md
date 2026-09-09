---
tags:
    - 运维/prometheus
---

> [!note]
> 拉取（Pull） ──> 存储（TSDB） ──> 计算（Rules） ──> 提供查询（PromQL）

---
# 核心架构

Prometheus Server 内部主要由以下三个核心模块协同工作：

```
                ┌──────────────────────────────────┐
                │     Prometheus Server            │
                │                                  │
┌──────────┐    │  ┌──────────────────┐            │
│ 监控目标 ├────┼─►│ 1. 抓取引擎      │            │
│ (Targets)│Pull│  │   (Scrape Loop)  │            │
└──────────┘    │  └────────┬─────────┘            │
                │           │ 写入                 │
                │           ▼                      │
                │  ┌──────────────────┐   触发告警 │                                      │  │                  │            │     ┌──────────────┐
                │  │ 2. 时序数据库    ├────────────┼────►│ Alertmanager │
                │  │   (TSDB Engine)  │   (Rule Engine)  │              │                 │  │                  │            │     └──────────────┘
                │  └────────▲─────────┘            │
                │           │ 读取                 │
                │  ┌────────┴─────────┘            │
                │  │ 3. 查询引擎      │            │
                │  │   (PromQL Engine)│            │
                │  └────────▲─────────┘            │
                └───────────┼──────────────────────┘
                            │ HTTP API 查询
                     ┌──────┴──────┐
                     │ Grafana / UI│
                     └─────────────┘
```

1. 抓取引擎 (Scrape Engine / Scrape Loop)
   - **职责**：负责把外面的监控指标“拿过来”。
   - **工作机制**：
        1. 抓取引擎会根据 `prometheus.yml` 中配置的 **服务发现 (Service Discovery)** 机制（如 K8s API、Consul 或静态配置），实时维护一份“待抓取目标列表”。
        2. 它为每个目标启动一个独立的**抓取循环 (Scrape Loop)**。
        3. 按照设定的采样周期（`scrape_interval`，如 15s），定期向目标的 HTTP 端点（默认 `/metrics`）发送 GET 请求。
        4. 接收到标准的文本格式指标后，进行**重新打标签（Relabeling）**处理，规范化标签命名。
1. 时序数据库 (TSDB Engine)
   - **职责**：负责把拿到的数据“存下来”，并保证读写极快。
   - **工作机制**：
        1. **内存缓冲 (Head Block)**：抓取引擎刚拉回来的最新数据（当前样本值和时间戳），会先直接写入内存中的 Head 块。为了防止断电数据丢失，数据写入内存的同时会同步追加写入 **预写日志 (WAL, Write-Ahead Log)**。
        2. **落盘与压缩 (Block Flush)**：每隔 2 小时，Head 块中的内存数据会被重构、压缩，并作为一个独立的**数据块 (Block)** 持续持久化到磁盘上。
        3. **长期存储**：默认在本地只保留 15 天。如果需要海量长周期存储，Prometheus Server 会通过 **Remote Write（远程写）** 接口将数据实时转发给分布式时序数据库（如 Thanos, Cortex, VictoriaMetrics）。
1. 查询引擎与规则引擎 (Query & Rule Engine)
   - **职责**：负责把存下来的数据“用起来”（计算告警、提供给 Grafana 展示）。
   - **工作机制**：
    1. **PromQL 解析**：解析用户或 Grafana 发送过来的 PromQL 语句，通过内置的高效算法从 TSDB 中提取特定标签和时间范围的时序数据。
    2. **定期规则计算**：规则引擎会在后台启动定时任务（依据 `evaluation_interval`，如 30s），循环执行用户定义的 `PrometheusRule` 告警规则。如果计算结果满足告警阈值，则将告警数据打包发送给外部的 Alertmanager。

---
# 配置
## 传统
```yml
# 1. 全局配置
global:
  scrape_interval:     15s # 默认每 15 秒拉取一次数据
  evaluation_interval: 15s # 默认每 15 秒评估一次告警规则
  scrape_timeout:      10s # 每次拉取请求的超时时间
  
  # 附加到当前 Prometheus 实例拉取的所有时序数据上的全局标签（多实例高可用时非常有用）
  external_labels:
    cluster: 'production-k8s'
    replica: 'prometheus-0'

# 2. 告警服务器配置（指定把告警推给谁）
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - 'alertmanager:9093' # Alertmanager 的地址

# 3. 规则文件加载（告警规则与记录规则）
rule_files:
  - "/etc/prometheus/rules/*.yml"

# 4. 抓取作业配置（Scrape Jobs）
scrape_configs:
  # 示例 A：静态配置（最简单的模式，写死 IP/端口）
  - job_name: 'prometheus-self'
    static_configs:
    - targets: ['localhost:9090'] # 监控 Prometheus 自身

  # 示例 B：K8s 环境下的动态服务发现（Service Discovery）
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod # 自动动态发现集群中所有的 Pod
    
    # 重新打标签（Relabeling）：过滤出只带有特定注解的 Pod 进行监控
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep # 只有当 Pod 包含 annotations: prometheus.io/scrape: "true" 时才抓取
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__ # 动态修改抓取路径（默认是 /metrics）
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__ # 动态组合出正确的 Pod IP 和端口
```

## Prometheus Operator
```yml
apiVersion: ://coreos.com
kind: Prometheus
metadata:
  name: k8s-prometheus
  namespace: monitoring # 通常部署在专用的监控命名空间下
  labels:
    prometheus: k8s-prometheus
spec:
  # 1. 基础集群属性
  image: quay.io/prometheus/prometheus:v3.1.0 # 指定 Prometheus 的镜像版本 [1]
  replicas: 2 # 部署 2 个 Pod 实现高可用（HA），它们会拉取完全相同的数据
  version: v3.1.0
  serviceAccountName: prometheus-k8s # 需要赋予该账户读取 K8s API 的权限（用于服务发现）

  # 2. 存储与保留策略
  retention: 15d # 数据在本地 TSDB 保留 15 天
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: "gp3-sc" # 指定 K8s 的存储类，自动绑定块存储（如 AWS EBS/阿里云盘）
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi # 每个副本分配 100GB 存储空间

  # 3. 对接外部 Alertmanager
  alerting:
    alertmanagers:
    - name: alertmanager-main # 对应集群中 Alertmanager Service 的名字
      namespace: monitoring
      port: web # 对应的端口名称

  # 4. 核心：标签选择器（指定该实例去加载哪些监控目标和规则）
  # 只有匹配了这些 Label 的 ServiceMonitor 和 PrometheusRule 才会生效！
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack # 匹配带有此标签的 ServiceMonitor
  
  podMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack # 匹配带有此标签的 PodMonitor

  ruleSelector:
    matchLabels:
      role: alert-rules # 匹配带有此标签的 PrometheusRule 告警规则

  # 5. 安全性与高级功能
  enableAdminAPI: false # 生产环境建议关闭 Admin API（防止误删数据）
```