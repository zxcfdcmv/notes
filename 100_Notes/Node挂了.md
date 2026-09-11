---
tags:
    - 运维/K8S
    - 问题
---

> [!note] 本质
> **Kubelet 守护进程无法正常工作**，或者 **Master 节点（kube-controller-manager）在规定时间内（默认 40s）没有收到该节点的心跳上报**
---
# 原因
## 资源耗尽与系统保护
- ==**内存耗尽引发 OOM Killer**==： 节点上运行的 Pod 或原生进程占用了全部内存，触发 Linux 内核的 OOM（Out Of Memory）机制。如果 **Kubelet 或系统关键进程（如 docker/containerd、systemd）被内核误杀**，节点就会失联。
- ==**磁盘空间爆满（DiskPressure）==：** 节点根分区或容器运行时存储目录空间被日志、未清理的镜像填满。Kubelet 无法写入数据，保护性停止或崩溃。
- **CPU 发生死锁/软锁定（Soft Lockup）：** 某个进程在内核态长时间占用 CPU 不释放（常见于高并发或内核 Bug），导致 Kubelet 得不到 CPU 时间片，无法发送心跳。

## 底层基础设施与硬件故障
- **物理机/虚拟机宕机：** 机房断电、服务器主板故障、CPU 或内存硬件损坏。在云环境中，底层宿主机可能发生故障导致虚拟机被强制关机。
- **网络断连（Network Partition）：** 节点本身运行正常，但由于交换机故障、网线断开、防火墙规则误改或大流量导致网卡丢包，导致**节点与 Master 节点之间的通信中断**。

## 操作系统与内核层异常
- **内核崩溃（Kernel Panic）：** 遇到严重的系统 Bug 或硬件不兼容（如不稳定的 GPU 驱动），内核直接挂起。
- ==**容器运行时（Container Runtime）崩溃==：** Docker 或 Containerd 因为某些异常死锁、内存泄漏或 Bug 停止响应，Kubelet 无法通过 CRI 与其通信，导致 PLEGs（Pod 生命周期事件生成器）健康检查超时。
- **PID 资源耗尽：** 某些程序发生进程泄漏，疯狂创建子进程，导致节点 PID 达到系统上限（`kernel.pid_max`），Kubelet 无法再创建新的线程

## Kubelet 组件自身问题
- **证书过期：** Kubelet 与 API Server 通信所需的 Kubelet client 证书过期且未自动轮转，导致 API Server 拒绝其心跳。
- **配置错误或升级失败：** 错误修改了 `kubelet.conf` 配置文件，或者在集群升级时该节点出现版本冲突、污点异常。

