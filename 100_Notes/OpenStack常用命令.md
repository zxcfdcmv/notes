---
tags:
    - 运维/OpenStack
---

> [!note] openstack <对象> <动作>
---
# 核心命令
## 身份认证与环境检查（Keystone）
在执行任何命令前，必须先加载环境变量文件（通常是 `admin-openrc.sh`）：
```sh
source admin-openrc.sh         # 导入凭证，获取 Token
openstack service list         # 查看所有服务是否正常在线
openstack endpoint list        # 查看所有服务的 API URL 地址是否正确
```

## 计算与虚拟机管理（Nova）
用命令看虚拟机和排查物理机
```sh
openstack server list --all-projects   # 查看所有项目下的所有虚拟机（运维最常用）
openstack server show <VM_ID>          # 查看某台虚拟机的详细信息（能看到卡在什么状态）
openstack hypervisor list              # 查看所有物理计算节点及其资源使用情况
openstack compute service list         # 查看计算节点上 nova-compute 服务状态是否为 up
```

## 网络运维管理（Neutron）
断网、IP 冲突时常用
```sh
openstack network list                 # 查看所有网络（VPC）
openstack subnet list                  # 查看所有子网及网段
openstack port list --router <ID>      # 查看某个路由器上连接的所有虚拟网卡接口
openstack security group list          # 查看安全组（防火墙规则）
```

## 镜像与存储管理（Glance & Cinder）
```sh
openstack image list                   # 查看系统镜像列表
openstack volume list                  # 查看所有的云硬盘（块存储）及挂载状态（in-use/available）
openstack volume show <Volume_ID>      # 查看某块云硬盘属于哪个后端存储集群
```

# 常用
