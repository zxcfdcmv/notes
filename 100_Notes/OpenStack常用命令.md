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


## 网络运维管理（Neutron）
## 镜像与存储管理（Glance & Cinder）