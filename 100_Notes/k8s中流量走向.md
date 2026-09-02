---
tags:
  - 运维/K8S/CNI插件/Calico
---

> [!note] k8s网络模型核心原则
> **每个 Pod 都拥有一个独立的 IP 地址，且所有 Pod 之间都可以不通过 NAT 直接通信**

```text
[用户浏览器] 
       ↓ (DNS 解析域名)
[云厂商负载均衡器 (四层 LB)]
       ↓ (分发流量)
[工作节点宿主机网卡]
       ↓ (直通或经 NodePort)
[Ingress Controller (如 Nginx Pod，七层路由)]
       ↓ (跳过 Service VIP，直接选择 Pod IP)
[CNI 覆盖网络隧道 (如 VXLAN 封装)]
       ↓ (跨物理网络传输)
[目标节点内核解包]
       ↓ (通过 veth 网桥)
[目标 Pod 的容器]
```
---
# 东西向流量
> 集群内部的横向通信
## pod内部容器间通信
- **路径**：直接通过 `localhost`（或 `127.0.0.1`）访问。
- **原理**：同一个 Pod 内的所有容器共享同一个 **Network Namespace（网络命名空间）**。它们看到的是同一块虚拟网卡，通信不需要走物理网络，直接在本地内存和回环接口（lo）完成。

## 同节点pod间通信
### overlay模式
- **路径**：`Pod A` -> `veth 对` -> `虚拟网桥 (cni0/docker0)` -> `veth 对` -> `Pod B`。
- **详细过程**：
    1. Pod A 发出请求，数据包通过其内部的 `eth0` 网卡外发。
    2. 每一个 Pod 在宿主机上都有一个对应的虚拟网卡（称为 **veth pair**），一端在 Pod 空间，一端在宿主机根空间（Root Netns）。
    3. 数据包通过 veth 隧道到达宿主机，并被注入到本地的虚拟网桥（如 `cni0`）。
    4. 网桥通过 MAC 地址表发现目标 Pod B 也在当前节点，于是直接将数据包通过 Pod B 的 veth 转发到 Pod B 内部。
### BGP模式
- **路径：Pod A (eth0) -> veth 对 (caliA) -> 宿主机内核路由表 -> veth 对 (caliB) -> Pod B (eth0)**
- 详细过程:
	1. Pod A 发出请求

		- Pod A 内的应用向 Pod B 的 IP（假设是 `10.244.1.6`）发送数据包。
		- Pod A 查看自己的内部路由表，发现没有明细路由，于是流量走向默认网关（通常是 `169.254.1.1`，这是一个专门的本地链路保留 IP）。
		- 数据包通过 Pod A 内部的 `eth0` 网卡发送出去。

	2. 穿过 veth pair 到达宿主机
	
		- 数据包通过 veth 隧道，直接“流”到了宿主机根网络空间（Root Netns）中对应的虚拟网卡 **`caliA`** 上。

	3. 宿主机内核路由表接管（核心差异点！）
	
		- 此时，数据包来到了宿主机内核。因为 **没有网桥** 接收它，宿主机内核的网络栈会直接去查找**宿主机的系统路由表（Routing Table）**。
		- Calico 早就通过其核心组件（Felix）在宿主机上写好了所有本地 Pod 的明细路由。内核在路由表中找到了这样一条规则：
    
		    ```
		    10.244.1.6 dev caliB scope link
		    ```
		    _这意味着：去往 `10.244.1.6`（Pod B）的包，请直接交给 `caliB` 这张网卡。_

	1. 流入 Pod B
	
		- 宿主机内核根据路由指示，把数据包转发给 **`caliB`** 网卡。
		- 数据包再次穿过 veth 隧道，直接进入 Pod B 的网络空间，最终到达 Pod B 的 `eth0` 网卡，由 Pod B 内部的应用接收。

> [!warning] 问题
> Pod A 发包前需要知道网关的 MAC 地址，但 Calico 的网关 IP（169.254.1.1）在宿主机上根本不存在，Pod A 是怎么拿到 MAC 地址的？

> [!success]- 回答
> ==因为 Calico 在宿主机端的 `caliXXXX` 网卡上开启了 **Proxy ARP（ARP 代理）** 功能。==
> - 当 Pod A 在外发数据包前广播询问：“谁是 `169.254.1.1`？请把 MAC 地址给我”时；
> - 宿主机端的 `caliA` 网卡收到这个请求，会**假装自己就是这个网关**，直接把自己的 MAC 地址回复给 Pod A。
> - 这样，Pod A 就能顺利把包发给宿主机，剩下的全靠宿主机查路由表转发。

