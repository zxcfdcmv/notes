---
tags:
    - 运维/K8S
---

> [!tip]
> **利用容器化技术来部署 K8s 的核心组件**（除了 `kubelet` 和容器运行时在宿主机运行，`kube-apiserver`、`etcd` 等都作为 Pod 运行）

---
# 环境准备（所有节点）
## 1 关闭swap分区
K8s 为了保证 Pod 的性能和稳定性，不允许使用虚拟内存。
```sh
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```
## 2 关闭防火墙与 SELinux
避免各种端口被拦截。
```sh
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
```
## 3 修改内核参数（开启桥接网络流量转发）
让 Linux 的 iptables 能够看到桥接网络（如 Calico/Flannel）的流量。
```sh
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```
# 安装容器运行时（所有节点）
1. **安装 containerd**：可以使用阿里云等镜像源通过 `yum` 或 `apt` 安装 `containerd.io`。
2. **生成并修改配置文件**：
    
    ```sh
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml
    ```
    
3. **配置 SystemdCgroup（非常关键）**：  
    打开 `/etc/containerd/config.toml`，找到 `SystemdCgroup = false`，**务必改成 `true`**。如果不改，后续 kubelet 会因为 Cgroup 驱动不一致而频繁重启。
4. **替换沙箱（Pause）镜像源**：  
    国内无法直接下载 `registry.k8s.io/pause`。在 `config.toml` 中，将 `sandbox_image` 修改为阿里云镜像：  
    `sandbox_image = "://aliyuncs.com"`
5. **重启服务**：`systemctl restart containerd && systemctl enable containerd`

# 安装 K8s 基础组件（所有节点）
配置好阿里云的 Kubernetes 源后，安装三个核心工具：

- **`kubelet`**：运行在每个节点上，负责管理容器生命周期。
- **`kubeadm`**：集群初始化和扩容的命令行工具。
- **`kubectl`**：运维人员使用的命令行工具。
```
# 以 CentOS/RHEL 顺应阿里云源为例安装指定版本（如 1.28.2）
yum install -y kubelet-1.28.2 kubeadm-1.28.2 kubectl-1.28.2
systemctl enable kubelet # 注意：此时不需要手动 start，因为还没有集群配置，启动会报错
```

# Master 节点初始化（仅在 Master 节点执行）
```sh
kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --image-repository=://aliyuncs.com \
  --kubernetes-version=v1.28.2 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

- `--apiserver-advertise-address`: 填你 Master 节点的内网 IP。
- `--image-repository`: 切换为**阿里云镜像源**，完美解决国内下载高墙问题。
- `--pod-network-cidr`: Pod 的网段。如果后续准备用 **Flannel** 网络插件，必须指定为 `10.244.0.0/16`；如果是 **Calico**，通常是 `192.168.0.0/16`。

初始化成功后，终端会输出两部分重要信息：
- **第1部分：配置 kubectl 权限（在 Master 上执行）**
    
    ```
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    
- **第2部分：保存 Join 命令（用于 Node 节点加入）**  
    类似这样的一行命令，先复制保存下来：
    
    ```
    kubeadm join 192.168.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx
    ```

# 部署网络插件CNI（仅在 Master 节点执行）
> [!warning]
> 刚初始化完时，执行 `kubectl get nodes` 会发现 Master 节点的状态是 **`NotReady`**。这是因为还没有安装集群网络插件，组件之间无法跨节点通信。

安装Calico
```sh
kubectl apply -f https://githubusercontent.com
```
等几分钟，所有的 Calico Pod 变成 `Running` 状态后，Master 节点就会变成 **`Ready`**。

# 工作节点加入集群（在所有 Node 节点执行）
把刚才在第四步保存的 `kubeadm join` 命令，直接在所有的 Node 节点上粘贴并运行：
```sh
kubeadm join 192.168.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx
```
运行完成后，在 Master 节点执行 `kubectl get nodes`，就能看到所有的 Node 节点都成功加入且变成了 **`Ready`** 状态。至此，集群部署完毕。

# 问题
> [!warning] 问题1
> Token 过期了怎么办？怎么让新节点加入?

> [!success]- 回答1
`kubeadm init` 生成的 Token 默认只有 **24 小时** 有效期。如果以后要扩容节点，需要在 Master 上执行 `kubeadm token create --print-join-command`，它会重新动态生成一条带新 Token 的 join 命令。

> [!warning] 问题2
> 什么是 Cgroup Driver 冲突？你怎么解决的?

> [!success]- 回答2
> 这是部署时最容易踩的坑。Linux 有两种 Cgroup 管理器：`cgroupfs` 和 `systemd`。K8s 官方强烈推荐在生产中全部统一使用 **`systemd`**。我们需要确保 **容器运行时（containerd）** 的 `SystemdCgroup = true`，同时 **kubelet** 也会默认使用 systemd。如果两者不一致，kubelet 就会崩溃

> [!warning] 问题3 
> kubeadm 部署的集群，Master 节点高可用（HA）怎么做?

> [!success]- 回答3
> 不能直接用上述的单机 init。如果是多 Master 高可用，需要：
> - 提前搭建 **Keepalived + HAProxy**（或者使用云厂商的内网负载均衡器 SLB），挂载一个虚拟 VIP（比如 `192.168.1.100:6443`）。
> - 在 `kubeadm init` 时，加上 `--control-plane-endpoint "192.168.1.100:6443"` 参数。
> - 其他 Master 节点加入时，使用 `kubeadm join ... --control-plane` 命令加入，从而实现控制平面的多活高可用。