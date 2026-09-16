> [!tip]
> **利用容器化技术来部署 K8s 的核心组件**（除了 `kubelet` 和容器运行时在宿主机运行，`kube-apiserver`、`etcd` 等都作为 Pod 运行）

---
# 环境准备（所有节点）
## 1 关闭swap分区
```sh
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```
## 2 关闭防火墙与 SELinux
```sh
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
```
## 3 修改内核参数（开启桥接网络流量转发）
```sh
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```
