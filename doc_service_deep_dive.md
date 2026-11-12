# 📌 Kubernetes Service 实战与网络底层机制深度解析

> **说明**：本文作为 [《Kubernetes 基础教程 - 第 7 篇 Service》](./doc_tutorial.md#7-使用-service) 的深度实战扩展篇。
> 基于跨公网双节点集群（Oracle ARM64 Master + AWS AMD64 Worker）真实多云混合环境，深度拆解 Service 核心访问路径、Linux 内核 iptables 负载均衡底层实现、跨公网 Flannel VXLAN 路由排障，以及自建与多云环境下 LoadBalancer 的真实落地形态。

---

## 目录
- [一、Service（ClusterIP）访问的四种方式](#一serviceclusterip访问的四种方式)
  - [1. 集群节点宿主机直接访问（最便捷，K3s / 裸机架构适用）](#1-集群节点宿主机直接访问最便捷k3s--裸机架构适用)
  - [2. 集群内部 Pod 通过 DNS 服务名访问（生产标准实践）](#2-集群内部-pod-通过-dns-服务名访问生产标准实践)
  - [3. `kubectl port-forward` 端口转发（本地开发调试利器）](#3-kubectl-port-forward-端口转发本地开发调试利器)
  - [4. 转为 NodePort / Ingress 暴露给公网外部访问（外部生产暴露）](#4-转为-nodeport--ingress-暴露给公网外部访问外部生产暴露)
- [二、iptables 负载均衡与网络底层实现原理](#二iptables-负载均衡与网络底层实现原理)
  - [1. 全链路穿透架构图](#1-全链路穿透架构图)
  - [2. 现场 iptables 规则深度拆解](#2-现场-iptables-规则深度拆解)
  - [3. 常见疑问：为什么连跑 4 次 curl 全访问到同一个 Pod？](#3-常见疑问为什么连跑-4-次-curl-全访问到同一个-pod)
- [三、跨公网多云网络互通与 Flannel VXLAN 路由排障解析](#三跨公网多云网络互通与-flannel-vxlan-路由排障解析)
  - [1. 故障现象与原因剖析](#1-故障现象与原因剖析)
  - [2. 解决方案：启用 `flannel-external-ip`](#2-解决方案启用-flannel-external-ip)
  - [3. 数据流转完整路径](#3-数据流转完整路径)
- [四、Flannel 核心原理与 CNI 技术全景](#四flannel-核心原理与-cni-技术全景)
  - [1. 什么是 Flannel？](#1-什么是-flannel)
  - [2. 为什么 Kubernetes 必须依赖 Flannel 这类 CNI？](#2-为什么-kubernetes-必须依赖-flannel-这类-cni)
  - [3. Flannel 的两大核心职责](#3-flannel-的两大核心职责)
  - [4. Flannel 的三大核心后端模式（Backend）](#4-flannel-的三大核心后端模式backend)
  - [5. 主流 CNI 横向对比（Flannel vs Calico vs Cilium）](#5-主流-cni-横向对比flannel-vs-calico-vs-cilium)
- [五、NodePort 三大端口定义与自定义端口范围](#五nodeport-三大端口定义与自定义端口范围)
  - [1. Service 三大端口定义（最易混淆的致命陷阱）](#1-service-三大端口定义最易混淆的致命陷阱)
  - [2. NodePort 端口范围限制深度剖析（为什么默认是 30000-32767？）](#2-nodeport-端口范围限制深度剖析为什么默认是-30000-32767)
  - [3. 如何解除这个限制？（改为支持 8000-9000 等自定义端口）](#3-如何解除这个限制改为支持-8000-9000-等自定义端口)
- [六、LoadBalancer 类型的两种落地形式与实现机制](#六loadbalancer-类型的两种落地形式与实现机制)
  - [形式一：K3s 内置 ServiceLB（Klipper-LB 软件负载均衡器，当前实际运行中）](#形式一k3s-内置-servicelbklipper-lb-软件负载均衡器当前实际运行中)
  - [形式二：宿主机外部反向代理（Caddy / Nginx 反代 NodePort，生产标准方案）](#形式二宿主机外部反向代理caddy--nginx-反代-nodeport生产标准方案)
- [返回主教程](#返回主教程)

---

## 一、Service（ClusterIP）访问的四种方式

在 Kubernetes 中，`ClusterIP` 是默认的 Service 类型，分配一个仅集群内部可见的虚拟 IP（VIP，如 `10.43.253.146`）。该 IP 并没有绑定在任何物理或虚拟网卡上，访问方式因场景而异：

### 1. 集群节点宿主机直接访问（最便捷，K3s / 裸机架构适用）
* **底层原理**：由于 K3s 直接运行在宿主机系统上，宿主机操作系统内核的 `iptables / nftables` 规则由 `kube-proxy` 实时维护，`10.43.0.0/16` 的 Service 虚拟 IP 会被内核的 Netfilter 钩子直接拦截，并就地做 NAT 转发至真实后端 Pod。
* **访问命令**：
  ```bash
  curl http://<ClusterIP>:<Port>
  # 现场示例：
  curl http://10.43.253.146:4001
  ```
* **适用场景**：在集群任意节点（Master 或 Worker）的终端排查服务可用性。

### 2. 集群内部 Pod 通过 DNS 服务名访问（生产标准实践）
* **底层原理**：Kubernetes 集群内置了 CoreDNS 组件，每个 Service 创建时都会自动向 CoreDNS 注册域名解析记录。业务容器相互调用时，不需要硬编码 IP。
* **域名格式规范**：
  - **同命名空间短域名**：`http://<service-name>:<port>` （例如 `http://service-hellok8s-clusterip:4001`）
  - **跨命名空间域名**：`http://<service-name>.<namespace>:<port>` （例如 `http://service-hellok8s-clusterip.default:4001`）
  - **全限定域名（FQDN）**：`http://<service-name>.<namespace>.svc.cluster.local:<port>`
* **测试方法**：
  ```bash
  kubectl run test-curl --rm -it --image=curlimages/curl -- curl http://service-hellok8s-clusterip:4001
  ```

### 3. `kubectl port-forward` 端口转发（本地开发调试利器）
* **底层原理**：利用客户端 `kubectl` 与 Kubernetes API Server 之间建立的双向加密隧道（SPDY/WebSocket），由 API Server 转发至目标 Node 的 Kubelet，进而映射到本地回环地址（`127.0.0.1`）。
* **转发命令**：
  ```bash
  # 将本地 8889 端口映射到远端集群的 Service 4001 端口
  kubectl port-forward svc/service-hellok8s-clusterip 8889:4001
  ```
* **访问方法**：在个人开发电脑或浏览器中直接访问 `http://127.0.0.1:8889`。

### 4. 转为 NodePort / Ingress 暴露给公网外部访问（外部生产暴露）
* **底层原理**：`ClusterIP` 无法从外部网络直接路由。如需从外部公网计算机或浏览器访问，应转换为 `NodePort` 或挂载 `Ingress`。
* **NodePort 端口范围配置**：
  K8s 官方默认仅允许 `30000-32767` 范围。为了匹配自定义安全组（如 `8000-9000`），我们在 Master 配置中添加了：
  ```yaml
  service-node-port-range: "8000-32767"
  ```
* **NodePort 配置示例（当前实际运行中）**：
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: service-hellok8s-nodeport
  spec:
    type: NodePort
    selector:
      app: hellok8s
    ports:
      - port: 3000
        targetPort: 80
        nodePort: 8333   # 使用开放范围内的 8333 端口
  ```
* **访问方法**：外部公网通过 `http://<任意节点公网IP>:8333` 即可直接打开服务。（详细原理解析见下文第五节）

---

## 二、iptables 负载均衡与网络底层实现原理

K8s 的 `kube-proxy` 默认采用 Linux 内核的 **iptables `nat` 表** 来实现流量转发与负载均衡。

### 1. 全链路穿透架构图

当你发起一次对 ClusterIP 的访问时，数据包经历如下三级链条处理：

```text
客户端请求 (目标: 10.43.253.146:4001)
   │
   ▼
[PREROUTING / OUTPUT 链] ── (内核拦截本地或外部进入的原始流量)
   │
   ▼
[KUBE-SERVICES 链] ──────── (匹配目标 VIP: 10.43.253.146:4001，识别为对应 Service)
   │
   ▼
[KUBE-SVC-BRULDGNIV2IQDBPU 链] ── (Service 专属链：执行 3 副本伪随机负载均衡)
   ├── (33.3% 概率) ──► 跳转 [KUBE-SEP-X7FM...] ──► DNAT 目标为 10.42.0.6:80 (Oracle 节点)
   ├── (50.0% 概率) ──► 跳转 [KUBE-SEP-KIK4...] ──► DNAT 目标为 10.42.0.7:80 (Oracle 节点)
   └── (100% 兜底)  ──► 跳转 [KUBE-SEP-KGYG...] ──► DNAT 目标为 10.42.2.2:80 (AWS 节点)
```

---

### 2. 现场 iptables 规则深度拆解

在 Master 节点上通过 `iptables -t nat -L -n -v` 查看当前的真实规则：

#### 步骤 1：服务总入口 `KUBE-SERVICES` 链
```text
pkts bytes target                     prot opt in out source    destination         
   7   420 KUBE-SVC-BRULDGNIV2IQDBPU  tcp  --  *  *   0.0.0.0/0 10.43.253.146  tcp dpt:4001
```
* **解析**：只要匹配到目的 IP 是 `10.43.253.146` 且端口是 `4001`，就跳转到专为该 Service 生成的链 `KUBE-SVC-BRULDGNIV2IQDBPU`。

#### 步骤 2：服务负载均衡链 `KUBE-SVC-xxx`（分母递减算法）
```text
Chain KUBE-SVC-BRULDGNIV2IQDBPU (1 references)
 pkts bytes target                     prot opt in out source       destination
    7   420 KUBE-MARK-MASQ             tcp  --  *  *  !10.42.0.0/16 10.43.253.146  tcp dpt:4001
    2   120 KUBE-SEP-X7FM5QH3NR4PGQDL  all  --  *  *   0.0.0.0/0    0.0.0.0/0     statistic mode random probability 0.33333333349
    2   120 KUBE-SEP-KIK4VDDQVU33FT2B  all  --  *  *   0.0.0.0/0    0.0.0.0/0     statistic mode random probability 0.50000000000
    3   180 KUBE-SEP-KGYGKKWPFN53UAXQ  all  --  *  *   0.0.0.0/0    0.0.0.0/0    
```
* **负载均衡算法原理**：
  - K8s 采用 `statistic --mode random` 伪随机匹配，而非严格轮询。
  - **规则 1**：以 **1/3**（`0.33333333349`）的概率命中第 1 个 Pod（`10.42.0.6`）。
  - **规则 2**：如果没命中规则 1，在剩下的 2/3 流量中以 **1/2**（`0.50000000000`）的概率命中第 2 个 Pod（`10.42.0.7`）：
    $$\frac{2}{3} \times \frac{1}{2} = \frac{1}{3}$$
  - **规则 3**：未命中前两条的流量由最后一条 100% 兜底接管（同样占总体的 **1/3**），打入第 3 个 Pod（AWS 上的 `10.42.2.2`）。
* **现场包计数验证**：
  左侧 `pkts` 计数分别为 **2、2、3**，合计恰好等于进来的 **7 个包**，精确印证了 3 个端点流量概率均分。

#### 步骤 3：端点链 `KUBE-SEP-xxx`（执行真实 DNAT 换目标 IP）
```text
# 对应 Oracle 节点 Pod 1
Chain KUBE-SEP-X7FM... -> DNAT tcp to:10.42.0.6:80

# 对应 Oracle 节点 Pod 2
Chain KUBE-SEP-KIK4... -> DNAT tcp to:10.42.0.7:80

# 对应 AWS 节点 Pod 3
Chain KUBE-SEP-KGYG... -> DNAT tcp to:10.42.2.2:80
```
* **核心动作**：数据包通过此处时，内核将 IP 报头彻底改写：
  - **原目的 IP:Port**：`10.43.253.146:4001`
  - **改写后 IP:Port**：`10.42.2.2:80`（或 `10.42.0.6/7`）
* **MASQ 源地址转换**：
  若请求来自 Pod 外部（如宿主机终端），还会打上 `KUBE-MARK-MASQ`，做 SNAT（源 IP 转换为节点 IP），确保 Pod 回包能够沿原链路正常返回。

---

### 3. 常见疑问：为什么连跑 4 次 curl 全访问到同一个 Pod？

有时在节点上执行 `for i in {1..4}; do curl ...; done`，发现返回的 Hostname 全是同一个节点。

**原因分析**：
1. **独立伪随机试验**：每次 curl 请求都是一次独立的抛骰子事件，每次命中任意一个 Pod 的概率是 1/3。连续 4 次都摇到同一个节点的理论概率是 $(1/3)^4 = 1/81 \approx 1.23\%$。
2. **小样本聚簇现象**：在毫秒级极短时间内循环发起 4 次连接，内核伪随机数生成器在小样本下极易出现短时连中同一目标的现象。
3. **验证方法**：只需将测试次数扩大到 10 次以上（如 `for i in {1..10}`），流量就会立刻平摊到所有 Pod 上（如 5 次 Oracle-1、2 次 Oracle-2、3 次 AWS）。

---

## 三、跨公网多云网络互通与 Flannel VXLAN 路由排障解析

### 1. 故障现象与原因剖析

在将 Oracle（ARM64）与 AWS（AMD64）两台跨公网云服务器组网后，虽然 `kubectl get nodes` 均显示 `Ready`，但在 Master 访问位于 AWS 机器上的 Pod（如 `10.42.2.2`）时出现请求超时（Connection timed out）。

**根本原因：**
1. **Flannel 默认提取内网 IP**：Flannel 启动时默认抓取网卡的私有 IP（Oracle 为 `10.0.0.248`，AWS 为 `172.31.5.46`），并写入 Node Annotation：`flannel.alpha.coreos.com/public-ip`。
2. **VXLAN 封包目标不可达**：跨节点通信时，Linux 内核会把内部 Pod 流量（`10.42.x.x`）打包进外层 UDP（集群指定为 `8601` 端口）。因为外层 UDP 的目标 IP 填入了对方的内网私有 IP（`172.31.5.46`），该私有地址在公网上无法路由，数据包离开主机网关后直接被公网路由器丢弃。

---

### 2. 解决方案：启用 `flannel-external-ip`

在 Master 节点的配置文件 `/etc/rancher/k3s/config.yaml` 中增加配置：

```yaml
flannel-external-ip: true
```

* **生效原理**：显式指示 Flannel 跨主机组网控制器，在建立 VXLAN 隧道端点（VTEP）时，强制使用各节点声明的外部公网 IP（`node-external-ip`），不再使用内网私有 IP。
* **修复前后 Node Annotation 对比**：

| 节点 | 修复前（Flannel Annotation 记录的隧道 IP） | 修复后（Flannel Annotation 记录的隧道 IP） | 封包路由状态 |
| :--- | :--- | :--- | :--- |
| **oracle** | `10.0.0.248` *(内网私有)* | **`161.33.37.3`** *(真实公网)* | 正常 |
| **aws** | `172.31.5.46` *(内网私有)* | **`3.107.200.83`** *(真实公网)* | 正常 |

---

### 3. 数据流转完整路径

当整个网络与 iptables 调优完成后，从 Oracle 访问位于 AWS 上的 Pod 或访问 ClusterIP 时的完整流转过程：

1. **发起请求**：终端执行 `curl http://10.43.253.146:4001`。
2. **iptables DNAT 改写**：Master 内核 `nat` 表通过随机算法将目标 IP 改为 AWS 节点上的 Pod `10.42.2.2:80`。
3. **内核查路由**：内核根据路由表 `10.42.2.0/24 dev flannel.1 onlink`，将数据包送入 `flannel.1` 虚拟隧道网卡。
4. **VXLAN 外层 UDP 封包**：Flannel 将内层 Pod 数据包封装为外层 UDP 报文：
   - **外层源 IP**：`161.33.37.3`
   - **外层目标 IP**：`3.107.200.83`
   - **外层目的端口**：`UDP 8601`
5. **跨公网传输与解包**：数据包通过公网传输至 AWS EC2，AWS 内核收到 `UDP 8601` 报文后自动解包，提取出内层原始请求并直接投递给 Pod `10.42.2.2`。
6. **响应原路返回**：AWS 节点按相同流程反向通过 UDP 隧道封装回复，实现跨公网双向通信。

---

## 四、Flannel 核心原理与 CNI 技术全景

### 1. 什么是 Flannel？
Flannel 是 Kubernetes 生态中最经典的 **CNI（Container Network Interface，容器网络接口）插件**，由 CoreOS 团队开发并开源。
它的核心使命是：**为集群中所有节点上的 Pod 铺设一张“扁平化、互不冲突的大内网”，确保任意两个 Pod 无需 NAT 即可通过独立 IP 直接通信。**

### 2. 为什么 Kubernetes 必须依赖 Flannel 这类 CNI？
单机 Docker 默认只能解决单机内部的容器通信（通过宿主机的 `docker0` 网桥分配 `172.17.0.0/16`）。
当引入多台服务器时，若不进行统筹：
- 每台机器的 Docker 会各自独立分配重复的 IP（如两台机器上都有 `172.17.0.2`），造成 IP 冲突。
- 两台机器上的容器由于没有底层路由规则，跨机无法直接寻址。

Kubernetes 官方网络模型提出了三大硬性要求：
1. **全集群 IP 唯一**：每一个 Pod 都分配全集群唯一的 IP。
2. **无需 NAT 直接通信**：所有 Pod 之间互相通信严禁通过 NAT 转换。
3. **节点与 Pod 互通**：任意节点都可以直接 ping 通任意 Pod 的 IP。

### 3. Flannel 的两大核心职责
1. **全局子网划分（CIDR Subnet 分配）**：
   - 将集群总网段（如 `10.42.0.0/16`）按节点切分成一个个子网并绑定到物理机。
   - 现场实际分配：
     - `oracle` 节点分配：`10.42.0.0/24`
     - `aws` 节点分配：`10.42.2.0/24`
2. **跨主机路由与隧道维护（跨机数据投递）**：
   - 将每个节点所属的子网及宿主机物理 IP 记录在 Kubernetes 的 Node 元数据中，确保每个节点都知晓对端 Pod 网段的下一跳（NextHop）。

### 4. Flannel 的三大核心后端模式（Backend）
| 后端模式 | 工作原理 | 适用场景 | 优缺点对比 |
| :--- | :--- | :--- | :--- |
| **VXLAN（当前使用）** | **包中包（Overlay）**：在 Linux 内核层将 Pod 的原始数据帧封装为外层 UDP 报文（默认 8472，本集群指定为 8601）。 | **跨云、跨公网、多网段分布式集群** | **优点**：无视底层物理拓扑，只要 UDP 互通即可组网。<br>**缺点**：封包解包有少许 CPU 与 MTU 开销。 |
| **host-gw** | **二层硬路由（Underlay）**：直接在宿主机写入静态路由，下一跳直接指向对端物理内网 IP。 | **同机房、同交换机下的局域网集群** | **优点**：零封装零损耗，接近物理网络裸速。<br>**缺点**：要求所有机器必须处于同一个二层物理局域网。 |
| **WireGuard** | **内核级加密隧道**：在 Linux 内核层使用 WireGuard 协议对所有跨主机的 Pod 流量做端到端加密传输。 | **公网裸传、敏感数据业务、零信任网络** | **优点**：原生强加密，安全性极高。<br>**缺点**：加密计算对 CPU 略有消耗。 |

### 5. 主流 CNI 横向对比（Flannel vs Calico vs Cilium）
| 维度 | Flannel | Calico | Cilium |
| :--- | :--- | :--- | :--- |
| **设计定位** | 极致轻量、极简修路队 | 生产级网络，功能最全面 | 基于 eBPF 的下一代云原生网络 |
| **资源消耗** | 极低（几十 MB 内存） | 中等 | 稍高 |
| **网络策略 (NetworkPolicy)** | **不支持**（K3s 借由 kube-router 补充） | 原生支持极其强大的四层防火墙 | 支持深度 L7/L4 细粒度安全策略与服务网格 |
| **部署维护** | 极简（K3s 开箱即用内置） | 较重（通常需安装 Operator 与 CRD） | 较重（对 Linux 内核版本有严格要求） |
| **典型适用场景** | 学习实验、轻量多云、边缘计算、快速上云 | 绝大多数企业级中大型中台与生产集群 | 超大规模微服务、极致性能监控与服务可观测性 |

---

## 五、NodePort 三大端口定义与自定义端口范围

### 1. Service 三大端口定义（最易混淆的致命陷阱）

在配置 NodePort 服务时，往往涉及三个端口：
| 字段 | 推荐配置 | 作用与访问地址 |
| :--- | :--- | :--- |
| **`targetPort`** | **`80`** | **Pod 容器内部应用程序实际监听的端口**（必须与 Deployment 中的 `containerPort: 80` 严格一致） |
| **`port`** | **`3000`** | **Service 虚拟 ClusterIP 暴露的端口**<br>访问形式：`http://<ClusterIP>:3000` |
| **`nodePort`** | **`8333`** (或 `30000`) | **集群所有宿主机节点（Node）对外暴露的物理端口**<br>访问形式：`http://<宿主机IP>:8333` |

* **致命默认行为**：
  在编写 YAML 时，如果漏写了 `targetPort`，Kubernetes 默认会认为 `targetPort == port`（即默认为 3000）！
  iptables 会把流量无情转发至 Pod 的 3000 端口，但容器内只有 80 端口在监听，因此内核直接返回 `Connection refused（拒绝连接）` 报错。编写时必须显式声明 `targetPort: 80`。

---

### 2. NodePort 端口范围限制深度剖析（为什么默认是 30000-32767？）

当你尝试指定 `nodePort: 8333` 时，K8s 默认会报错：
```text
The Service "service-hellok8s-nodeport" is invalid: spec.ports[0].nodePort: Invalid value: 8333: provided port is not in the valid range. The range of valid ports is 30000-32767
```

**为什么官方偏偏选了这个范围？**
Linux 系统的端口范围为 `1 ~ 65535`，Kubernetes 设计团队选择 `30000-32767`（共 2768 个端口）主要是为了**绝对避免与宿主机上的其他网络流量打架**：

```text
0            1024                   30000        32767           60999         65535
├──────────────┼──────────────────────┼────────────┼───────────────┼─────────────┤
│  特权端口区   │    常用软件服务区    │  NodePort  │ Linux临时端口  │  预留保留区  │
│  (80, 443等) │ (MySQL 3306, Redis等)│  (默认安全)│ (出站TCP随机端口)│             │
```

1. **避开知名系统与开源软件端口（1 ~ 30000）**：
   - `1 ~ 1023`：Linux 特权端口（Root 才能监听，如 22, 80, 443）。
   - `1024 ~ 30000`：绝大多数常见服务、数据库的默认端口都在这里（如 MySQL `3306`、PostgreSQL `5432`、Redis `6379`、以及 API Server `8600` 等）。如果允许随意指定，NodePort 很容易和机器上的本地服务抢端口导致冲突。
2. **避开 Linux 出站临时端口范围（32768 ~ 60999）**：
   - 宿主机对外发起网络请求（比如执行 `curl` 或系统连接外部 API）时，Linux 内核默认分配的随机客户端端口（Ephemeral Ports）就是 **`32768 ~ 60999`**（查看 `cat /proc/sys/net/ipv4/ip_local_port_range`）。
   - 如果 NodePort 占用了这一段，宿主机的出站连接很容易因为端口被 K8s 抢占而偶发失败。

因此，K8s 官方挑了中间这段“真空地带”（`30000-32767`）作为零冲突的安全区间。

---

### 3. 如何解除这个限制？（改为支持 8000-9000 等自定义端口）

该范围由 Kubernetes API Server 的参数 **`--service-node-port-range`** 控制。

在我们当前的集群中，可以通过修改 Master 配置文件直接将它扩大到包含自定义的 `8000-9000` 范围：

1. **修改 Master 配置**（在 `/etc/rancher/k3s/config.yaml` 中追加）：
   ```yaml
   # 允许 NodePort 使用 8000-32767 范围（包含开放的 8000-9000）
   service-node-port-range: "8000-32767"
   ```
2. **重启 K3s Master 生效**：
   ```bash
   sudo systemctl restart k3s
   ```
3. **编写完整实战 YAML（`service-nodeport.yaml`）**：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: service-hellok8s-nodeport
   spec:
    type: NodePort
    selector:
      app: hellok8s
    ports:
      - port: 3000        # ClusterIP 虚拟端口
        targetPort: 80    # Pod 真实监听端口
        nodePort: 8333   # 宿主机开放的自定义端口
   ```
4. **现场连通性验证**：
   在集群任意节点终端或外部浏览器直接请求即可访问：
   ```bash
   # Oracle 节点验证
   curl http://161.33.37.3:8333
   # AWS 节点验证
   curl http://3.107.200.83:8333
   ```

---

## 六、LoadBalancer 类型的两种落地形式与实现机制

在公有云托管集群（如 AWS EKS、阿里云 ACK）中，`type: LoadBalancer` 会由云厂商的 Cloud Controller Manager（CCM）自动在云端创建真正的商业硬件/云负载均衡器（如 AWS NLB）。但在自建、物理机或多云混合集群中，云厂商无法跨异构机器组网，常见两种落地方案：

### 形式一：K3s 内置 ServiceLB（Klipper-LB 软件负载均衡器，当前实际运行中）
* **底层原理**：
  - K3s 内置了轻量级软件负载均衡控制器 **Klipper-LB**。
  - 当创建 `type: LoadBalancer` 且指定端口为 `8334` 时，K3s 会自动在每个节点启动一个微型代理 Pod（`kube-system/svclb-service-hellok8s-lb-xxx`）。
  - 这些代理 Pod 借助宿主机的 `hostPort` 监听对应的端口（`8334`），并将请求转发给底层 Service 的 ClusterIP。
  - 控制器会自动将集群所有可用节点的公网 IP 填入 Service 的 `EXTERNAL-IP` 列中，彻底摆脱 `<pending>` 状态。
* **配置清单（`service-loadbalancer.yaml`）**：
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: service-hellok8s-lb
  spec:
    type: LoadBalancer
    selector:
      app: hellok8s
    ports:
      - port: 8334      # 外部负载均衡器暴露端口
        targetPort: 80  # Pod 真实监听端口
  ```
* **现场状态验证**：
  ```text
  $ kubectl get svc service-hellok8s-lb -o wide
  NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP                PORT(S)          AGE   SELECTOR
  service-hellok8s-lb   LoadBalancer   10.43.45.251   161.33.37.3,3.107.200.83   8334:24317/TCP   1m    app=hellok8s
  ```
* **访问方法**：外部直接请求 `http://161.33.37.3:8334` 或 `http://3.107.200.83:8334`。
* **适用场景**：想要标准的 Kubernetes LoadBalancer 体验，零额外第三方组件依赖。

### 形式二：宿主机外部反向代理（Caddy / Nginx 反代 NodePort，生产标准方案）
* **底层原理**：
  - K8s 内部保持 Service 为 `NodePort`（例如之前配置的 `8333` 端口）。
  - 由宿主机上已有的高性能反向代理（如当前机器上运行的 **Caddy** 或 Nginx），在 7 层处理域名、SSL 证书以及流量接入，再反向代理到本地的 NodePort（`127.0.0.1:8333`）。
* **配置示例（Caddyfile）**：
  ```caddyfile
  # 比如通过域名对外暴露，自动申请 Let's Encrypt HTTPS 证书
  hellok8s.demo.local {
      reverse_proxy 127.0.0.1:8333
  }
  ```
* **生产优势**：
  1. **端口收敛**：对外只需要开放标准的 `80` 和 `443` 端口，无需为每个业务对外暴露高位杂乱端口。
  2. **自动化证书与安全**：统一由外部代理进行 SSL/TLS 证书自动续签、WAF 防火墙过滤和防盗链配置。
  3. **架构解耦**：当底层容器集群重启或迁移时，对外访问的公网 IP 和接入网关保持完全稳定。

---

## 返回主教程

[← 返回原教程：第 7 篇 Kubernetes 服务之 Service](./doc_tutorial.md#7-使用-service)
