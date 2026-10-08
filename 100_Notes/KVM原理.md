---
tags:
    - 运维/KVM
---
> [!note]
> - `KVM` 内核模块负责 CPU、内存和虚拟机运行控制。
> - `QEMU` 负责模拟虚拟硬件和处理大部分 I/O。
> - `libvirt` 提供统一的管理接口。
> - `virsh`、`virt-manager`、`OpenStack` 等工具通过 `libvirt` 管理虚拟机。

---
# 核心组成
## KVM内核模块
**常见模块**：
```text
kvm.ko
kvm_intel.ko
kvm_amd.ko
```
**主要负责**：
- 创建和销毁虚拟机。
- 创建和调度虚拟 CPU。
- 分配和管理虚拟机内存。
- 保存、恢复虚拟 CPU 寄存器。
- 进入和退出 Guest 执行模式。
- 处理敏感指令、异常和中断。
- 将无法直接执行的操作交给 QEMU 处理。

## QEMU
> [!note] 
> - QEMU 是用户空间的虚拟机模拟器。启用 KVM 后，QEMU 通常==使用 KVM 加速 CPU 执行==，而自己主要==负责虚拟设备==
> - QEMU 通过 `/dev/kvm` 与 KVM 内核模块通信

**主要负责**：
- 虚拟磁盘控制器。
- 虚拟网卡。
- 虚拟显卡。
- USB 设备。
- 音频设备。
- BIOS 或 UEFI。
- 虚拟串口。
- 磁盘和网络 I/O。

## libvirt
> [!note] libvirt 是一套虚拟化管理 API，支持 KVM/QEMU，也可以管理 Xen、LXC 等

- `libvirtd` 或 `virtqemud`：后台管理服务。
- `virsh`：命令行工具。
- `virt-manager`：图形化管理工具。
- `virt-install`：命令行创建虚拟机工具。