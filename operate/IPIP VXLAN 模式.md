---
tags:
  - 运维
  - CNI插件/Calico
---
> [!tip] 本质
> - Overlay（叠加/隧道）网络技术
> - 把 Pod 发出的、物理网络不认识的数据包，在宿主机端套上一层物理网络认识的“外壳”，传送到目的地后再剥掉外壳
# IPIP（IP-in-IP）
> [!note]
> 三层隧道技术(网络层封包)
> 把一个 IP 报文直接封装在另一个 IP 报文里面

---
## 封包结构

当 Pod A（`10.244.1.5`）访问 Pod B（`10.244.2.6`）时，数据包在出宿主机前会被套上一层新的 IP 头部：

- **外层 IP 报文头**：源 IP = 宿主机 A 的物理 IP，目的 IP = 宿主机 B 的物理 IP，**Protocol(协议) 号固定为 4**（代表内容是 IPIP 包）。
- **内层 IP 报文头**：源 IP = Pod A 的 IP，目的 IP = Pod B 的 IP。
- **Payload**：实际的业务数据（如 TCP/UDP 内容）。

## 通信路径

```
Pod A -> 宿主机 A 路由表 -> tunl0 设备 (执行 IPIP 封包) -> 物理网络三层路由 -> 宿主机 B -> tunl0 设备 (解包) -> Pod B
```

- **特点**：Calico 使用 IPIP 模式时，宿主机上会有一个名为 **`tunl0`** 的虚拟隧道设备。数据包根据路由表指引进入 `tunl0`，内核自动完成外层 IP 头的剥离或组装。

---
# VXLAN（Virtual Extensible LAN）
> [!note]
> 四层隧道技术（MAC-in-UDP）

## 封包结构

VXLAN 封包比 IPIP 要沉重和复杂得多，因为它不仅封装了 IP，还封装了二层以太网的 MAC 地址。它把原始数据包包装成一个宿主机间的 **UDP 报文**：

- **外层 IP/MAC 头**：用于宿主机之间在物理网络中的标准路由与交换。
- **外层 UDP 头**：**目的端口固定为 4789**（VXLAN 官方标准端口）。
- **VXLAN 头部**：包含一个核心的 **VNI（VXLAN Network Identifier，24位）**，相当于虚拟化网络里的 VLAN ID，用于隔离不同的容器网络。
- **内层 MAC/IP 头**：Pod 间通信的原始二层和三层头部。

## 通信路径

```
Pod A -> cni0 网桥 -> flannel.1 / vxlan.calico 设备 (VTEP 执行 VXLAN 封包) -> 物理网络 UDP 转发 -> 宿主机 B -> 隧道设备 (解包) -> Pod B
```

- **特点**：负责封包和解包的组件叫做 **VTEP（VXLAN Tunnel End Point）**。在 K8s 中，宿主机上的 `flannel.1` 或 `vxlan.calico` 虚拟网卡就是软件实现的 VTEP。

---