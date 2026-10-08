---
tags:
    - 运维/OpenStack
    - 问题
---
---
# 创建虚拟机失败，状态变成了 ERROR
> [!note] 可能原因
> - **计算节点资源超卖严重：** 调度器（Scheduler）认为还有资源，但发到具体的物理机上时，物理机真实的内存或 CPU 已经被占满，底层的 KVM 无法拉起进程。
> - **网络分配超时（Neutron 报错）：** 消息队列太拥堵，导致 Nova-Compute 向 Neutron 申请 IP 和 Port 时超时。

1. **查 ID 和状态：** 先用 `openstack server show <VM_ID>` 查看该虚拟机的详细信息，找到报错信息（Fault）和具体的 `Request ID`。
2. **看控制节点日志：** 如果是调度阶段失败，去控制节点看 `/var/log/nova/nova-scheduler.log`。
3. **看计算节点日志：** 如果已经分配到了物理机但拉起失败，去对应物理机查看 `/var/log/nova/nova-compute.log`。通常可以通过全局唯一的 `Request ID` 在日志里搜索，判断是镜像下载超时、网络端口打不开，还是底层的 KVM/Libvirt 报错。

# 用户反映他的虚拟机登不上了，如何排查
> [!tip] 先看状态，再网络，后日志

1. 先用 `openstack server show` 查看虚拟机的平台状态。如果是 `ERROR`，说明底层拉起失败，直接通过 `virsh list` 去对应的计算节点看 KVM 进程还在不在。
2. 如果状态是 `ACTIVE`，但连不上，说明是网络问题。我会去网络节点，用 `ping` 虚拟机的内网 IP，并用 `ovs-vsctl` 检查对应的底层虚拟网口流量，判断是安全组拦截了，还是底层大二层网络断了。
3. 如果以上都正常，我会去查看该计算节点的 `/var/log/nova/nova-compute.log`，根据报错的 `Request ID` 定位是存储超时还是网络超时，从而根本解决问题。