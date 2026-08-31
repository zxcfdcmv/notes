---
tags:
    - 运维
    - K8S
---

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: default
spec:
  # 目标对象：针对哪个工作负载扩缩容
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-deployment
  
  # 副本数边界（必填）
  minReplicas: 2
  maxReplicas: 10

  # 核心：指标配置（v2 版本支持多指标）
  metrics:
  # 1. 资源指标（CPU）
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization        # 用法：Utilization（使用率）或 AverageValue（平均值）
        averageUtilization: 60   # 期望 Pod 平均 CPU 使用率为 60%
  
  # 2. 资源指标（内存）- 可选
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi      # 期望每个 Pod 内存平均值不超过 500Mi

  # 3. 自定义指标（例如 Prometheus 采集的 QPS）- 需安装 Prometheus Adapter
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 10         # 期望每个 Pod 平均 QPS 为 10

  # 扩缩容行为策略（v2 支持细粒度控制，强烈建议生产环境配置）
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 缩容冷却窗口：5 分钟内指标持续低于阈值才缩容
      policies:
      - type: Percent
        value: 50               # 每次缩容最多减少 50% 的 Pod
        periodSeconds: 60       # 每 60 秒内最多执行一次
      selectPolicy: Min         # 取多个策略中效果最保守的（Max 则最激进）
    scaleUp:
      stabilizationWindowSeconds: 0     # 扩容无冷却（默认即为 0，立即响应）
      policies:
      - type: Percent
        value: 100              # 每次扩容最多翻倍
        periodSeconds: 15       # 每 15 秒可扩容一次
      - type: Pods
        value: 4                # 或每次最多增加 4 个 Pod
        periodSeconds: 15
      selectPolicy: Max         # 取扩容量最大的策略（快速响应）
```