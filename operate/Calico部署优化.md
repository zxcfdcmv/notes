---
tags:
  - 运维
  - CNI插件/Calico
---

# 部署
## 1. 添加 Calico 官方 Helm 仓库
```sh
helm repo add projectcalico https://tigera.io
helm repo update
```
## 2. 创建一个自定义的 `values.yaml`
```yaml
# calico-values.yaml
installation:
  enabled: true
  kubernetesProvider: Kubernetes
  cni:
    type: Calico
  calicoNetwork:
    bgp: Enabled                       # 开启 BGP 路由模式
    ipPools:
      - cidr: 10.244.0.0/16            # 集群 Pod 网段
        blockSize: 26                  # 每个节点分配的子网大小
        encapsulation: None            # 纯 BGP 模式设为 None（若要VXLAN则填VXLAN）
        natOutgoing: Enabled
```

## 3. 安装
```sh
helm install calico projectcalico/tigera-operator \
  --version v3.28.0 \
  --namespace tigera-operator \
  --create-namespace \
  -f calico-values.yaml
```

# 优化
> [!tip]
> 路由反射器(Route Reflector)、CrossSubnet
```yaml
# ==========================================
# 1. 禁用默认的 Full-Mesh（全网互联）模式
# ==========================================
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  # 当节点数超过 50-100 时，必须关闭此项，改用下文的 Route Reflector
  nodeToNodeMeshEnabled: false
  
  # 性能调优：显式指定 BGP 的自治系统号（AS Number），自建机房常用 64512
  asNumber: 64512
  
  # 日志级别，生产环境建议 Info 即可，排查故障时可临时改为 Debug
  logSeverityScreen: Info

---
# ==========================================
# 2. IP 池配置（支持 CrossSubnet 混合路由模式）
# ==========================================
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  blockSize: 26                   # 每个节点分 64 个 IP
  
  # 【生产推荐优化】
  # 如果你的 K8s 节点处于同一个二层交换机下（同网段），设为 None 性能最高。
  # 如果你的 K8s 节点跨越了不同的机房/三层网段，设为 CrossSubnet：
  # 同网段走纯 BGP 路由（无损），跨网段自动降级为 IPIP 隧道封装，兼顾性能与兼容性。
  encapsulation: CrossSubnet
  
  natOutgoing: true               # 允许 Pod 访问外网（SNAT）
  nodeSelector: all()             # 应用于集群所有节点

---
# ==========================================
# 3. 声明哪些节点作为 Route Reflector（路由反射器）
# ==========================================
# 提示：你需要在生产中选择 2-3 台不跑核心业务、位置关键的 Master 或工具节点
# 并在 K8s 节点上打上标签：kubectl label node node-master-01 route-reflector=true
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: node-master-01            # 必须和你的 K8s 节点名字（hostname）完全一致
spec:
  bgp:
    # 将此节点指定为群组内的路由反射器，并分配一个集群内唯一的 Cluster ID
    routeReflectorClusterID: 224.0.0.1
    # 强制指定该节点用于 BGP 通信的物理 IP（防止多网卡机器走错接口）
    ipv4Address: 192.168.10.11/24

---
# ==========================================
# 4. 配置 BGP Peer：让普通 Worker 节点连上 RR 节点
# ==========================================
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: peer-workers-to-rr
spec:
  # 筛选普通节点：没有被打上 route-reflector 标签的节点
  nodeSelector: "!has(route-reflector)"
  
  # 筛选目标邻居：被打上了 route-reflector 标签的节点
  peerSelector: "route-reflector == 'true'"

---
# ==========================================
# 5. 配置 BGP Peer：让多个 RR 节点之间互相建立连接
# ==========================================
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: peer-rr-to-rr
spec:
  # 筛选反射器节点：自己跟自己这个组内的其他反射器同步数据
  nodeSelector: "route-reflector == 'true'"
  peerSelector: "route-reflector == 'true'"
```