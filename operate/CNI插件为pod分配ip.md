> [!note] 总结
> 1. **集群级/节点级的网段划分（IPAM）**
> 2. **容器启动时的网络命名空间配置**

---
# 核心机制: IPAM
> [!tip]
> CNI 插件通常会内置或调用一个 **IPAM（IP Address Management）** 插件。它的主要职责是管理和分配 IP 资源，防止 IP 冲突

1. **Host-Local 模式（按节点分段，如 Flannel / Calico 默认）**
    - **原理**：在集群部署时，会定义一个巨大的集群 Pod 网段（Cluster CIDR，例如 `10.244.0.0/16`）。
    - **节点划分**：Kubernetes 的 `kube-controller-manager` 会为每个加入集群的 Node 节点分配一个子网段（Node CIDR，例如 Node A 分配 `10.244.1.0/24`，Node B 分配 `10.244.2.0/24`）。
    - **本地分配**：当 Pod 被调度到某个 Node 上时，该节点上的 CNI 插件只需从自己本地的这个子网段中挑一个未使用的 IP 分配给 Pod 即可。这种方式速度极快，不需要全局加锁。
2. **Cilium / Calico CRD 模式（全局/动态分配）**
    - **原理**：不固定节点网段，而是通过自定义资源（CRD，如 Calico 的 `IPPool`）来全局管理 IP。
    - **动态申请**：节点上的 CNI 代理会动态地向集群申请一块 IP 块（Block），用完后再申请，这样可以提高 IP 的利用率，适合大规模集群。

---

---
# 分配流程
