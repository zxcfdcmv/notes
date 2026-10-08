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

---
# 磁盘
|格式|特点|
|---|---|
|raw|性能简单直接，功能少|
|qcow2|支持快照、稀疏分配、压缩|
|LVM volume|性能好，适合生产环境|
|Ceph RBD|适合分布式存储|
|NFS 文件|部署方便，但性能取决于网络|

创建 qcow2 磁盘：

```sh
qemu-img create -f qcow2 vm01.qcow2 40G
```


查看磁盘信息：
```sh
qemu-img info vm01.qcow2
```

注意：qcow2 的快照和写时复制功能很方便，但在高 I/O 场景下，性能通常不如 raw 或块设备。

---
# 网络模式
## NAT
> 虚拟机通过宿主机访问外部网络

```text
虚拟机 → 宿主机 → 外部网络
```
优点是配置简单；缺点是外部设备通常不能直接访问虚拟机。

## Bridge
> 虚拟机直接接入物理局域网

```text
虚拟机 ─┐
虚拟机 ─┼── Linux Bridge ── 物理网卡 ── 局域网
宿主机 ─┘
```
适合服务器和生产环境

## Host-only
> 虚拟机只能和宿主机或同一虚拟网络中的其他虚拟机通信，不能直接访问外网。

## macvtap
> 可以让虚拟机直接使用物理网卡的二层连接，但宿主机与虚拟机之间的通信可能受限。