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
