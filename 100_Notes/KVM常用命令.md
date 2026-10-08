---
tags:
    - 运维/KVM
---
---
# 常用
```sh
virsh list --all
virsh start vm01
virsh shutdown vm01
virsh reboot vm01
virsh destroy vm01
virsh undefine vm01
```

查看虚拟机信息:
```sh
virsh dominfo vm01
virsh domblklist vm01
virsh domiflist vm01
```

进入虚拟机控制台:
```sh
virsh console vm01
```

# 快照
创建快照
```sh
virsh snapshot-create-as vm01 snap01
```

查看快照
```sh
virsh snapshot-list vm01
```

恢复快照
```sh
virsh snapshot-revert vm01 snap01
```

克隆
```sh
virt-clone \
  --original vm01 \
  --name vm02 \
  --file /var/lib/libvirt/images/vm02.qcow2
```
克隆后应注意：

- 修改主机名。
- 修改网卡 MAC 地址。
- 清理 SSH host key。
- 检查 Linux machine-id。
- 检查 Windows SID。
- 确认 IP 地址不会冲突。