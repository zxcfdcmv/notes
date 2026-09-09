---
tags:
    - 运维/prometheus
---
> [!tip]
> 让 proemtheus 根据标签（Label）去自动发现并抓取这一组 Service 的监控指标
---
# 工作原理
1. **标签关联**：业务 Pod 暴露 `/metrics` 端口，并且有一个 K8s Service 代理了这些 Pod。
2. **动态发现**：`ServiceMonitor` 通过 `spec.selector` 匹配到这个 K8s Service。
3. **配置渲染**：Prometheus Operator 监测到集群中多了一个 `ServiceMonitor`，并且它的标签满足 Prometheus 实例的 `serviceMonitorSelector` 要求。
4. **自动热重载**：Operator 自动把这个 `ServiceMonitor` 翻译成 Prometheus 能够理解的 `kubernetes_sd_configs`（K8s 服务发现规则），写入 Prometheus 配置并触发 `/-/reload` 接口。监控立刻生效。
---
# 配置案例
## 被监控的业务 Service
```yml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: dev-team-a
  labels:
    app: order-service-app # 1. 业务 Service 自身的标签
spec:
  ports:
  - name: metrics-port     # 2. 必须给监控端口起一个名字！
    port: 8080
    targetPort: 8080
  selector:
    app: order-service-pod
```
## 配套的 `ServiceMonitor` 配置文件