## 跨节点同网段pod间通信
- **路径**：`Pod A` -> `veth 对` -> `Node A 宿主机路由表` -> `Node A 物理网卡` -> **物理交换机（二层直接转发）** -> `Node B 物理网卡` -> `Node B 宿主机路由表` -> `veth 对` -> `Pod B`。
- **详细过程**：
    1. Pod A 发出请求，目标 IP 为 Pod B (10.244.2.8)。Pod A 内部只有一条默认路由指向自己的网关（宿主机），流量通过 `eth0` 送往宿主机的 **veth pair** 端口。
    2. 流量到达 Node A 根命名空间。由于 Calico 没建网桥，Node A 内核直接查看**系统路由表**。
    3. [[Calico]] 的 BGP 守护进程（Felix/Bird）此前已通过 BGP 协议得知：_10.244.2.0/24 网段在 Node B 上_。因此路由表写着：`10.244.2.0/24 via Node_B_物理_IP dev eth0`。
    4. Node A 内核将原始数据包直接从物理网卡发往 Node B 的物理 IP，期间**不做任何封包（Overlay）或 NAT 转换**。
    5. 物理交换机根据 MAC 地址，将纯净的 IP 包直接送达 Node B 的物理网卡。
    6. Node B 收到包后查看自己的路由表，发现目标 10.244.2.8 就在本地，对应网卡是 `caliXXXX`（Pod B 的 veth 端）。
    7. 内核直接将数据包通过该 veth 隧道注入到 Pod B 内部。

## 跨节点跨网段pod间通信
- **路径**：`Pod A` -> `veth 对` -> `Node A 宿主机路由表` -> `tunl0 虚拟网卡（内核封包）` -> `Node A 物理网卡` -> **跨子网物理路由器（三层路由）** -> `Node B 物理网卡` -> `tunl0 虚拟网卡（内核解包）` -> `Node B 宿主机路由表` -> `veth 对` -> `Pod B`。
- **详细过程**：
    1. Pod A 发出请求，目标 IP 为 Pod B (10.244.2.8)，流量通过其内部 `eth0` 穿过 veth 隧道到达 Node A 宿主机根命名空间。
    2. Node A 内核查看系统路由表。由于 Node A 和 Node B 不在同一个子网，无法二层直达，Calico 写入的路由表指示：_要去往 10.244.2.8，必须将流量送往 `tunl0` 虚拟接口_（`10.244.2.0/24 via Node_B_物理_IP dev tunl0 proto bird onlink`）。
    3. 流量进入 `tunl0` 驱动后，Linux 内核的 IPIP 模块执行**隧道封装**：在原始 Pod 数据包外面，强行套上一层新宿主机 IP 头部。此时，外层源 IP 为 Node A 物理 IP，外层目的 IP 为 Node B 物理 IP。
    4. 封装后的“大包裹”通过 Node A 的物理网卡发送出去。
    5. 沿途的物理三层路由器只检查外层的宿主机 IP，认为这是一个普通的宿主机之间通信的包，顺利将其跨网段路由分发到 Node B。
    6. Node B 的物理网卡接收到该包，内核发现其协议号为 4（IP-in-IP），于是将其送往本地的 `tunl0` 设备进行**剥离（解封装）**，还原出里面原本的 Pod A 到 Pod B 的数据包。
    7. 解包后的原始数据包再次触发 Node B 的路由查找，匹配到本地路由规则，通过对应的 `caliXXXX` 网卡（veth 对）精准投递进 Pod B 内部。

## 通过Service访问
> [!tip]
> 由于 Pod 的生命周期短暂、IP 经常变动，K8s 使用 **Service（四层负载均衡）** 来提供固定 IP（ClusterIP）

- **路径**：`Pod` -> `Service VIP` -> **kube-proxy（iptables/IPVS 规则拦截修改）** -> `实际后端 Pod`。
- **详细过程**：
    1. Pod 发起请求，目标 IP 是 Service 的虚拟 IP（**ClusterIP**）。
    2. 请求刚离开 Pod 到达宿主机时，会被宿主机内核的 **`kube-proxy` 维护的规则（IPVS 或 iptables）** 强行拦截。
    3. 内核在此时执行 **DNAT（目标地址转换）**：根据负载均衡算法，把目标 IP 从“Service VIP”修改为“某个具体的后端真实 Pod IP”。
    4. 随后，数据包带着修改后的真实 Pod IP，重复上述【同节点】或【跨节点】的 Pod 间通信流程。

