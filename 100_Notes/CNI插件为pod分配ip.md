---
tags:
  - 运维/K8S/CNI插件/Calico
---

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
# 分配流程

```
[kubelet] 
   │ (1) 调用 CRI 创建容器 (创建 Network Namespace)
   ▼
[Container Runtime (如 Containerd)]
   │ (2) 读取 CNI 配置，调用 CNI 插件
   ▼
[CNI 插件 (如 Calico/Flannel 二进制文件)]
   │ (3) 调用 IPAM 插件获取可用 IP
   ├──────────────────────────────► [IPAM 插件] (从本地网段/CRD 中挑一个 IP)
   │ (4) 创建 veth pair，一端挂宿主机，一端塞入容器
   │ (5) 在容器内配置 IP、网关和路由
   ▼
[Pod 成功获取 IP 并联网]
```

## 详细步骤
1. **创建网络命名空间**：  
    `kubelet` 调度 Pod 到节点后，通过 CRI（容器运行时接口）通知容器运行时（如 Containerd/Docker）创建 Pod 的基础设施容器（Pause 容器）。此时，容器运行时会为该 Pod 创建一个独立的**网络命名空间（Network Namespace）**。
2. **触发 CNI 插件**：  
    容器运行时去读取节点上的 CNI 配置文件（通常在 `/etc/cni/net.d/`），然后调用标准的 CNI 二进制文件（通常在 `/opt/cni/bin/`），并传入操作指令 `ADD` 以及该 Pod 的网络命名空间路径。
3. **获取 IP 地址**：  
    CNI 插件收到 `ADD` 请求后，调用 **IPAM 插件**。IPAM 根据上文提到的机制（如查找本地已分配的子网数据库），计算出一个当前未被占用的 IP 地址，并将其返回给 CNI 插件。
4. **管道搭建（veth pair）**：  
    CNI 插件在宿主机上创建一对虚拟网卡（**veth pair**，就像一根虚拟网线，数据从一端进，必从另一端出）。
    - 将 veth pair 的**一端**留在宿主机上（例如命名为 `caliXXXX` 或 `vethXXXX`），并挂载到宿主机的网桥或路由表中。
    - 将 veth pair 的**另一端**“塞入” Pod 的网络命名空间内，并重命名为 `eth0`。
5. **配置网络与绑定**：  
    CNI 插件在 Pod 的网络命名空间内，将刚刚从 IPAM 获取到的 **IP 地址绑定到 `eth0` 网卡上**，并配置默认网关（指向宿主机端的虚拟网卡或网桥）。
6. **状态上报**：  
    CNI 插件将分配好的 IP 信息以 JSON 格式返回给容器运行时，容器运行时再上报给 `kubelet`。最后，`kubelet` 将该 IP 更新到 Pod 的 `status.podIP` 中，同步到 etcd，此时通过 `kubectl get pod -o wide` 就能看到这个 IP 了。

---
# 对比
|CNI 插件|默认 IPAM 机制|IP 存储在哪里？|底层网络原理|
|---|---|---|---|
|**Flannel**|host-local|存储在宿主机的本地文件/本地缓存|**Overlay 隧道**（默认 VXLAN），通过封包实现跨节点通信。|
|**Calico**|calico-ipam|存储在 **etcd**（作为 Kubernetes CRD 存储）|**三层路由**（BGP / IPIP）。Pod IP 也是一个真正的路由条目，节点间直接通过路由表转发。|
|**Cilium**|cilium-ipam|存储在 **etcd** / 动态分配|**eBPF 技术**。在内核层直接绕过传统的 iptables，通过 eBPF 路由和分配 IP，性能最高。|
