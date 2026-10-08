---
tags:
    - 运维/KVM
    - 问题
---
---

# `/dev/kvm` 不存在
检查：
```sh
ls -l /dev/kvm
```
如果不存在，可以检查：
```sh
lsmod | grep kvm
egrep -o 'vmx|svm' /proc/cpuinfo | sort -u
```

可能原因：

- BIOS/UEFI 未开启虚拟化。
- 虚拟化模块未加载。
- 当前系统运行在不支持嵌套虚拟化的虚拟机中。
- 宿主机禁用了硬件虚拟化。

# 虚拟机很慢
重点检查：
```sh
virsh domblklist vm01
virsh domiflist vm01
```
优化方向：

- 确认使用 KVM 加速，而不是纯 QEMU 模拟。
- 磁盘使用 virtio。
- 网卡使用 virtio。
- 避免过度使用 qcow2 快照链。
- 生产环境考虑 raw、LVM 或 Ceph。
- 根据场景使用 HugePages。
- 检查 CPU overcommit。
- 检查 NUMA 和 CPU 绑核。
- 确认宿主机没有严重 I/O 等待。

# 网络不通
检查虚拟网络：
```sh
virsh net-list --all
ip addr
bridge link
```
检查虚拟机网卡：
```sh
virsh domiflist vm01
```

常见原因：

- `default` 网络未启动。
- 虚拟机没有 DHCP 地址。
- 防火墙阻止了转发。
- Bridge 配置错误。
- 物理交换机端口未允许相应 VLAN。
- 网卡名称或 NetworkManager 配置不一致。

# 虚拟机无法在线迁移
常见原因：

- 两台主机 CPU 型号或特性不兼容。
- 虚拟磁盘不是共享存储。
- 目标主机缺少相同的虚拟网络。
- 虚拟机使用了直通 PCI 设备。
- 两台主机的 libvirt/QEMU 配置差异过大。