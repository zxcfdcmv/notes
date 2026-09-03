---
tags:
  - 运维/K8S
  - 问题
---
### 1. 容器端口监听地址错误（最高频）

这是初学者最常踩的坑。容器内的应用监听了 `127.0.0.1`（localhost），但 Pod 网络是隔离的，外部流量通过 Pod IP 进入容器时，目标地址不是 127.0.0.1。

- **现象**：`curl pod_ip:port` 无响应或拒绝连接。
    
- **解决**：检查应用启动命令，强制监听 `0.0.0.0`（如 `node server.js` 需加 `--host 0.0.0.0`，Tomcat 需修改 `catalina.sh`）。
    

### 2. Service 与 Pod 的匹配问题（标签/端口错配）

Service 是流量入口，如果它找不到对应的 Pod，流量就断了。

- **Selector 不匹配**：检查 `spec.selector` 是否完全匹配 Pod 的 `labels`（多了空格或拼写错误都会失效）。
    
- **端口命名冲突**：如果 Service 通过 `targetPort` 名称引用端口，需确认 Pod 的 `containerPort` 定义了对应的 `name`。
    
- **检查命令**：执行 `kubectl get endpoints <service-name>`。如果 `ENDPOINTS` 列为空，说明 Selector 没选中任何 Pod；如果有 IP，说明 Service 发现机制正常。
    

### 3. Service 类型（Type）选择错误

不同类型的 Service 决定了流量能到达的范围：

- **ClusterIP**：只能在集群内部访问。如果你在电脑本地（集群外）直接访问，肯定不通。
    
- **NodePort**：需要确认端口范围（默认 30000-32767）是否被防火墙拦截，且节点 IP 是否可达。
    
- **LoadBalancer**：需要确认云厂商是否真的分配了公网 IP，且安全组/防火墙是否放行了该端口。
    

### 4. Ingress 配置错误（七层路由）

如果使用 Ingress 暴露服务，问题通常出在协议或路径上：

- **路径规则**：`path` 写错（如 `/api` 和 `/api/` 区别），或路径类型 `pathType` 设置不符（`Prefix`/`Exact`）。
    
- **TLS/HTTPS 强制跳转**：后端是 HTTP，但 Ingress 配置了 `https` 后端注解，导致 SSL 握手失败。
    
- **Service 端口名**：Ingress 转发时，Service 的端口定义必须包含协议名称（如 `name: http`）。
    

### 5. Pod 层面的网络策略（NetworkPolicy）限制

集群内可能启用了网络隔离插件（如 Calico, Cilium）。

- 即使 Pod 在跑，如果某个 `NetworkPolicy` 拒绝所有入站流量（`ingress`），则 Service 无法将请求转发进 Pod。
    
- **检查**：执行 `kubectl get networkpolicies -A`，看是否存在限制性策略。
    

### 6. 目标端口（targetPort）数字错误

Service 的 `targetPort` 必须等于 Pod 内应用实际监听的端口，而不是 Service 自己的 `port`。

- **案例**：Service `port` 是 80，但容器内应用跑在 8080，如果 `targetPort` 误写成 80，流量进入容器后没有进程监听 80 端口，连接被拒绝（Connection refused）。
    

### 7. 容器启动延迟或健康检查（ReadinessProbe）失败

虽然 Pod 显示 `Running`，但如果 `ReadinessProbe`（就绪探针）失败，Pod 的 IP 不会被加入 Service 的负载均衡列表（Endpoints 中不会出现该 Pod IP）。

- **检查**：查看 `kubectl describe pod` 中的 `Readiness` 状态，若显示 `Failed`，即使容器跑着，流量也不会转发进去。