## pod到宿主机
- **路径**：`Pod 内部程序` -> `Pod 内部路由表` -> `eth0（veth的一端）` -> `宿主机的 caliXXXX（veth的另一端）` -> `宿主机内核网络栈（查路由表）`。

**详细过程：**

1. **Pod 发包**：Pod 内的应用发起请求（比如访问宿主机 IP，或者访问外网）。
2. **查 Pod 内路由**：Pod 内部的内核网络栈一看目标 IP，发现不在自己本地，于是走默认路由（Default Route）。Pod 内的默认路由通常指向一个虚拟的网关 IP（如 `169.254.1.1`）。
3. **ARP 请求（关键插曲）**：Pod 需要知道网关的 MAC 地址，于是发出 ARP 请求。
4. **宿主机响应（Proxy ARP）**：开启了 Calico 后，宿主机端的 `caliXXXX` 网卡配置了 `proxy_arp`。宿主机直接**假装自己是那个网关**，用自己的 MAC 地址回复给 Pod。
5. **飞跃边界**：Pod 拿到 MAC 地址后，把数据包从 `eth0` 发出。由于 veth pair 的特性，这个包**不经过任何隧道包装，直接肉身出现在宿主机**的 `caliXXXX` 网卡上。
6. **宿主机接管**：此时数据包已经成功肉身到达宿主机。宿主机内核的路由引擎介入，开始查看宿主机自己的 `ip route` 路由表，决定是转给本地的其他 Pod，还是通过宿主机的物理网卡发到外网。

---
# 南北向流量
> 集群外部接入的纵向通信

## NodePort模式(四层)
- **路径**：`外部客户端` -> `任意 Node IP:NodePort` -> `kube-proxy (DNAT 转换)` -> `Pod`。
- **详细过程**：
    1. K8s 在集群的所有节点（Node）上开放一个相同的端口（如 32000）。
    2. 用户请求访问 `http://<任意Node_IP>:32000`。
    3. 节点收到流量后，内核中的 `kube-proxy` 规则立刻介入，通过 DNAT 将目标 IP 和端口转换为具体的 **Backend Pod IP**。
    4. 如果选中的 Pod 在其他节点，该节点还会额外做一次 **SNAT（源地址转换）**，把源 IP 改为当前节点 IP，以确保响应数据包能原路返回。最后将流量跨节点转发给 Pod。

## LoadBalancer模式(四层)
- **路径**：`外部客户端` -> `云厂商公网 LB (如 AWS ELB/阿里云 SLB)` -> `多个 Node 的 NodePort` -> `kube-proxy` -> `Pod`。
- **详细过程**：
    1. 用户访问公网 IP 或域名，流量首先到达云厂商提供的**负载均衡器（[[云LB]]）**。
    2. 云厂商的 LB 充当了流量第一站，它把流量分发到 K8s 集群的各个工作节点（Node）的 NodePort 端口上。
    3. 节点收到流量后，再次通过内部的 `kube-proxy` 路由到最终的 Pod。

## Ingress / Gateway API 模式(七层)
- **路径**：`外部客户端` -> `域名解析` -> `负载均衡器 (LB)` -> **Ingress Controller (如 Nginx Ingress)** -> `Pod`。
- **详细过程**：
    1. 用户在浏览器输入域名 `://example.com`。
    2. 流量通过公网 LB，直接打到集群内部部署的 **Ingress Controller**（本质上是一个高性能的 Nginx/Envoy Pod，它通过 HostNetwork 或 LoadBalancer 暴露在最外端）。
    3. Ingress Controller 收到 HTTP 请求后，读取其配置的路由规则（**Ingress 资源对象**）：
        - 发现 `://example.com` 应该对应 `order-service`。
    4. **关键点**：Ingress Controller 并**不会**把流量发给 Service VIP 让 `kube-proxy` 再转一次，而是通过 K8s API 直接监听该 Service 背后对应的 **Endpoints/EndpointSlice（即直接获取所有后端 Pod 的真实 IP 列表）**。
    5. Ingress Controller 在应用层自己做负载均衡，**直接将 HTTP 请求转发给选中的后端 Pod IP**，绕过了四层虚拟转发，效率极高。

[^1]: 
