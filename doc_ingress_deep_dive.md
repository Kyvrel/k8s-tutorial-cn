# 📌 Kubernetes Ingress 实战与多云混合架构落地深度解析

> **说明**：本文作为 [《Kubernetes 基础教程 - 第 8 篇 Ingress》](./doc_tutorial.md#8-使用-ingress) 的深度实战扩展篇。
> 基于跨公网多云混合集群（Oracle ARM64 Master + AWS AMD64 Worker）真实环境，深度拆解 Ingress-NGINX 新版版本选型与兼容性矩阵、宿主机现有 Caddy（80/443）端口冲突与 NodePort 落地防范、ARM 架构下后端微服务优雅平替（`traefik/whoami`）、以及 Ingress 七层路径分发与 `rewrite-target` 核心工作机制。

---

## 目录
- [一、Ingress 控制器选型与版本兼容演进](#一ingress-控制器选型与版本兼容演进)
  - [1. Kubernetes 1.37 与 Ingress-NGINX 兼容性选型](#1-kubernetes-137-与-ingress-nginx-兼容性选型)
  - [2. 为什么原教程的 v1.8.2 不能直接用于新集群？](#2-为什么原教程的-v182-不能直接用于新集群)
- [二、自建 / 多云混合架构下的端口避坑指南](#二自建--多云混合架构下的端口避坑指南)
  - [1. 痛点：宿主机系统服务（Caddy）80/443 端口冲突](#1-痛点宿主机系统服务caddy-80443-端口冲突)
  - [2. K3s ServiceLB（Klipper-LB）默认端口抢占机制](#2-k3s-servicelbklipper-lb默认端口抢占机制)
  - [3. 解决方案：采用 Baremetal NodePort 模式（固定 30080 / 30443）](#3-解决方案采用-baremetal-nodeport-模式固定-30080--30443)
- [三、多架构环境原生适配与业务后端无感平替](#三多架构环境原生适配与业务后端无感平替)
  - [1. 为什么教程在 8.3 节必须部署 `deployment_httpd_svc.yaml`？](#1-为什么教程在-83-节必须部署-deployment_httpd_svcyaml)
  - [2. ARM64 架构陷阱与 `traefik/whoami` 优雅平替方案](#2-arm64-架构陷阱与-traefikwhoami-优雅平替方案)
  - [3. Service 端口真实环境对齐（4001 vs 3000）](#3-service-端口真实环境对齐4001-vs-3000)
- [四、七层 URL 路径分发与 `rewrite-target` 机制深度拆解](#四七层-url-路径分发与-rewrite-target-机制深度拆解)
  - [1. 四层转发 vs 七层路由的核心区别](#1-四层转发-vs-七层路由的核心区别)
  - [2. `rewrite-target: /` 路径重写核心机制](#2-rewrite-target--路径重写核心机制)
  - [3. 全链路请求流转图解](#3-全链路请求流转图解)
- [五、全链路实测与验证输出](#五全链路实测与验证输出)
  - [1. 控制器状态与 Ingress 路由生效验证](#1-控制器状态与-ingress-路由生效验证)
  - [2. 跨节点访问与负载均衡验证](#2-跨节点访问与负载均衡验证)
- [返回主教程](#返回主教程)

---

## 一、Ingress 控制器选型与版本兼容演进

### 1. Kubernetes 1.37 与 Ingress-NGINX 兼容性选型

原教程编写时期使用的 Kubernetes 版本为 `1.27.0`，推荐安装的 Ingress 控制器版本为 `v1.8.2`。但在生产及现代集群（如当前环境运行的 Kubernetes `v1.37.2+k3s1`）中，控制器需要匹配对应支持的版本。

根据官方 [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx#supported-versions-table) 版本的支持矩阵：

| Ingress-NGINX 版本 | 支持的 Kubernetes 版本区间 | Alpine 底包 | 内嵌 NGINX 版本 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **`v1.15.1`**（本次选用） | **1.31 ~ 1.35+（全面兼容 1.36 / 1.37）** | 3.23.3 | 1.27.1 | 官方最新终版稳定发布，全面支持 `networking.k8s.io/v1` API |
| `v1.8.2`（原教程版） | 1.25 ~ 1.28（已废弃） | 3.18.2 | 1.21.6 | 仅兼容旧版 K8s，缺少对新版安全特性与多架构镜像支持 |

### 2. 为什么原教程的 v1.8.2 不能直接用于新集群？

1. **Admission Webhook 与 Certgen 兼容性**：
   `ingress-nginx` 通过 ValidatingWebhook 在创建 Ingress 时执行语法与安全校验。`v1.8.2` 使用的 `kube-webhook-certgen:v20230407` 调用的准入 API 和鉴权组件与新版 Kubernetes API Server 存在兼容隐患。新版 `v1.15.1` 配合 `v1.6.9` 的 certgen，在容器启动后自动打上证书补丁并设置 `ttlSecondsAfterFinished: 0` 自动清理 Job。
2. **多架构（Multi-Arch）原生支持**：
   `v1.15.1` 官方镜像仓库 `registry.k8s.io/ingress-nginx/controller:v1.15.1` 为原生 OCI Image Index，同时完整包含 `linux/amd64` 与 `linux/arm64` 的原生编译二进制，在混合架构集群中调度无异常。

---

## 二、自建 / 多云混合架构下的端口避坑指南

### 1. 痛点：宿主机系统服务（Caddy）80/443 端口冲突

在自建物理机、私有云或混合云节点上，Master 机器（如当前 `oracle` 节点）往往已经作为反向代理网关运行着宿主机原生服务（例如运行在系统 systemd 中的 Caddy 或 Nginx），独占了操作系统的 `0.0.0.0:80` 和 `0.0.0.0:443` 端口：

```bash
$ ss -tulpn | grep -E ':80|:443'
tcp   LISTEN 0      4096         *:80        *:*    users:(("caddy",pid=951,fd=18))
tcp   LISTEN 0      4096         *:443       *:*    users:(("caddy",pid=951,fd=17))
```

### 2. K3s ServiceLB（Klipper-LB）默认端口抢占机制

如果直接按照官方云端快速入门下载 `cloud/deploy.yaml`，其 Service 类型为 `LoadBalancer`：
* 在托管公有云（AWS EKS、GCP GKE）中，这会触发云厂商 CCM 创建云端负载均衡器；
* 但在 **K3s / 裸机环境** 中，K3s 自带的 **ServiceLB (Klipper-LB)** 会感知到该服务，并在**集群的所有 Node 上启动 `svclb-*` 守护容器**，试图通过 `hostPort: 80` 和 `hostPort: 443` 绑定宿主机网络。
* 结果：在已运行 Caddy 的节点上，`svclb` 容器直接触发 `bind: address already in use` 发生崩溃或导致端口抢占故障。

### 3. 解决方案：采用 Baremetal NodePort 模式（固定 30080 / 30443）

使用官方 Baremetal 安装架构（对应教程 8.5 节推荐的 NodePort 方案），将 Service 改为 `type: NodePort`，并在清单中显式指定业务测试端口：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
      nodePort: 30080   # 固定暴露 HTTP 端口，避开 80/443 冲突
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
      nodePort: 30443   # 固定暴露 HTTPS 端口
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
```

**方案优势**：
1. **彻底解耦**：不触发 K3s ServiceLB，完全不影响宿主机 80/443 的 Caddy。
2. **随时可通过外部 Caddy 反代**：若未来需要通过域名对外提供标准 80/443 访问，只需在宿主机 `Caddyfile` 中配置：
   ```caddyfile
   demo.yourdomain.com {
       reverse_proxy 127.0.0.1:30080
   }
   ```
   即可由 Caddy 负责自动 HTTPS 证书申请，将流量转发给集群内 Ingress，形成标准的生产分层网关。

---

## 三、多架构环境原生适配与业务后端无感平替

### 1. 为什么教程在 8.3 节必须部署 `deployment_httpd_svc.yaml`？

在 8.3 节中，许多初学者会疑惑：“前面章节我们已经有了 `hellok8s` 服务，为什么还要部署一个 `httpd` ？”

* **Ingress 的本质是七层（HTTP/HTTPS）路由器**：它的核心价值在于**基于请求内容（路径、域名、Header）进行流量分发**。
* 如果只有一个后端，无法体现 Ingress 的路由能力；
* 教程部署 `httpd`，是为了构建两个具有鲜明差异的独立业务：
  * **后端 A（业务容器）**：响应 `/hello`，返回程序运行信息与 Hostname；
  * **后端 B（静态容器）**：响应 `/httpd`，返回 Apache 的默认欢迎页 `<html><body><h1>It works!</h1></body></html>`。

### 2. ARM64 架构陷阱与 `traefik/whoami` 优雅平替方案

* **原教程痛点**：
  教程作者提供的演示镜像 `leigg/hellok8s:v3_nginxingress` 是在 x86_64 环境编译的，在 ARM 架构（如 Oracle 机器）运行会因体系结构不匹配报错 `exec format error`。且教程要求读者自己安装 Go 环境、修改代码、执行 `docker build` 并推送 DockerHub，操作门槛高且易出错。
* **`traefik/whoami` 优雅平替方案**：
  在第四章与第七章中，我们已使用行业标杆的轻量级测试镜像 **`traefik/whoami:v1.10`** 替代 `hellok8s`：
  1. 原生支持 Multi-Arch（ARM64、AMD64），在跨云双节点无缝运行；
  2. 自身原生自带 HTTP 服务，内部监听 `80` 端口；
  3. 收到任何 HTTP 请求，会自动回显客户端连接信息以及**当前 Pod 的 Hostname**，这与教程作者特意编写的 `host: hellok8s-go-http-...` 输出完全等价！

因此，**直接复用已稳定运行的 `hellok8s-go-http` Deployment，无需编译任何 Go 代码**。

### 3. Service 端口真实环境对齐（4001 vs 3000）

在教程原版示例中，`service-hellok8s-clusterip` 的端口为 `3000`。
但我们实战集群中，之前定义的 [01-service.yaml](./01-service.yaml) 的配置为：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-clusterip
spec:
  ports:
    - port: 4001       # 真实暴露的 Service 端口
      targetPort: 80   # whoami Pod 内部监听端口
```
**避坑要点**：在编写 `ingress-hellok8s.yaml` 时，`backend.service.port.number` 必须填写真实的 **`4001`**，否则 Ingress 控制器在反向代理时会因端口不存在报错 `503 Service Unavailable` 或 `Connection refused`。

---

## 四、七层 URL 路径分发与 `rewrite-target` 机制深度拆解

### 1. 四层转发 vs 七层路由的核心区别

| 维度 | 四层负载均衡（Service / L4） | 七层反向代理（Ingress / L7） |
| :--- | :--- | :--- |
| **转发依据** | 仅基于 `IP + Port + 传输层协议（TCP/UDP）` | 基于 `HTTP Host（域名）、URL Path（路径）、Header、Cookie` |
| **应用场景** | 数据库连接、Redis 缓存、裸 TCP 流量 | 微服务 API 网关、前后端分离路由、灰度发布、SSL 卸载 |
| **路径感知** | 无法感知 URL（不能区分 `/hello` 与 `/httpd`） | 深度解析 HTTP 请求行与 URI |

### 2. `rewrite-target: /` 路径重写核心机制

在 Ingress 注解中，`nginx.ingress.kubernetes.io/rewrite-target: /` 起到了至关重要的 URL 重写作用：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

* **客户端视角**：客户端向 Ingress 控制器发送请求：
  `GET http://127.0.0.1:30080/httpd`
* **控制器处理**：
  1. Ingress 规则根据 `path: /httpd` 命中后端 `service-httpd`；
  2. 触发 `rewrite-target: /` 注解规则，控制器内部将 URI 路径从 `/httpd` 重写为 `/`；
  3. 控制器将请求改写为 `GET /` 发送给后端的 Apache httpd Pod。
* **为什么必须重写？**
  因为 Apache 容器内部的 Web 根目录只有 `/index.html`（对应根路径 `/`）。如果不对路径进行重写，Ingress 将直接透传 `GET /httpd` 给 Apache，Apache 会去寻找其文档目录下的 `httpd` 文件夹，从而返回 `404 Not Found`。

### 3. 全链路请求流转图解

```text
                                  【外部客户端请求】
                           curl http://127.0.0.1:30080/hello
                                        │
                                        ▼
                  ┌───────────────────────────────────────────┐
                  │   NodePort: 30080 (Ingress Controller)    │
                  │   Namespace: ingress-nginx (NGINX 网关)    │
                  └─────────────────────┬─────────────────────┘
                                        │
                       URL 路由匹配与路径重写 (rewrite: /)
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 │                                             │
      Path: /hello 匹配                              Path: /httpd 匹配
                 ▼                                             ▼
  ┌──────────────────────────────┐              ┌──────────────────────────────┐
  │ service-hellok8s-clusterip   │              │ service-httpd                │
  │ ClusterIP: 10.43.253.146:4001│              │ ClusterIP: 10.43.x.x:8080    │
  └──────────────┬───────────────┘              └──────────────┬───────────────┘
                 │ (NAT 负载均衡)                              │ (NAT 负载均衡)
                 ▼                                             ▼
  ┌──────────────────────────────┐              ┌──────────────────────────────┐
  │ whoami Pod (3副本轮询调度)     │              │ httpd Pod (Apache 容器)       │
  │ 返回: Hostname / IP / Header │              │ 返回: <h1>It works!</h1>      │
  └──────────────────────────────┘              └──────────────────────────────┘
```

---

## 五、全链路实测与验证输出

### 1. 控制器状态与 Ingress 路由生效验证

部署 [ingress-hellok8s.yaml](./ingress-hellok8s.yaml) 后查看规则状态：

```bash
$ kubectl get ingress hellok8s-ingress
NAME               CLASS   HOSTS   ADDRESS   PORTS   AGE
hellok8s-ingress   nginx   *                 80      1m
```

### 2. 跨节点访问与负载均衡验证

在 Master 节点直接通过 Ingress 控制器的 NodePort 端口进行验证：

#### 验证 1：访问 `/httpd` 静态服务
```bash
$ curl http://127.0.0.1:30080/httpd
<html><body><h1>It works!</h1></body></html>
```
成功接收到 Apache 返回的网页内容，证明七层路径命中与 `rewrite-target: /` 路径重写完全生效。

#### 验证 2：访问 `/hello` 业务微服务
```bash
$ curl http://127.0.0.1:30080/hello
Hostname: hellok8s-go-http-54f9dbc6b9-tddw5
IP: 127.0.0.1
IP: ::1
IP: 10.42.2.2
RemoteAddr: 10.42.0.12:49852
GET / HTTP/1.1
Host: 127.0.0.1:30080
User-Agent: curl/8.5.0
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: 127.0.0.1:30080
X-Forwarded-Port: 30080
X-Forwarded-Proto: http
X-Real-Ip: 127.0.0.1
```
多次发起请求，可以观察到 `Hostname` 在 `hellok8s-go-http` 的 3 个 Pod 之间自动轮转，且 Ingress 控制器自动注入了标准的反向代理 Header（`X-Forwarded-For`, `X-Real-Ip` 等）。

#### 验证 3：未定义的路径默认隔离
```bash
$ curl -I http://127.0.0.1:30080/not-exist
HTTP/1.1 404 Not Found
Date: Sun, 22 Nov 2026 09:18:00 GMT
Content-Type: text/html
Connection: keep-alive
```
未匹配任何 Ingress 规则的流量被安全收敛并返回 Ingress 控制器默认的 404 响应。

---

## 返回主教程
[← 返回原教程：第 8 篇 Ingress](./doc_tutorial.md#8-使用-ingress)
