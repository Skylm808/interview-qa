# Docker / Kubernetes（K8s）/ etcd 基础与面经合集（答案版）

> 适用人群：刚接触容器 / K8s 的后端同学。  
> 你的当前状态我按这个假设来整理：**会用 `kubectl` 连公司集群、能看部署和服务，但还没有系统学过 Docker / K8s 原理。**  
> 目标：先分别理解 Docker、Kubernetes、etcd，再用一条真实部署链路把三者串起来，最后补高频面试题。

---

## 一、阅读顺序：先 Docker，后 Kubernetes，再做面试题

```text
Docker：应用 + Dockerfile -> Image -> Registry -> Container -> Volume
K8s：Node / Control Plane -> Pod -> Deployment -> Service / Ingress
                                 -> ConfigMap / Secret / PersistentVolume(PV) / PersistentVolumeClaim(PVC)
                                 -> Scheduler / kubelet / CNI（容器网络接口）/ CSI（容器存储接口）
etcd：Kubernetes API 对象的强一致持久化状态存储
```

- **Docker** 解决“应用如何连同依赖被一致地构建、分发和运行”。
- **Kubernetes（K8s）** 解决“许多容器如何部署、调度、扩缩容、联网、存储与自愈”。
- **etcd** 保存 Kubernetes 控制面认定的集群状态，使多个控制面组件基于同一份事实做收敛。

因此不要先背 K8s 名词：先弄清镜像和容器，再理解 Pod 为什么是调度单位，然后理解 etcd 为什么保存“期望状态”，最后再看控制器怎样收敛它。

### 先查这张表：全文缩写、中文名与所属层

第一次看到缩写时，先不要急着背。先问三个问题：**它在哪一层、谁调用它、它解决什么问题**。下表覆盖本文会反复出现的名词；正文第一次出现时也会尽量写出全称。

| 缩写 / 术语 | 英文全称 | 中文解释 | 所属层 / 谁使用 |
| --- | --- | --- | --- |
| K8s | Kubernetes（8 表示 k 与 s 中间的 8 个字母） | **容器编排系统**：调度、伸缩、发布、自愈、网络与存储编排。 | 集群编排层 |
| Docker | Docker（产品名） | 用于构建、分发、运行容器的工具与运行环境。 | 构建与单机容器层 |
| AI | Artificial Intelligence | **人工智能**；本文的 AI / 云原生部分指承载训练、推理等工作负载的容器平台问题。 | 业务工作负载层 |
| OCI | Open Container Initiative | 开放容器规范；镜像格式和运行时行为的共同标准。Docker/BuildKit 构建出的镜像通常兼容它。 | 容器规范层 |
| CLI | Command-Line Interface | 命令行客户端，例如 `docker`、`kubectl`。它发请求，不等于后端实际执行者。 | 人与系统的入口层 |
| API | Application Programming Interface | 程序之间约定的调用接口；K8s 中通常特指 API Server 暴露的资源接口。 | 控制面入口 |
| YAML | YAML Ain't Markup Language | 人类可读的配置文件格式；用来声明 Deployment、Service、Pod 等“期望状态”。 | 声明配置层 |
| Pod | Pod（K8s 对象名） | K8s 最小调度单位，一组共享网络与可共享存储的容器。 | 工作负载层 |
| CRI | Container Runtime Interface | **容器运行时接口**；kubelet 通过它调用 containerd、CRI-O 等运行时创建 Pod sandbox 和容器。 | 节点运行时接口 |
| CNI | Container Network Interface | **容器网络接口**；运行时调用网络插件，为 Pod 配置网卡、IP、路由和网络策略。 | 节点网络层 |
| CSI | Container Storage Interface | **容器存储接口**；存储插件通过它把云盘、NFS、Ceph 等卷挂到节点和 Pod。 | 节点存储层 |
| PV | PersistentVolume | **持久卷**：集群可提供的一块实际持久化存储资源。 | K8s 存储对象 |
| PVC | PersistentVolumeClaim | **持久卷声明 / 申请单**：工作负载申请存储的对象，Pod 通常引用它而不是直接操作 PV。 | K8s 存储对象 |
| DNS | Domain Name System | **域名系统**；K8s 常由 CoreDNS 提供 Service 名到地址的解析。 | 服务发现层 |
| IP | Internet Protocol | 网络地址协议；同一 Pod 中的容器共享一个 Pod IP。 | 网络基础层 |
| RBAC | Role-Based Access Control | **基于角色的访问控制**；用 Role/ClusterRole 与 Binding 决定谁能读写哪些 K8s API 对象。 | 控制面安全层 |
| CPU | Central Processing Unit | 处理器资源；K8s 用 `m` 表示毫核，例如 `500m` 是半个核。 | 节点资源层 |
| GPU | Graphics Processing Unit | 图形 / 通用并行计算设备；在 K8s 中常以扩展资源申请。 | 节点异构资源层 |
| OOM | Out Of Memory | **内存耗尽**；进程超过可用内存或容器内存上限时，可能被内核杀死。 | Linux / 容器运行层 |
| VM | Virtual Machine | **虚拟机**；虚拟硬件并运行自己的 Guest OS，隔离通常比共享内核容器更强。 | 虚拟化层 |
| OS | Operating System | **操作系统**；容器共享宿主机 Linux 内核，镜像提供的是用户态文件与依赖，不是完整 Guest OS。 | 操作系统层 |
| cgroup | control group | Linux 的资源控制机制，限制 / 统计进程组的 CPU、内存、IO 等资源。 | Linux 内核资源层 |
| namespace | Linux namespace | Linux 的视图隔离机制，例如 PID、Network、Mount、IPC、UTS namespace。 | Linux 内核隔离层 |
| PID | Process ID | 进程编号；PID namespace 让容器看到独立的进程号空间。 | Linux 隔离层 |
| IPC | Inter-Process Communication | **进程间通信**；IPC namespace 隔离共享内存、信号量、消息队列等。 | Linux 隔离层 |
| NET / MNT / UTS | Network / Mount / Unix Time-sharing System namespace | 分别是网络、挂载点、主机名 / 域名视图隔离；都是 Linux namespace 的具体类型。 | Linux 隔离层 |
| IO | Input / Output | 输入输出资源，常在 cgroups 与存储性能语境中指磁盘 / 网络等 I/O。 | Linux 资源层 |
| NFS | Network File System | 网络文件系统，可作为 CSI Driver 对接的一类后端存储。 | 存储后端 |
| SSD / HDD | Solid State Drive / Hard Disk Drive | 固态硬盘 / 机械硬盘；StorageClass 可按这类后端性能特征区分。 | 存储硬件层 |
| LB | Load Balancer | **负载均衡器**；将流量分发到多个后端。K8s `LoadBalancer` Service 常请求云厂商 LB。 | 流量入口层 |
| CNAME | Canonical Name | DNS 别名记录；`ExternalName` Service 通常让集群 DNS 返回一个 CNAME，而不是创建 Pod 转发规则。 | DNS 层 |
| HTTP / HTTPS | Hypertext Transfer Protocol / HTTP Secure | Web 应用协议；HTTPS 是 HTTP 加 TLS 加密保护。Ingress 通常处理这类七层流量。 | 应用网络层 |
| TLS | Transport Layer Security | **传输层安全协议**；为 HTTPS 等连接提供加密、身份认证和完整性保护。 | 传输安全层 |
| TCP / UDP | Transmission Control Protocol / User Datagram Protocol | 传输层的可靠字节流协议 / 无连接数据报协议；Ingress 主要处理 HTTP(S)，TCP / UDP 需看网关或 Controller 的额外能力。 | 传输层 |
| SSH | Secure Shell | 远程安全登录协议；“SSH 到节点手改容器”会绕开 K8s 的声明式管理。 | 运维访问层 |
| CA | Certificate Authority | **证书颁发机构**；`scratch` 等极简镜像若没有 CA 根证书，程序可能无法校验 HTTPS 服务端证书。 | PKI / TLS 安全层 |
| PKI | Public Key Infrastructure | **公钥基础设施**；包含证书、私钥、CA 与信任链等体系。 | 安全基础设施 |
| CI | Continuous Integration | **持续集成**；代码提交后自动构建、测试、扫描并产出镜像的流水线。 | 交付流程层 |
| CD | Continuous Delivery / Deployment | **持续交付 / 持续部署**；把通过验证的版本逐步发布到环境。 | 交付流程层 |
| QPS | Queries Per Second | 每秒请求数，常作为服务负载或扩缩容参考指标。 | 性能指标 |
| P99 | 99th Percentile | 99% 请求不超过的延迟；用于观察长尾体验。 | 性能指标 |
| SLO | Service Level Objective | 服务等级目标，例如 P99 小于 200ms、可用性 99.9%。 | 可靠性目标 |
| SIGTERM | Signal Terminate | Linux 的“请求进程优雅退出”信号；Pod 终止时通常先发它。 | Linux 进程生命周期 |
| DDL | Data Definition Language | 数据库定义语言，例如建表、加列；发布时要考虑与新旧应用版本兼容。 | 数据库变更 |
| SBOM | Software Bill of Materials | **软件物料清单**；记录镜像中包含哪些组件和版本，便于漏洞追踪。 | 软件供应链安全 |
| P2P | Peer-to-Peer | **点对点**分发；大规模拉取镜像时可让节点间协作分发，降低 Registry 压力。 | 镜像分发层 |
| JSON | JavaScript Object Notation | 常用结构化数据格式；Kubernetes API 在传输 / 存储语义上处理结构化对象，YAML 常只是人写配置时的表现形式。 | 数据序列化层 |
| URL | Uniform Resource Locator | 网络资源地址，例如 Registry、Webhook 或 API 地址。 | Web 基础 |
| MQ | Message Queue | **消息队列**；可靠异步消息系统。etcd Watch 是状态变更通知，不提供 MQ 那种消费确认、积压与重投语义。 | 消息中间件层 |
| DB | Database | **数据库**；etcd 是面向控制面元数据的强一致 KV 存储，不替代业务关系型数据库。 | 数据存储层 |
| ID | Identifier | 标识符；例如容器 ID、节点 ID、资源 ID，用于唯一定位对象。 | 通用术语 |
| CUDA | Compute Unified Device Architecture | NVIDIA 的 GPU 计算平台与编程环境。 | GPU 软件栈 |
| MIG | Multi-Instance GPU | NVIDIA 将一张支持 MIG 的 GPU 切成多个硬件隔离实例的能力。 | GPU 切分 |
| MPS | Multi-Process Service | NVIDIA 让多个进程更好共享 GPU 执行资源的机制；不是 K8s 原生资源模型。 | GPU 共享 |
| RDMA | Remote Direct Memory Access | **远程直接内存访问**；高性能网络能力，多机训练会关心其拓扑与延迟。 | 高性能网络 |
| CAP | Consistency, Availability, Partition tolerance | 分布式系统面对网络分区时，一致性与可用性的取舍框架。 | 分布式系统理论 |
| KV | Key-Value | 键值数据模型；etcd 是强一致 KV 存储。 | etcd 数据模型 |
| TTL | Time To Live | 生存时间 / 过期时间；etcd Lease 到期可自动删除绑定的 key。 | 存储协调机制 |

> Dockerfile 指令（`FROM`、`RUN`、`COPY`、`CMD`、`ENTRYPOINT` 等）是构建脚本关键字，不是系统组件；其含义见第二章 Dockerfile 小节。遇到 `CNI / CSI / CRI` 时记住：**CRI 起容器，CNI 配网络，CSI 挂存储。**

### 再查这张表：不是缩写、但最容易把层次搞乱的 K8s 名词

| 名词 | 中文 / 直白解释 | 所属层与关系 |
| --- | --- | --- |
| Control Plane（控制面） | 管理集群、保存期望状态、安排工作；通常包含 API Server、Scheduler、Controller Manager、etcd。 | “大脑”，做全局决策，不在业务 Node 上亲自运行你的应用容器。 |
| Node（节点） | 集群中的一台工作机器，可能是物理机或 VM。 | “手脚”，kubelet 在此调用 runtime 真正运行 Pod。 |
| Controller（控制器） | 不断观察对象，把实际状态拉回期望状态的循环程序。 | 例如 Deployment Controller 发现少一个副本，会创建 Pod 对象。 |
| Reconciliation Loop（调谐 / 收敛循环） | `观察当前状态 → 与期望比较 → 执行动作 → 再观察`。 | 是 K8s 自愈、扩缩容、滚动更新的共同工作方式。 |
| Label / Selector（标签 / 选择器） | 给对象贴键值标签；Selector 用条件选中一组对象。 | Deployment、Service 常靠它找到应管理或转发给哪些 Pod。 |
| EndpointSlice（端点切片） | 记录某个 Service 当前有哪些可用后端地址的 API 对象。 | Service 的“后端名单”；readiness 失败的 Pod 通常不应在可用名单中。 |
| Pod sandbox（Pod 沙盒） | runtime 为一个 Pod 创建的共享运行边界，承载共享网络等基础设施。 | 先有 sandbox / 网络，再在其中运行该 Pod 的一个或多个容器。 |
| Container runtime（容器运行时） | 真正创建、停止、管理容器的节点软件，如 containerd、CRI-O。 | kubelet 经 CRI 调用它；Docker Engine 不是所有 K8s 节点都必须安装。 |
| Ingress Controller（Ingress 控制器） | 实际 watch Ingress 规则并承接 HTTP(S) 请求的代理 / Controller。 | Ingress 只是规则对象；Controller 才是实际处理流量的程序。 |
| CoreDNS | Kubernetes 集群常用的 DNS 服务实现。 | 让应用用 `service.namespace.svc` 这样的稳定域名找到 Service。 |

### 术语补充：流程图里剩下的“黑话”也先翻译

| 术语 | 中文解释 | 在本文中出现时到底表示什么 |
| --- | --- | --- |
| Registry（镜像仓库） | 存放、分发容器镜像的服务，例如 Docker Hub、Harbor 或云厂商镜像仓库。 | CI 把镜像推入 Registry；Node 上的 runtime 再从中拉取。 |
| image digest（镜像摘要） | 镜像内容的哈希标识，例如 `sha256:...`，内容不变则摘要不变。 | 比可变的 `latest` / `v1` tag 更适合锁定生产实际运行的版本。 |
| Admission / Webhook（准入 / 准入回调） | API Server 持久化对象前的检查或修改阶段；Webhook 是可插入的外部回调。 | 可做默认值、策略校验、注入 sidecar；它不是业务请求的普通 HTTP 网关。 |
| Bind（绑定） | Scheduler 把一个尚未调度的 Pod 指向某个 Node 的动作。 | 结果表现为 Pod 的 `spec.nodeName` 被写入；并不是创建容器。 |
| resourceVersion / revision（资源版本 / 修订号） | 对象或 etcd 状态的单调版本标记。 | List + Watch 依赖它避免漏看变更；历史被压缩后可能需要重新 List。 |
| init container（初始化容器） | 在业务容器前按顺序执行、成功后退出的一次性容器。 | 适合初始化目录、迁移、等待前置条件；未成功时业务容器不会启动。 |
| sidecar（边车容器） | 与业务容器在同一 Pod 长期并行的辅助容器。 | 常做日志采集、代理、配置同步；是容器角色，不是单独的 K8s 顶级对象。 |
| veth（virtual Ethernet pair） | 虚拟网卡对，一端在 Pod 网络命名空间，一端连到宿主机 / 网络插件。 | 是 CNI 为 Pod 组网时常见的 Linux 实现细节，不是 K8s API 对象。 |
| imagePullSecrets（拉取镜像凭据） | 让 kubelet / runtime 访问私有 Registry 的认证信息引用。 | 镜像拉取失败时需核对镜像地址、该 Secret、ServiceAccount 与网络。 |
| throttling（限流 / 限速） | 超过 CPU limit 后，cgroups 限制进程可获得 CPU 时间的现象。 | 常表现为延迟升高，不等于进程被杀；内存超限才更接近 OOM Kill。 |
| quorum（法定多数） | Raft 中可提交日志、可选出 Leader 的多数成员集合。 | 3 节点 quorum 是 2；失去 quorum 时 etcd 为保持一致性拒绝继续写。 |
| compaction / defragmentation | 历史版本压缩 / 数据文件碎片整理。 | 前者会让太旧的 Watch revision 失效；后者才可能回收已释放的磁盘空间。 |
| fencing token（栅栏令牌） | 单调递增的操作编号，资源端只接受更新的编号。 | 用于阻止旧锁持有者超时后“复活”再写入外部系统。 |

---

## 二、Docker 基础：小白先掌握这些就够了

### 0. 先把“容器”这个词讲清楚

很多人第一次学 Docker，最容易误会成：

> 容器 = 一个轻量虚拟机

这个说法不完全对。  
更准确地说：

> **Docker 容器本质上还是宿主机上的进程，只是这组进程被做了隔离和资源限制，看起来像一台独立的小环境。**

也就是说：

- 它不是像虚拟机那样真的模拟出一整套硬件
- 它也不是自带一个完整 Guest OS（客户机操作系统）
- 它更像是：
  - 共享宿主机内核
  - 但拥有相对独立的进程视图、网络视图、文件系统视图和资源配额

你可以把容器理解成：

```text
容器 = 被隔离起来的一组进程 + 运行它们所需的文件系统视图
```

#### 一个最容易懂的例子

假设宿主机上跑着：

- Java 服务 A
- Python 服务 B
- Nginx 服务 C

如果不用容器，这些程序：

- 共用宿主机环境
- 依赖版本容易冲突
- 部署时互相影响

如果把它们分别装进 Docker 容器：

- A 容器里是自己的 Java 运行时
- B 容器里是自己的 Python 运行时
- C 容器里是自己的 Nginx 配置
- 但它们底层还是共享宿主机 Linux 内核

所以容器的核心不是“虚拟一台机器”，而是：

> **把应用及其依赖装进一个相对隔离、可复制、可迁移的运行单元。**

---

### 1. Docker 到底解决了什么问题？

传统部署经常会有这些问题：

- 我的机器能跑，你的机器跑不起来
- Python / Java / Go 版本不一致
- 依赖库缺失
- 线上、测试、开发环境差异太大

Docker 的核心价值是：

> **把应用 + 依赖 + 运行环境打包成一个标准运行单元。**

所以部署时不是“再装一遍环境”，而是“直接运行这个镜像”。

---

### 2. 镜像和容器是什么关系？

#### 镜像（Image）

- 是一个**只读模板**
- 里面包含：
  - 应用代码
  - 运行时
  - 库
  - 配置

#### 容器（Container）

- 是镜像运行起来后的实例
- 可以理解成：

```text
镜像 = 类 / 模板
容器 = 对象 / 实例
```

一个镜像可以启动多个容器。

---

### 3. Docker 为什么比虚拟机轻？

#### 虚拟机（VM）

- 每个实例都有自己的 Guest OS（客户机操作系统）
- 隔离更完整
- 启动慢，资源开销大

#### 容器

- 本质上还是宿主机上的进程
- 共享宿主机内核
- 靠 Linux 内核能力做隔离和限额
- 启动快、资源开销小

一句话：

> **虚拟机是“虚拟一台机器”，容器是“隔离一组进程”。**

---

### 4. Docker 是怎么实现隔离的？

这是非常高频的大厂问题。只回答“namespace 做隔离、cgroup 做限流”还不够，最好把 Linux、Docker、容器运行时和 Kubernetes 串成一条链。

#### 4.1 先看全景：它们不是四套互相替代的技术

```text
┌──────────────────────── Kubernetes ─────────────────────────┐
│ 声明 Pod、调度 Node、维持副本、发布、自愈、Service、权限等    │
│ kubelet 根据 PodSpec，通过 CRI 请求节点 runtime 创建 Pod       │
└────────────────────────────┬─────────────────────────────────┘
                             │ CRI
                    containerd / CRI-O
                             │ OCI runtime spec
                          runc 等
                             │ 系统调用 / cgroup filesystem
                             ▼
┌──────────────────────── Linux 内核 ─────────────────────────┐
│ namespace：进程“看见什么”                                  │
│ cgroup：进程“能用多少、用了多少”                            │
│ capabilities / seccomp / AppArmor / SELinux：还能做什么       │
└─────────────────────────────────────────────────────────────┘

开发机另一条常见路径：
docker CLI → Docker Engine（dockerd）→ containerd → runc → Linux 内核
```

四者的关系可以压缩为：

| 名词 | 所在层 | 核心职责 |
| --- | --- | --- |
| Linux namespace | 内核隔离原语 | 给一组进程不同的 PID、网络、挂载点、主机名等视图 |
| Linux cgroup | 内核资源原语 | 分组统计并控制 CPU、内存、IO、进程数等资源 |
| Docker | 容器开发与运行产品 | 用镜像、文件系统、namespace、cgroup 和安全策略把应用作为容器运行 |
| Kubernetes | 集群编排系统 | 声明和调度 Pod，通过节点 runtime 批量管理容器，并把资源要求传到底层 |

因此：**namespace 和 cgroup 是 Linux 内核能力；Docker 把这些能力封装成好用的容器产品；Kubernetes 不重新实现容器，而是编排节点上的容器运行时。**Docker 官方也把容器定义为带所需文件的隔离进程，而不是一台迷你虚拟机。[Docker：What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)

#### 4.2 namespace：隔离的是“视图”，不是资源额度

宿主机最终仍只有一套 Linux 内核。namespace 让同一个内核中的不同进程看到不同的系统资源视图：

```text
宿主机真实进程树：
PID 1 systemd
PID 4201 java（容器 A）
PID 5301 nginx（容器 B）

容器 A 的 PID namespace：只看到 java 是 PID 1 及其子进程
容器 B 的 PID namespace：只看到 nginx 是 PID 1 及其子进程
```

常见类型：

| Linux namespace | 隔离什么 | 容器中的直观表现 |
| --- | --- | --- |
| PID | 进程号和进程树视图 | 容器主进程常看到自己是 PID 1，看不到其他容器进程 |
| NET | 网卡、IP、路由、端口、网络栈 | 容器有自己的 `eth0`、IP 和端口空间 |
| MNT | 挂载点和文件系统挂载视图 | 容器看到镜像 rootfs 和挂入的 Volume |
| IPC | System V IPC、POSIX 消息队列等 | 默认不与其他隔离单元共享 IPC 对象 |
| UTS | hostname、domain name | 容器或 Pod 可拥有自己的主机名 |
| USER | UID/GID 与 capability 映射 | 容器内 UID 0 可映射为宿主机非特权 UID |
| CGROUP | 进程看到的 cgroup 层级 | 隐藏宿主机上不相关的 cgroup 路径 |
| TIME | 部分系统时钟视图 | 特定场景可使用不同的时间偏移 |

namespace 的关键边界：

- 它解决“看不看得见”，不负责限制最多使用几个 CPU、多少内存；
- 不同 namespace 里的进程仍共享同一个内核，内核漏洞可能影响隔离边界；
- `--network=host`、`--pid=host`、特权容器等配置会主动减少隔离；
- USER namespace 能降低“容器内 root 等于宿主机 root”的风险，但是否启用取决于 Docker/K8s 和节点配置。

#### 4.3 cgroup：组织进程并进行资源统计、分配和限制

`cgroup` 是 control group。Linux 把进程放进一棵层级树，再由 controller 管理资源：

```text
Node
├── system.slice/                 系统服务
└── kubepods.slice/               K8s 工作负载（概念示意）
    ├── burstable/
    │   └── pod-abc/
    │       ├── container-app/
    │       └── container-sidecar/
    └── besteffort/
        └── pod-def/
```

具体目录名取决于 cgroup v1/v2、`systemd` 或 `cgroupfs` driver 以及 runtime，面试时不应死背路径。cgroup v2 使用统一层级，核心控制文件可以这样理解：

| cgroup v2 文件/控制器 | 作用 | 超限后的典型结果 |
| --- | --- | --- |
| `cpu.max` | CPU 带宽上限 | 周期内额度用完后 throttling，进程变慢而不是被杀 |
| `cpu.weight` | CPU 竞争时的相对权重 | CPU 忙时按权重竞争，不代表预留一颗物理核 |
| `memory.max` | 内存硬上限 | 回收仍失败时可能在该 cgroup 中触发 OOM Kill |
| `memory.current` | 当前内存记账 | 用于统计与监控，不是限制本身 |
| `io.max` / `io.weight` | 块设备 IO 上限/权重 | IO 延迟增加或吞吐被限制 |
| `pids.max` | 可创建的进程/线程数量 | `fork`/创建线程失败，防止 fork bomb |

Linux 内核文档将 cgroup 定义为：按层级组织进程，并受控地分配系统资源。[Linux Kernel：Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)

要记住两个反例：

```text
只有 namespace，没有 cgroup：
容器看不到别人，但仍可能吃光宿主机 CPU/内存。

只有 cgroup，没有 namespace：
进程用量受限，但仍可能看到宿主机进程、网络和挂载点。
```

所以 namespace 和 cgroup 是互补关系，不是谁包含谁。

#### 4.4 Docker 做的不只是调用 namespace 和 cgroup

执行下面的命令：

```bash
docker run --cpus=1 --memory=512m --pids-limit=200 nginx:1.27
```

概念上会经历：

```text
1. Docker 读取镜像配置和只读 layers
2. 为容器增加可写层，组合出 root filesystem
3. Docker Engine / containerd 创建容器任务
4. OCI runtime（常见 runc）创建/加入相应 Linux namespaces
5. 把容器进程放入 cgroup，设置 CPU、内存、PID 等限制
6. 应用 capabilities、seccomp、AppArmor/SELinux 等安全策略
7. 在隔离环境中启动 nginx 主进程
```

所以更完整的容器抽象是：

```text
容器
= 普通 Linux 进程
+ namespace 隔离视图
+ cgroup 资源治理
+ image/rootfs 文件系统
+ capabilities/seccomp/LSM 等权限与系统调用边界
+ runtime 生命周期管理
```

Docker Engine 启动容器时会创建 namespaces 和 control groups；默认 seccomp 等策略还会进一步收缩系统调用面。[Docker：Engine security](https://docs.docker.com/engine/security/) [Docker：Seccomp profiles](https://docs.docker.com/engine/security/seccomp/)

#### 4.5 Kubernetes 在这条链路中负责什么？

Kubernetes 负责声明、调度和收敛，不亲自在控制面调用 Linux 系统调用创建容器：

```text
Pod YAML
  │  replicas、image、resources、securityContext
  ▼
API Server → etcd → Controller → Scheduler 选择 Node
                                      │
                                      ▼
                              目标 Node 的 kubelet
                                      │ CRI
                                      ▼
                              containerd / CRI-O
                                      │
                   ┌──────────────────┼──────────────────┐
                   ▼                  ▼                  ▼
            Pod sandbox/CNI       OCI runtime       CSI/Volume
            网络 namespace       namespaces/cgroup   挂载到 MNT
```

各层职责必须讲准：

- Scheduler 根据 `requests`、亲和性、污点等选择 Node，但不创建 namespace/cgroup；
- kubelet 观察分配给本机的 Pod，通过 CRI 调用 runtime；
- runtime/OCI runtime 创建 Pod sandbox 和容器，最终使用 Linux namespace、cgroup 等能力；
- CNI 配置 Pod 网络 namespace、veth、IP 和路由；
- kubelet 与 runtime 把资源配置落实到 cgroup；
- Kubernetes Controller 发现 Pod 消失时创建替代 Pod，但不会复活原来的 Linux 进程。

Kubernetes 官方将 Pod 的共享上下文描述为一组 Linux namespaces、cgroups 和其他隔离机制；节点必须安装容器运行时才能真正运行 Pod。[Kubernetes：Pods](https://kubernetes.io/docs/concepts/workloads/pods/)

#### 4.6 Pod 里多个容器，到底共享哪些 namespace 和 cgroup？

“同一 Pod 的容器共享所有 namespace”是错误说法。

```text
Pod sandbox（常由 pause/infra container 持有 Pod 级上下文）
├── 共享 NET namespace：一个 Pod IP、同一端口空间、localhost 通信
├── Pod 级资源边界：Pod/容器 cgroup 形成层级
├── Container A：自己的 rootfs / MNT namespace，默认有自己的 PID 视图
└── Container B：自己的 rootfs / MNT namespace，默认有自己的 PID 视图
```

关键点：

- **网络一定按 Pod 共享。**同一 Pod 的容器共用 Pod IP 和端口空间，可以用 `localhost` 通信；
- **文件系统根视图不直接共享。**每个容器有自己的镜像 rootfs 和 mount namespace，需要通过同一个 Volume 共享目录；
- **PID 默认不必共享。**设置 `shareProcessNamespace: true` 后，同一 Pod 的容器才可直接看到彼此进程；
- **cgroup 通常是层级关系。**Pod 有整体边界，各容器还可有自己的资源配置，具体层级由 kubelet、QoS 和 runtime 实现；
- `hostNetwork`、`hostPID`、`hostIPC` 会让 Pod 加入宿主机对应 namespace，应谨慎使用。

Kubernetes 网络模型明确规定，一个 Pod 有独立且由其中所有容器共享的 network namespace。[Kubernetes：Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)

#### 4.7 requests / limits 最终怎样落到 cgroup？

示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-api
spec:
  containers:
    - name: app
      image: example.com/order-api:v1
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
```

不要把 request 和 limit 都说成“cgroup 硬限制”：

| 配置 | 调度阶段 | 运行阶段 |
| --- | --- | --- |
| CPU request | Scheduler 用它核算 Node 是否放得下 | 通常影响 CPU 竞争权重，不是独占 0.5 个物理核 |
| CPU limit | 不代表调度预留 | runtime 配置 cgroup CPU 带宽；超过后 throttling |
| Memory request | Scheduler 用它核算容量，也影响 Pod QoS | 不是提前锁住一块不可被别人使用的内存 |
| Memory limit | 不代表节点一定有等量空闲内存 | runtime 配置内存上限；内存压力/超限时可能 OOM Kill |

完整链路是：

```text
resources.requests
   └──► Scheduler：决定“放到哪个 Node”

resources.requests / limits
   └──► kubelet + runtime：计算 Pod/容器资源配置
           └──► Linux cgroup：统计、权重、限速、OOM 等真正执行
```

cgroup 不是调度器：它只能管理已经在这台机器上运行的进程；Scheduler 才负责事前选择 Node。Kubernetes 官方也要求 kubelet 与 runtime 使用一致的 cgroup driver；在使用 systemd 的系统上，通常应配合 `systemd` driver，避免两个管理者看到不一致的资源层级。[Kubernetes：Container runtimes and cgroup drivers](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) [Kubernetes：cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/)

#### 4.8 Docker 和 Kubernetes 是什么关系？K8s 是否必须安装 Docker？

不必须。要把“Docker 镜像”和“Docker Engine”分开：

```text
开发 / CI：Dockerfile → Docker/BuildKit 构建 OCI image → Registry
生产 K8s：Registry → containerd/CRI-O 拉 OCI image → runc → Linux 进程
```

- Docker 常用于开发、构建和本地运行容器；
- OCI 标准让镜像可以被不同运行时识别；
- kubelet 通过 CRI 连接 containerd、CRI-O 等 runtime；
- Kubernetes v1.24 已移除内置 `dockershim`，但这不影响用 Docker/BuildKit 构建镜像；
- 如确需 Docker Engine，可通过外部 `cri-dockerd` 适配，但它不是 K8s 的必需组件。

因此“Docker 是 K8s 的底层”只在非常宽泛的历史语境下勉强成立。更准确的表达是：

> **Docker 和 Kubernetes 都使用容器标准及 Linux 内核能力；Docker 偏构建与单机容器体验，Kubernetes 通过 CRI 编排集群节点上的 runtime。**

Kubernetes 官方容器运行时文档明确说明 dockershim 自 v1.24 起移除，并列出 containerd、CRI-O 等 CRI runtime。[Kubernetes：Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

#### 4.9 namespace + cgroup 是否等于安全容器？

不等于。容器共享宿主机内核，namespace/cgroup 主要解决视图和资源边界，完整安全还需要：

- 非 root 用户、USER namespace 或 rootless 模式；
- 删除不必要的 Linux capabilities，禁止 `privileged`；
- seccomp 限制系统调用；
- AppArmor/SELinux 限制文件和进程访问；
- 只读 root filesystem，谨慎使用 `hostPath`；
- Kubernetes Pod Security、RBAC、NetworkPolicy 和最小权限 ServiceAccount；
- 不可信多租户场景按风险使用 gVisor、Kata Containers/microVM 或独立节点。

虚拟机通常有独立 Guest Kernel，安全边界更厚但启动和资源成本更高；普通容器共享 Host Kernel，密度更高但更依赖内核与配置安全。

#### 4.10 一道完整例子：一个 Pod 最终怎样成为受限进程？

```text
1. 用户提交 Deployment：3 个副本，每个 500m/512Mi，limit 1C/1Gi
2. Controller 创建 Pod 对象；Scheduler 按 request 为每个 Pod 选 Node
3. Node 上 kubelet 经 CRI 请求 containerd 创建 Pod sandbox
4. runtime 创建 Pod 网络 namespace；CNI 接入 veth、分配 Pod IP
5. runtime 准备镜像 rootfs 和各容器的 mount/PID 等 namespace
6. kubelet/runtime 创建 Pod 与 container cgroup，写入 CPU/内存参数
7. OCI runtime 启动应用进程；它仍能在宿主机进程表中找到
8. CPU 超 limit 时内核 throttling；内存超限且回收失败时可能 OOM Kill
9. kubelet 观察退出并上报；restartPolicy/Controller 决定重启或补新 Pod
```

这一串分别体现：

```text
K8s：我要几个、放哪里、挂了怎样收敛
Runtime：怎样把 PodSpec 变成节点上的容器
namespace：这些进程看到什么
cgroup：这些进程最多能用多少、当前用了多少
Docker/OCI image：启动所需文件和配置从哪里来
```

#### 4.11 常见面试追问

**🟢 追问 1：namespace 和 cgroup 有什么区别？**

namespace 隔离系统资源视图，例如 PID、网络和挂载点；cgroup 按进程组统计、分配和限制 CPU、内存、IO、PID 数。前者管“看见什么”，后者管“能用多少”，二者互补。

**🟢 追问 2：容器是不是一台轻量虚拟机？**

不是。容器本质是宿主机上的受限进程，共享 Host Kernel；虚拟机虚拟硬件并运行独立 Guest OS/Kernel。容器轻量，虚拟机通常隔离更强。

**🟡 追问 3：Kubernetes Namespace 和 Linux namespace 是一回事吗？**

不是。Linux namespace 是内核进程隔离机制；Kubernetes Namespace 是 API 对象的逻辑分组，用于名称范围、RBAC、ResourceQuota 等管理。后者本身不会创建一套新的 Linux 网络/PID namespace，也不是强多租户边界。

**🟡 追问 4：K8s 的 CPU limit 和 memory limit 超过后表现一样吗？**

不一样。CPU 是可压缩资源，超出带宽额度通常被 throttling，表现为延迟升高；内存不可压缩，超过上限且回收失败时可能触发 cgroup OOM，进程被杀，K8s 再依据策略重启或补实例。

**🔴 追问 5：为什么 K8s 移除 Docker，Docker 构建的镜像还能运行？**

移除的是 kubelet 内置的 dockershim，不是 OCI 镜像标准。Docker/BuildKit 构建并推送的 OCI 兼容镜像仍能由 containerd、CRI-O 拉取，再交给 runc 等 OCI runtime 运行。

#### 面试里推荐这样答

> 容器本质是宿主机上的一组进程。Linux namespace 隔离 PID、网络、挂载点等视图，cgroup 对进程组统计并限制 CPU、内存、IO 和进程数；镜像 rootfs、capabilities、seccomp 和 LSM 再补齐文件系统与安全边界。Docker 把这些内核能力、OCI runtime、镜像和生命周期封装成开发与单机运行体验。Kubernetes 位于更上层：Controller 维持期望状态，Scheduler 根据 request 选择 Node，目标 Node 的 kubelet 通过 CRI 调 containerd 或 CRI-O，最终由 runtime 创建 Pod/容器 namespace 和 cgroup。Pod 内容器共享 network namespace，但不代表共享所有 namespace；CPU limit 超出通常 throttling，内存超限可能 OOM Kill。K8s 从 v1.24 移除的是 dockershim，不是 Docker 镜像，生产节点并不必须安装 Docker Engine。

---

### 5. Dockerfile 是什么？

`Dockerfile` 是“**怎么构建镜像的配方**”。

比如：

```dockerfile
FROM nginx:1.27
COPY ./dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

这个例子意思是：

- 以 nginx 镜像为基础
- 把前端产物拷进去
- 暴露 80 端口
- 启动 nginx

你不一定要熟练写很复杂的 Dockerfile，但至少要知道：

- `FROM`：基础镜像
- `COPY` / `ADD`：拷文件
- `RUN`：构建时执行命令
- `CMD`：容器启动时默认执行命令
- `ENTRYPOINT`：容器主入口

#### Dockerfile 常用命令：按“构建、文件、运行、治理”记

| 指令 | 发生在何时 | 用途与易错点 |
| --- | --- | --- |
| `FROM image[:tag|@digest] [AS stage]` | 构建起点 | 选基础镜像；多阶段构建每个 `FROM` 开一个新 stage。生产应尽量锁定 digest 或受控版本，不能只漂浮在 `latest`。 |
| `ARG` | 构建期 | 传构建参数；不会像 `ENV` 一样默认留给运行容器，但也**不适合放密钥**，可能出现在构建历史/缓存中。 |
| `ENV` | 构建后仍生效 | 设置运行时环境变量；配置可以用，密码/token 不可写死。 |
| `WORKDIR` | 后续指令 | 设置后续 `RUN`、`COPY`、`CMD` 的工作目录，比反复 `cd` 清晰可靠。 |
| `COPY` | 构建期 | 把 build context 的文件复制进镜像；日常优先用它。配合 `.dockerignore` 缩小上下文。 |
| `ADD` | 构建期 | 额外支持解压本地 tar、拉远程 URL 等；语义较隐式，普通复制优先 `COPY`。 |
| `RUN` | 构建期 | 在构建容器内执行安装、编译、测试；结果进入镜像层，不是容器每次启动都执行。 |
| `USER` | 运行期默认身份 | 切到非 root 用户；不要把“镜像能跑”建立在 root 权限上。 |
| `EXPOSE` | 元数据 | 声明应用预期监听端口，**不会**自动发布宿主机端口；发布仍需 `-p` 或 K8s Service。 |
| `VOLUME` | 运行期挂载点声明 | 指明适合外置持久化的路径；生产数据仍要由 volume/PVC 生命周期管理。 |
| `ENTRYPOINT` | 启动时 | 固定主可执行文件，通常用 exec JSON 形式以正确接收信号。 |
| `CMD` | 启动时 | 默认命令或 `ENTRYPOINT` 的默认参数；运行 `docker run image other` 可覆盖。 |
| `HEALTHCHECK` | 运行期 | Docker 层健康检查；在 K8s 中通常更应配置 readiness/liveness/startup probes。 |
| `LABEL` / `STOPSIGNAL` | 元数据 / 退出 | 放镜像来源、版本等元数据；指定优雅退出信号。 |

`ENTRYPOINT ["/app/server"]` + `CMD ["--port=8080"]` 的组合很常用：默认执行 `/app/server --port=8080`，运行时传额外参数可以替换 CMD 而不必改变主程序。尽量使用 exec 形式（JSON 数组），避免 shell 作为 PID 1 吞掉 `SIGTERM`，导致容器或 Pod 不能优雅退出。

#### `docker build` 时，镜像一层层到底是什么关系？

镜像可理解为**按顺序叠加的只读差异层（diff layers）+ 配置/清单**。基础镜像已经有父层；后续会按 Dockerfile 顺序执行，每个会改变文件系统的 `RUN`、`COPY`、`ADD` 等操作产生一个新 diff layer，最终由 manifest 按父子顺序引用它们。运行容器时，运行时把这些只读层用 union filesystem 合并成一个视图，并在最上面加一个容器专属可写层。

```text
基础层：        OS / runtime
第 1 层：       COPY go.mod go.sum
第 2 层：       RUN go mod download
第 3 层：       COPY 源码
第 4 层：       RUN go build
---------------------------------  镜像：只读层栈 + config
容器运行时：    最上方增加 writable container layer
```

- 层按 content digest 去重：多个镜像可共享相同基础层，Registry 与节点缓存也可复用它们。
- 缓存依赖父层：某层的 Dockerfile 指令、输入文件或父层变了，该层和其后所有层都需重建。因此依赖清单应先复制并安装依赖，业务源码后复制，避免改一行代码就重复下载依赖。
- **删除不等于缩小历史层。**若先 `RUN apt install ...`，下一层才 `RUN rm ...`，大文件仍在前一层中；应在同一层清理，或更好地用多阶段构建让构建层不进入最终镜像。
- `CMD`、`ENV`、`EXPOSE` 等主要改镜像 config / history，不一定增加文件系统 diff；面试中说“每行都等于一个很大的文件层”不够严谨。

#### 编写镜像时注意什么？怎样真正把体积做小？

目标不只是“小”：还要可复现、可缓存、可观测、最小权限和可修复。下面是一个 Go 服务的典型双阶段结构：构建工具、源码和缓存留在 build stage；最终 stage 只带可运行产物。

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

# scratch 仅适合静态二进制；若需要 CA 证书、时区或 shell，选受控的极小 runtime base
FROM scratch
COPY --from=build /out/server /server
USER 65532:65532
ENTRYPOINT ["/server"]
```

| 做法 | 为什么有效 | 代价 / 注意 |
| --- | --- | --- |
| 多阶段构建，最终只 `COPY --from=build` 产物 | 编译器、包管理器、源码、测试缓存不进最终镜像 | 不能因为多阶段而跳过测试、SBOM 和漏洞扫描 |
| 合理基础镜像 / distroless / scratch | 少包、少攻击面、少下载量 | `scratch` 没 CA、时区、shell、动态库；应用需静态构建并接受调试方式变化 |
| `.dockerignore` 排除 `.git`、测试产物、node_modules、密钥、本地缓存 | 减小 build context，也避免误把敏感文件复制进镜像 | 忽略规则要审查，别把运行必需文件排掉 |
| 先复制 lockfile，再安装依赖，源码后复制 | 提高依赖层缓存命中，缩短构建 | lockfile 必须可靠，依赖需定期更新 |
| 同一 `RUN` 中安装并清理包管理器缓存 | 不把下载缓存留在某个历史层 | 不要把大量无关命令硬拼成难维护的一行；多阶段通常更干净 |
| 非 root、只读根文件系统、构建时 secret mount | 降低权限与密钥泄露风险 | 密钥不能用 `ARG` / `ENV` / `COPY` 写进层；运行配置交给 Secret/环境注入 |
| 固定版本/digest、生成 SBOM、扫描与签名 | 可复现、可追溯、可修复供应链风险 | 不能只扫描一次，基础镜像更新也要重建 |

---

### 6. Volume（卷）为什么重要？

因为容器本身通常是**临时的**。

容器删了：

- 里面的临时写入数据可能就没了

所以要把需要保留的数据挂到 Volume 上，比如：

- 数据库数据
- 日志目录
- 上传文件目录

一句话：

> **容器适合跑程序，Volume 适合放需要持久化的数据。**

---

### 7. Docker 容器和我现在用的 Codex 沙盒有什么区别？

这题不是传统面试高频，但对理解“容器”和“受限执行环境”很有帮助。

先记一句：

> **Docker 容器更偏“应用运行环境封装”，Codex 沙盒更偏“给 AI/工具调用设边界的受控执行环境”。**

#### 共同点

它们都体现了“隔离”和“限制”的思想，比如：

- 限制能访问哪些文件
- 限制网络能力
- 限制资源或执行范围
- 让不同任务互相少影响

#### 核心区别

| 维度     | Docker 容器          | Codex 沙盒            |
| ------ | ------------------ | ------------------- |
| 主要目标   | 运行应用、部署服务          | 约束 AI/命令执行边界        |
| 关注点    | 打包环境、隔离进程、可迁移部署    | 安全、权限、读写范围、网络范围     |
| 是否强调镜像 | 很强调，镜像是核心          | 不一定强调镜像，更多强调权限策略    |
| 使用场景   | 部署 Web 服务、数据库、任务进程 | 让模型安全地读文件、跑命令、修改工作区 |
| 本质     | 一种容器化运行时环境         | 一种受控执行策略 / 沙盒机制     |

#### 直观理解

- Docker 更像：
  - “我给这个应用准备一套固定环境，让它在哪都能跑”
- Codex 沙盒更像：
  - “我允许你在这里做事，但只能在我划定的边界里做”

所以：

> **Docker 关注的是“怎么运行应用”，沙盒更关注“允许做什么、不允许做什么”。**

---

### 8. 为什么有了 Docker 还需要 Kubernetes？

这是容器方向最经典的问题之一。

Docker 解决的是：

- 单个应用怎么打包
- 单个容器怎么运行
- 单机上怎么起几个容器

但如果你面对的是：

- 几十台机器
- 上百个服务
- 容器挂了要自动拉起
- 版本升级要滚动发布
- 流量要稳定转发到可用实例
- 要根据负载自动扩缩容

那只会 Docker 就不够了。

这时候需要 K8s 去解决：

- 调度
- 自愈
- 服务发现
- 发布
- 扩缩容
- 配置和存储管理

一句话：

> **Docker 更像“怎么造并运行一个容器”，K8s 更像“怎么大规模管理很多容器”。**

补一个容易被追问的边界：Docker 既可指开发者常用的 CLI / build 工具，也可指 Docker Engine 运行时；Kubernetes 关注的是 OCI 容器镜像和 CRI 兼容 runtime（常见 containerd、CRI-O），并不要求每个节点都运行 Docker Engine。Kubernetes 从 v1.24 起不再内置 dockershim；但用 Docker/BuildKit 构建并推送 OCI 镜像，再由 K8s 节点拉取运行，仍是常见工作流。

---

## 三、Kubernetes 基础：你先把这些对象搞懂

### 1. K8s 到底解决什么问题？

如果只有一两个容器，Docker 已经够用了。  
但如果你有：

- 几十台机器
- 上百个服务
- 成千上万个容器
- 要扩缩容、滚动更新、自动恢复

那就需要 K8s 这样的容器编排系统。

K8s 主要解决：

- 调度到哪台机器
- 副本数怎么维持
- 容器挂了怎么恢复
- 服务怎么发现
- 配置怎么分发
- 存储怎么挂载

更本质地说，K8s 是一个**声明式的、持续收敛的控制系统**。你提交的不是“去 node-3 执行一条 `docker run` 命令”，而是“我希望一直有 3 个 `order-api` 副本，且它们满足这些资源、网络和配置条件”。之后控制器不断比较“期望”与“现实”，发现少一个就补一个，发现版本不同就逐步替换。

```text
你提交 YAML（期望状态）
        |
        v
API Server ----> etcd：保存对象与版本
        ^                    |
        |                    v
   状态回写 <---- Controller / Scheduler / kubelet 反复 List + Watch
                         |
                         v
                    真实节点、容器、网络、存储（实际状态）

核心动作：观察差异 -> 采取动作 -> 再观察，直到两者尽量一致
```

这也解释了一个面试常见误区：**K8s 不保证“某个原来的 Pod 永远不消失”，它保证“声明的副本和约束尽量被满足”。**Pod 所在节点故障时，控制器会创建替代 Pod；新 Pod 的 IP、名字甚至节点都可能改变，因此访问它必须依赖 Service，而不是写死 Pod IP。

---

### 2. K8s 集群最核心的角色

#### 控制面（Control Plane）

- `kube-apiserver`：所有请求入口
- `etcd`：保存集群状态
- `scheduler`：决定 Pod 调度到哪个 Node
- `controller-manager`：让实际状态朝期望状态收敛

#### 工作节点（Worker Node）

- `kubelet`：节点上的管家，负责拉镜像、起 Pod、汇报状态
- `kube-proxy`：维护 Service 转发规则
- `container runtime`：真正运行容器，比如 containerd

一句话：

> `apiserver` 是门口，`etcd` 是账本，`scheduler` 负责分配宿舍，`controller-manager` 负责查寝，`kubelet` 负责落实，`kube-proxy` 负责网络转发。

把控制面与节点侧画在一起，会更容易记住谁“做决定”、谁“动手执行”：

```text
                    ┌──────────── 控制面（全局决策）─────────────┐
kubectl / CI  ────> │ API Server <──> etcd                       │
                    │     ^          保存期望状态 / 状态记录      │
                    │     |                                      │
                    │ Controller Manager：补副本、滚动更新等      │
                    │ Scheduler：为未绑定的 Pod 选择 Node        │
                    └─────|──────────────────────────────────────┘
                          API Watch / 写状态
                            |
          ┌─────────────────┴─────────────────┐
          v                                   v
 ┌──────────── Node A ────────────┐  ┌──────────── Node B ────────────┐
 │ kubelet：接收分配，落实 Pod     │  │ kubelet：接收分配，落实 Pod     │
 │ CRI runtime：真正创建容器       │  │ CRI runtime：真正创建容器       │
 │ CNI：Pod 网络；CSI：挂卷        │  │ CNI：Pod 网络；CSI：挂卷        │
 │ kube-proxy：Service 转发规则    │  │ kube-proxy：Service 转发规则    │
 └────────────────────────────────┘  └────────────────────────────────┘
```

| 组件 | 它真正做的事 | 初学者最容易说错的点 |
| --- | --- | --- |
| `kube-apiserver` | 认证、鉴权、准入、校验 API 对象；是控制面读写的统一入口。 | 不是“只给 kubectl 用的 Web 服务”；Controller、Scheduler、kubelet 也通过它协作。 |
| `etcd` | 持久、强一致地保存 Kubernetes API 对象。 | 不直接拉镜像、不做服务转发，也不应该被业务代码直接读写。 |
| `kube-controller-manager` | 运行多类控制器，例如 Deployment / ReplicaSet 控制器，持续补齐期望副本。 | 它创建或更新 **API 对象**，不在 Node 上直接启动进程。 |
| `kube-scheduler` | 给尚未绑定节点的 Pod 做过滤、打分和绑定。 | 它只决定“去哪里”，不负责“在该节点怎样起容器”。 |
| `kubelet` | 每个 Node 上的代理，观察分配给本机的 Pod，调用 CRI 落实并汇报状态。 | 它不是全局调度器；它只管理本机。 |
| `kube-proxy` | 维护 Service 到后端 Pod 的转发规则（具体实现随模式而变）。 | Service 不是一个常驻代理进程；它是 API 对象，转发由节点网络实现。 |

---

### 3. Pod 是什么？为什么不是直接调度容器？

Pod 是 K8s 里的**最小调度单元**。

它不是“某个容器”，而是“**一组共享网络和存储的容器**”。

#### 一个 Pod 有哪些组件？先区分“YAML 里声明的”和“运行时提供的”

| 组成 | 是否通常写在 Pod spec | 作用 |
| --- | --- | --- |
| `containers`（应用容器） | 是 | 真正提供业务的一个或多个容器；每个容器各自有 image、command、resources、probes、env、volumeMounts。 |
| `initContainers` | 可选 | 在 app container 之前**按顺序**运行并成功结束，适合初始化目录、等待依赖、下载一次性配置。 |
| sidecar | 可选 | 与主应用长期并行运行，如代理、日志采集、配置同步；它是容器角色，不是一个独立 K8s 顶级对象。 |
| ephemeral container | 仅排障时动态添加 | 临时调试已运行 Pod，不是业务容器，不保证资源也不会自动重启。 |
| `volumes` + `volumeMounts` | 可选 | Pod 级共享存储，如 `emptyDir`、ConfigMap、Secret、PVC；多个容器可挂载同一卷。 |
| Pod 网络 / sandbox（常被称 pause） | 运行时创建 | 让同一个 Pod 的容器共享 Pod IP、端口空间和网络命名空间；不是业务 YAML 中要手写的一个普通容器。 |
| Pod 级配置 | 是 | metadata/labels、serviceAccount、securityContext、DNS、restartPolicy、调度约束等，定义身份、安全和运行边界。 |

Service、Ingress、Deployment、ConfigMap/PVC 都会被 Pod 引用或管理，但**不属于 Pod 内部组件**。最常见形态仍是“一个 app container + 可选 init container / sidecar + volume + Pod 网络”；不要为了凑多容器把无关服务塞进一个 Pod，独立伸缩、独立故障域的服务应拆为不同 Pod。

通常你会遇到两种情况：

1. **一个 Pod 里一个主容器**（最常见）
2. **一个 Pod 里多个容器**
   - 主业务容器
   - sidecar 容器（日志收集、代理、监控）

#### 用类比怎么理解 Pod？

你可以把：

- **容器** 理解成“宿舍里的人”
- **Pod** 理解成“宿舍本身”

K8s 真正调度的不是“某个人”，而是“这一整间宿舍”。

因为同一个 Pod 里的容器通常会：

- 共享同一个 Pod IP
- 共享网络命名空间
- 可以共享同一个 Volume
- 生命周期绑定在一起

这就很像宿舍里的人：

- 共用同一个门牌号
- 共用同一套基础设施
- 一起入住，一起搬走

#### 为什么 K8s 不直接调度单个容器？

因为真实业务里经常不是“一个容器自己就能完整工作”，而是：

> **有些容器必须一起出现、一起消失、一起共享网络和存储。**

比如：

- 主业务容器负责提供服务
- sidecar 容器负责收日志
- sidecar 容器负责代理、监控或配置同步

这时候如果直接调度单个容器，就很难表达这种“强绑定关系”。  
而 Pod 恰好就是为了表达：

> **这几个容器是一组，要一起被调度。**

#### 一个最容易懂的例子

假设：

- 主容器把日志写到共享目录
- sidecar 容器负责从这个目录收集日志并发走

如果它们在同一个 Pod 里：

- 可以共享卷
- 可以通过 `localhost` 通信
- 被当成一个整体调度

这就很自然。

#### 面试里怎么答最稳？

> Pod 可以理解成 Kubernetes 里的一个“小运行舱”，而容器是舱里的具体进程。一个 Pod 里可以只有一个主容器，也可以有多个强关联容器，比如主业务容器和 sidecar。它们共享同一个网络命名空间、同一个 Pod IP，也可以共享卷，所以 Kubernetes 不直接调度单个容器，而是调度 Pod，因为 Pod 更适合表达“一组需要一起运行、一起共享资源的容器”。

一句压缩版：

> **容器像人，Pod 像宿舍；K8s 管的是宿舍，不是单独某个人。**

---

### 4. Deployment 是什么？

`Deployment` 用来管理一组无状态 Pod。

它解决的问题：

- 我要几个副本
- 升级怎么滚动发布
- 失败了怎么回滚

最重要的关系是：

```text
Deployment -> ReplicaSet -> Pod
```

你一般不会手动管 ReplicaSet，主要管 Deployment。

再向下展开一次：

```text
Deployment（我要 3 个 v2 副本，按滚动方式更新）
      |
      v
ReplicaSet（维持“这个版本”应该有 3 个 Pod）
      |
      v
Pod（真正被调度到 Node 的运行单元）
```

当你把镜像从 `v1` 改为 `v2`，Deployment 通常不会把所有旧 Pod 一把删掉。它会新建一个代表 `v2` 的 ReplicaSet，逐步扩出就绪的新 Pod，同时逐步缩小 `v1` ReplicaSet，直到达到发布策略所允许的数量。这就是滚动更新。回滚本质上是把 Deployment 的 Pod 模板恢复到上一版本，而不是去某台机器手动改容器。

---

### 5. Service 是什么？

因为 Pod 是会变的：

- Pod 可能被删
- Pod IP 可能变
- 副本数可能变化

所以不能直接依赖 Pod IP。

`Service` 的作用就是：

> **给一组 Pod 提供稳定的访问入口。**

高频类型：

- `ClusterIP`：集群内访问
- `NodePort`：通过节点端口暴露
- `LoadBalancer`：通过云厂商 LB 暴露
- `ExternalName`：DNS 别名映射

Service 由两部分组成：**稳定名字 / 虚拟入口**，以及**一组随 Pod 变化的可用后端**。它一般通过 label selector 找 Pod；只有就绪的 Pod 才应进入 EndpointSlice（端点切片）并接收流量。

```text
调用方访问 order.default.svc.cluster.local:80
                  |
                  v
       Service：稳定 DNS + ClusterIP + selector: app=order
                  |
                  v
 EndpointSlice：当前就绪的 Pod IP 列表
                  |
                  v
 节点网络转发规则（常见 kube-proxy 或 CNI 的实现）
                  |
                  v
        Pod A / Pod B / Pod C
```

所以要区分两个问题：Service 解决“**稳定地找到哪一组后端**”，负载均衡实现负责“**一次请求具体转到哪个后端**”。如果 Service 访问失败，先查 selector 是否匹配 Pod 标签、Pod 是否 Ready、EndpointSlice 是否有地址，再查 DNS / 网络策略 / 端口，而不是先怀疑容器本身。

---

### 6. Ingress 是什么？

如果你有很多 HTTP 服务，不可能每个服务都暴露一个外网端口。  
这时通常用 `Ingress` 统一做七层路由。

比如：

- `/api` 转到后端服务 A
- `/admin` 转到后台服务 B
- `app.xxx.com` 转到服务 C

一句话：

> `Service` 更像服务内部稳定入口，`Ingress` 更像集群外部 HTTP/HTTPS 入口路由层。

注意：Ingress 是一份“路由规则”对象，**自己并不处理请求**；还必须有已经安装并运行的 Ingress Controller（例如 NGINX Ingress Controller 或云厂商实现）去 watch 规则并配置真实的代理 / 负载均衡器。

```text
浏览器 https://app.example.com/api/orders
       -> 外部 LB（可选）
       -> Ingress Controller（TLS 终止、按 host/path 匹配）
       -> Service order-api
       -> 就绪的 Pod
```

Ingress 主要面向 HTTP / HTTPS 七层路由。TCP、UDP、非 HTTP 协议，或者需要更复杂的 API 网关能力时，要看具体 Controller、Gateway API 或专门网关的能力，不能笼统说“Ingress 可以代理任何流量”。

---

### 7. ConfigMap 和 Secret 是什么？

- `ConfigMap`：放普通配置
- `Secret`：放敏感信息，比如密码、token、证书

它们通常会被挂到：

- 环境变量
- 文件

两者都只是 K8s API 对象，区别在于使用意图与权限控制：Secret 会以适合敏感数据的方式处理，并应限制 RBAC 读取权限；但它**不等于自动端到端加密的密码箱**。例如把 Secret 直接打印到日志、写进镜像、给所有 ServiceAccount 读取，依然会泄露。

实践上，普通配置变更是否能让已运行应用自动看到，取决于注入方式和应用是否会重载：环境变量通常需要重建 Pod；以 volume 挂载的 ConfigMap / Secret 会被 kubelet 异步更新文件，但业务进程仍要自己监听或重载配置。

---

### 8. PV / PVC / StorageClass 是什么？

这是新手最容易乱的地方。

#### `PV`（PersistentVolume）

持久卷，集群里的存储资源。

#### `PVC`（PersistentVolumeClaim）

持久卷申请单，Pod 不直接找 PV，而是提 PVC。

#### `StorageClass`

存储类型 / 动态供给策略，比如：

- SSD
- HDD
- 不同云盘类型

一句话理解：

```text
PV  = 仓库里的真实货
PVC = 你提交的领货单
StorageClass = 货物类别 / 供货规则
```

若使用动态供给，链路通常是：

```text
Pod 引用 PVC
  -> PVC 指定 StorageClass（例如高性能 SSD）
  -> 对应 CSI Driver 创建 / 选择真实后端卷
  -> PVC 与 PV 绑定
  -> kubelet / CSI 在目标 Node 挂载卷，再挂进容器目录
```

这张图能帮你避免两个常见误解：第一，PVC 不是目录，而是“存储需求的声明”；第二，PVC 绑定成功不等于应用已能读写，仍可能在节点挂载、权限、文件系统或应用路径上失败。

### 9. 一个 Pod 怎样从 YAML 变成真正运行的容器？

这题建议按“**声明被接收 → 控制器造 Pod → Scheduler 选节点 → kubelet 在节点落地 → 网络/存储准备 → 就绪后接流量**”讲。这样既有主线，也不会把每个组件的职责说反。

先分清两种提交：你可以直接提交一个 Pod YAML；生产更常见的是提交 Deployment YAML。后者不会立即出现容器，而是先由 Deployment 控制器创建 ReplicaSet，再由 ReplicaSet 创建 Pod。两种情况从“Pod 已被创建但还没分配节点”开始，后半段一致。

```text
                 ① 声明与持久化（控制面）
kubectl apply Deployment.yaml
          |
          v
 API Server：认证 -> 鉴权(RBAC) -> 准入(默认值 / 策略 / Webhook) -> 字段校验
          |
          v
 etcd：保存 Deployment 的期望状态与 resourceVersion
          |
          v
 Deployment Controller --watch--> 创建 ReplicaSet --watch--> 创建 Pending Pod

                 ② 调度（只做“选哪台机器”）
 Pending Pod（spec.nodeName 为空）
          |
          v
 Scheduler：过滤不合格 Node -> 给候选 Node 打分 -> Bind
          |
          v
 API Server / etcd：Pod.spec.nodeName = node-b

                 ③ 节点落地（真正“起容器”）
 node-b 上的 kubelet --watch--> 看到分给自己的 Pod
          |
          +--> 准备 volume：需要持久盘时协调 CSI Driver 挂载
          +--> 通过 CRI 请求 containerd / CRI-O 创建 Pod sandbox
          |       \-> runtime 调用 CNI 插件，建立 Pod 网络、分配 IP、配置路由
          +--> 拉取镜像 -> 创建 initContainers（按顺序成功结束）
          +--> 创建并启动业务 containers / sidecar，按 cgroups 设置资源限制
          |
          v
 kubelet 持续把 PodStatus、容器状态、探针结果回写 API Server

                 ④ 变成“可接流量的服务”
 readiness probe 成功
          -> EndpointSlice 记录该 Pod 为可用后端
          -> Service / DNS / 节点转发规则将请求送到该 Pod
```

#### 每一步谁做、做完看到什么

| 阶段 | 真正的执行者 | 关键动作 | 常见可观察状态 |
| --- | --- | --- | --- |
| 1. 接收声明 | `kube-apiserver` | 验证请求；把 API 对象写入 etcd。 | `kubectl get deploy/pod` 能看到对象；错误则在 apply 阶段返回。 |
| 2. 补出 Pod | Deployment / ReplicaSet Controller | 发现期望副本大于实际副本，创建 Pod 对象。 | Pod 出现，常为 `Pending`。 |
| 3. 选择节点 | `kube-scheduler` | 根据 `requests`、剩余资源、节点选择器、亲和 / 反亲和、污点与容忍等过滤、打分并绑定。 | `spec.nodeName` 有值；`describe pod` Events 有 `Scheduled`。 |
| 4. 准备节点资源 | 目标 Node 的 `kubelet`、CSI Driver | 拉取 Secret / ConfigMap，若有 PVC 则准备并挂载卷。 | 失败常见 `FailedMount`、`FailedAttachVolume`。 |
| 5. 建 sandbox 与网络 | runtime + CNI 插件 | 创建 Pod sandbox（共享网络的运行边界）；配置网络命名空间、veth、IP、路由、网络策略等。 | 失败可能停在 `ContainerCreating`，Events 常有 CNI 错误。 |
| 6. 启动容器 | `kubelet` 经 CRI 调用 runtime | 拉镜像；init container 顺序执行；再启动 app container 和 sidecar。 | `Pulling` / `Pulled` / `Created` / `Started`，或 `ImagePullBackOff`、`CrashLoopBackOff`。 |
| 7. 加入流量 | kubelet + EndpointSlice 相关控制器 + Service 网络实现 | readiness 成功后才把它列为可用后端。 | `Running` 不一定 `Ready`；`kubectl get endpointslice` 可验证后端。 |

#### 三个必须讲对的细节

1. **Scheduler 不创建容器。**它只给 Pending Pod 写入绑定结果（目标 Node）；真正拉镜像、调用 runtime、启动容器的是该 Node 的 kubelet。
2. **etcd 不直接给组件“发命令”。**它是 API 对象的一致性存储；各组件经 API Server 用 List / Watch 观察变化，再按职责采取动作。
3. **`Running` 不等于已对外可用。**容器进程启动后 Pod 可以是 Running，但 readiness 未通过、Service selector 不匹配或 EndpointSlice 未更新时，请求仍不会被正常转给它。

#### 面试中可直接复述的 60 秒版本

> 以 Deployment 为例，`kubectl apply` 先把声明交给 API Server。API Server 做认证、RBAC 鉴权、准入和校验后，把 Deployment 持久化到 etcd。Deployment 和 ReplicaSet 控制器通过 API Watch 发现期望副本，创建还没有绑定节点的 Pending Pod。Scheduler 根据资源 request、亲和性、污点容忍等筛选并打分，只负责选出 Node 并绑定。目标节点上的 kubelet watch 到这个 Pod 后，先准备 ConfigMap、Secret 和可能的 PVC，然后通过 CRI 调 containerd 等 runtime 创建 Pod sandbox；runtime 调 CNI 配网络，CSI 在需要时挂存储；接着拉镜像、依次跑 init container、启动业务容器。kubelet 上报状态并执行 probes，readiness 成功后 EndpointSlice 更新，Service 才把流量导到这个 Pod。Scheduler 只调度，kubelet 才是节点上的实际执行者。

#### 它卡住时怎么排查：先按所处阶段缩小范围

```text
对象不存在 / apply 失败       -> YAML、认证、RBAC、准入策略
Pod 一直 Pending              -> Scheduler 事件：资源、污点、亲和性、PVC 未绑定
ContainerCreating 很久        -> 镜像、CNI、CSI、节点磁盘 / runtime
ImagePullBackOff              -> 镜像名 / digest、Registry 权限、imagePullSecrets、网络
CrashLoopBackOff              -> 应用日志、退出码、命令、配置、依赖、端口
Running 但访问不到            -> readiness、Service selector、EndpointSlice、端口、网络策略
```

新手先用这四条命令，不要凭感觉猜：

```bash
kubectl get pod -o wide                  # 看 Pod 状态和被调度到哪个 Node
kubectl describe pod <pod-name>          # 重点看最下方 Events
kubectl logs <pod-name> -c <container>   # 看当前容器日志
kubectl logs <pod-name> -c <container> --previous  # 容器重启过时看上一次日志
```

`CNI`（Container Network Interface，容器网络接口）负责网络；`CSI`（Container Storage Interface，容器存储接口）负责存储；`CRI`（Container Runtime Interface，容器运行时接口）是 kubelet 调 runtime 的接口。最短记忆法：**CRI 起容器，CNI 配网络，CSI 挂存储。**

### 10. Pod 的 CPU、内存怎样隔离？

完整路径是：**`requests / limits → scheduler → kubelet / runtime → Linux cgroups`。**

- `requests` 是调度预留量。Scheduler 将一个 Pod 中各容器的 request 汇总，只有 Node 剩余可分配资源足够才会放置它；CPU request 在竞争时通常对应更高的 CPU 时间权重。
- `limits` 是运行上限。kubelet 将数值交给 runtime，Linux 节点一般由 cgroups 执行：CPU 到上限会被 throttling；内存超过 limit 时，内存压力下可能被 OOM Kill。
- 例如 `request: 500m / 512Mi`、`limit: 1 CPU / 1Gi`：调度需要节点至少有 0.5 核和 512Mi 余量；运行时超过 1 CPU 会被限速，内存失控则可能被杀后重启。

这里的 Linux `namespace` 解决“进程能看见哪些 PID、网络、挂载点”；`cgroups` 解决“最多能用多少 CPU、内存、IO”。不要把它和 Kubernetes Namespace（API 对象的逻辑管理边界）混为一谈。

### 11. CNI、CSI 和 RuntimeClass 分别解决什么？

这三个词很像，但分别在三条完全不同的链路上。先记全称：

| 名词 | 英文全称 / 中文 | 谁调用谁 | 解决的问题 |
| --- | --- | --- | --- |
| `CNI` | **Container Network Interface，容器网络接口** | runtime 在创建 Pod sandbox 时调用 CNI Plugin。 | 给 Pod 建网络命名空间、网卡、IP、路由；网络策略也常由相应网络方案落实。 |
| `CSI` | **Container Storage Interface，容器存储接口** | kubelet 与 CSI Driver 协作。 | 将 PVC 对应的云盘、NFS、Ceph 等真实存储创建、附着、挂载进 Pod。 |
| `RuntimeClass` | Runtime Class，**运行时类别** | Pod 在 `spec.runtimeClassName` 选择；kubelet / runtime 据此选配置。 | 让不同工作负载选择不同 runtime handler 或隔离配置，例如更强的 sandbox 运行时。 |

```text
同一个 Pod 被分到 Node 后

kubelet --CRI（容器运行时接口）--> runtime --> CNI Plugin --> 获得 Pod 网络
   |
   +--PVC / PV--> CSI Driver --> 后端盘 / NFS / Ceph --> 挂到容器目录
   |
   +--runtimeClassName（可选）--> 选择 runtime handler / 隔离配置
```

它们分别位于网络、存储和运行时隔离三层。不要把“装了 CSI”理解成网络打通，也不要把“Pod 拿到 IP”理解成 PVC 已可用；一个 Pod 可能没有 PVC，因此 CSI 不参与，但每个正常联网的 Pod 都需要某种网络配置。RuntimeClass 也不是“给容器加一个名字”，它会影响底层运行时选择、调度开销或隔离边界，取决于集群管理员的配置。

---

## 四、etcd 基础：Kubernetes 控制面的强一致账本

### 1. etcd 是什么，适合存什么？

etcd 是一个分布式、强一致的键值（Key-Value）存储，适合保存**数据量不一定大、但不能各说各话的元数据**：配置、选主状态、服务实例元信息、锁和集群期望状态。

它不适合替代 MySQL、对象存储或日志系统去承载海量业务明细；Kubernetes 使用它，是因为 Pod、Node、Deployment、Service、Secret 等集群对象必须有可靠、可一致读取的事实来源。

在 Kubernetes 的正常架构里：

```text
kubectl / Controller / Scheduler / kubelet
              -> kube-apiserver
              -> etcd
```

在**日常 Kubernetes 控制路径**中，`kube-apiserver` 是 etcd 的直接客户端；备份 / 恢复工具是例外，但也应受严格证书与网络边界控制。其他组件通过 API Server 的 API、List / Watch 获取对象并回写状态；不要把 etcd 当成业务服务注册中心，也不要让业务服务绕过 API Server 直接访问它。Service、CoreDNS、kube-proxy 才是服务发现和转发链路中的直接角色。

### 2. etcd 为什么常说是 CP？

CAP 中：

- `C`（Consistency）：写成功后，后续强一致读看到的是同一最新提交结果；
- `A`（Availability）：每次请求都能获得响应；
- `P`（Partition Tolerance）：网络分区时系统仍能继续按既定语义工作。

网络分区无法回避。etcd 基于 Raft，在分区时优先守住一致性：写入必须经过 Leader 且得到多数派确认。3 节点集群需要至少 2 个节点形成多数；少数派即使机器还活着，也不能继续提交写入。它牺牲的是这部分场景的可写性，而不是让两边各写一份冲突的集群状态，因此通常称为偏 CP。

### 3. Raft：选主与日志复制怎样保证一致？

Raft 的核心是“一个写入口、一条日志顺序、一个多数派提交规则”。角色有：

- `Follower`：默认角色，接收 Leader 的心跳和日志；
- `Candidate`：在选举超时未收到心跳时发起竞选；
- `Leader`：当前处理写请求并复制日志的唯一节点。

```text
Follower 长时间未收到心跳
  -> term + 1，成为 Candidate，先投自己
  -> RequestVote 拉票（候选人的日志不能明显落后）
  -> 获得多数派投票，成为 Leader
  -> 定期以 AppendEntries 发送心跳 / 推进日志
```

客户端写入时，Leader 先追加日志、复制给 Follower；**多数派确认后**才标记 committed，再按相同顺序应用到各节点状态机并返回成功。`term` 是任期编号：旧 Leader 恢复后若发现更大的 term，必须退回 Follower，避免脑裂后继续写。选举超时会随机化，避免所有 Follower 同时竞选造成持续平票。

多数派的关键是“任意两个多数派必有交集”，已提交日志不会被新 Leader 遗失。面试压缩版：**Leader 统一写入顺序，多数派决定提交，term 和日志新旧约束保证新 Leader 不倒退。**

### 4. etcd 的读、Watch、Lease 分别是什么？

- **写：**写入走 Leader，日志多数派确认后提交；强一致读需走线性一致语义，不能把任意本地旧副本的值当成最新值。
- **Watch：**订阅某个 key 或前缀的变更事件，用事件驱动替代不停轮询。Kubernetes 的控制器就是通过 List + Watch 感知对象变化，再让实际状态向期望状态收敛。
- **Lease：**给 key 绑定租期并持续 keepalive；客户端失联后 Lease 到期，相关 key 自动删除。它适合临时注册或选主等“实例失联就应自动失效”的状态。

Watch 不是可靠消息队列：消费者要保存 revision，遇到 compaction 或连接中断时重新 List 并从新 revision Watch；事件处理本身也要幂等。

### 5. etcd、ZooKeeper、Redis 怎么选？

- etcd：强一致 KV、Watch、Lease，并与 Kubernetes / 云原生生态贴合；
- ZooKeeper：经典协调系统，生态成熟；
- Redis：缓存和高性能数据访问很强，也可实现简单注册 / 锁，但不能因为它快就忽略一致性、故障转移和锁语义。

不要回答“谁绝对更好”。先看目标是强一致协调、生态兼容、吞吐延迟还是实现复杂度；Kubernetes 控制面选择 etcd 的关键是保存一致的集群元数据，不是追求存业务大数据。

### 6. etcd 为什么要奇数节点？节点故障和备份怎么回答？

etcd 使用 Raft 多数派提交。3 个节点需要 2 个节点在线，5 个节点需要 3 个节点在线；因此从“可容忍故障数”看，3 节点可坏 1 个，4 节点也仍只可坏 1 个，5 节点才可坏 2 个。4 个节点比 3 个节点多消耗资源、写入确认更多，却没有提高多数派容错能力，所以常部署为 3 或 5 个节点，并尽量跨可用区 / 故障域。

```text
3 节点：A、B、C，写成功需多数派 = 2

A 挂掉：B + C 仍是 2，能选 Leader、能写
A、B 挂掉：只剩 C，不足 2，拒绝写，避免产生两份冲突状态
```

备份不是“把某台机器上的数据目录随手复制走”这么简单。生产应该定期执行一致性快照、验证快照可用，并演练在隔离环境恢复；恢复后还要按集群 / 控制面流程重新建立成员关系，不能把旧数据目录直接覆盖到还在运行的集群。另一个常见运维概念：**compaction（压缩历史 revision）**用于清理旧版本事件，避免数据库无限增长；**defragmentation（碎片整理）**用于回收已经不再使用的磁盘空间。二者都要结合集群版本、磁盘与业务窗口审慎操作。

面试回答可以收束为：**“etcd 用奇数节点是为了用最少节点得到所需多数派容错；高可用不等于不备份，要做一致性快照、恢复演练，并监控 leader 变化、磁盘延迟、空间和 quorum。”**

---

## 五、把 Docker、Kubernetes、etcd 串起来：部署一个订单 API 的完整例子

假设我们有一个 `order-api`：需要 3 个副本、只接受就绪流量，配置来自 ConfigMap，订单库密码来自 Secret。开发者先用 Dockerfile 把 Go 二进制和运行依赖构建成镜像，推到 Registry：

```text
代码 + Dockerfile
  -> CI 构建 order-api@sha256:abc（不可变镜像）
  -> Registry
  -> 提交 Deployment / Service YAML
```

随后 Kubernetes 与 etcd 的协作如下：

```text
1. kubectl apply YAML
   -> API Server 认证、鉴权、准入校验
   -> API Server 将 Deployment、Service、ConfigMap、Secret 的期望状态写入 etcd

2. Deployment Controller 通过 API Server Watch 到 Deployment
   -> 创建 ReplicaSet，再创建 3 个尚未绑定 Node 的 Pod
   -> 这些对象的期望状态继续由 API Server 持久化到 etcd

3. Scheduler 通过 API Server Watch 到 Pending Pod
   -> 按 request、节点可用资源、污点容忍、亲和性筛选和打分
   -> 选择 node-b，并通过 API Server 写入 Pod.spec.nodeName
   -> API Server 再将这次绑定写入 etcd

4. node-b 的 kubelet 通过 API Server Watch 到“分配给我”的 Pod
   -> 通过 CRI 让 containerd 拉取 order-api@sha256:abc
   -> 创建容器；CNI 配置 Pod 网络；若有 PVC 则 CSI 挂盘
   -> 按 requests / limits 配置 cgroups，并持续上报 PodStatus 给 API Server

5. readiness probe 通过
   -> EndpointSlice / Service 相关控制器更新可用后端
   -> CoreDNS / kube-proxy 等据此让流量进入这个就绪 Pod
```

这里三者的职责不能互换：**Docker 产出可运行、可复现的镜像；Kubernetes 把“3 个副本应该运行”的声明调度并收敛为现实；etcd 持久化并一致地保存这份声明和控制面状态。**etcd 不存镜像层、不直接创建容器，也不承担订单业务数据。

### 同一个例子的两次追问

**Pod 崩了怎么办？**kubelet 上报容器退出；控制器从 API Server 观察到实际副本不足，创建替代 Pod。整个过程依赖 etcd 中“期望 3 副本”的事实，而不是某台机器临时记忆“原来有 3 个”。

**把镜像升级为 `sha256:def` 怎么办？**修改 Deployment 的 Pod template。API Server 写入 etcd；Deployment Controller Watch 到新版本后按滚动升级策略创建新 Pod、等待 readiness，再逐步缩掉旧 ReplicaSet。发生异常则暂停 / 回滚 template；不能仅在某台 Node 上手工 `docker pull`，否则控制面声明与真实状态会漂移。

### etcd 不可用时会怎样？

已经在节点上运行的容器通常不会因为 etcd 短暂不可用立刻停止；但 API Server 无法可靠读写控制面状态，创建 Pod、调度、扩缩容、发布、控制器收敛等会受影响。生产中 etcd 常部署奇数节点（通常 3 或 5）跨故障域，监控 leader 变化、磁盘延迟、请求延迟、DB 大小和告警；并定期做快照备份与恢复演练。

---

## 六、你现在会用 kubectl，但要知道它到底在做什么

`kubectl` 本质上是一个客户端工具。  
它通常不是直接去某个 Pod 上执行魔法，而是：

> **把命令发给 `kube-apiserver`，再由控制面和节点组件协同完成。**

### 你最常见的几个命令到底干嘛

- `kubectl get pods`
  - 看资源列表
- `kubectl describe pod xxx`
  - 看更详细的状态、事件、调度信息
- `kubectl logs xxx`
  - 看容器日志
- `kubectl exec -it xxx -- /bin/sh`
  - 进入容器执行命令
- `kubectl get svc`
  - 看 Service
- `kubectl get deploy`
  - 看 Deployment
- `kubectl top pod`
  - 看资源使用（前提是 metrics 能力存在）

你现在虽然还没深入底层，但只要把这些命令背后的对象关系理解了，面试就会稳很多。

---

## 七、高频面试题：按“结论 → 原理 → 易错点”作答

> 第二、三章负责把概念讲透；本章把它压缩成面试时可讲的 30～60 秒答案。每题先说结论，再说关键机制，最后主动补一个易错点或排查点；不会显得只会背定义。看到不认识的缩写，回到第一章的“缩写导航”查其全称与层次。

### A. Docker 与容器

#### 1. Docker 和虚拟机的区别是什么？

**答案：**

先讲边界：VM 虚拟硬件并带自己的 Guest OS / 内核，隔离更强但重；容器是共享宿主机内核的一组受限进程，启动更快、密度更高。再补取舍：运行不可信代码或多租户 Agent 时，容器外可再选择 microVM / Sandbox；代价是启动与资源开销，收益是更强的宿主机隔离。详细的 namespace、cgroups 与 RuntimeClass 已在前文基础部分说明。

---

#### 2. Docker 是怎么实现轻量级隔离的？

**答案：**

先给结论：容器仍是宿主机进程，**namespace 管“看见什么”，cgroups 管“最多用多少”**。追问时再举 PID / Network namespace 和 CPU / 内存配额的例子；安全边界还需叠加最小权限、seccomp、AppArmor / SELinux、只读文件系统和网络策略，不能只说“有容器就安全”。

---

#### 3. 镜像和容器的区别是什么？

**答案：**

镜像是不可变的只读运行模板；容器是镜像启动后的运行实例，带进程、网络和可写层。生产发布应尽量用镜像 digest 锁定版本，不只依赖可变 tag。

---

### B. Kubernetes 核心机制

#### 4. Pod 和容器的关系是什么？

**答案：**

Pod 是 K8s 的最小调度单位，不等于一个容器。一个 Pod 最常见是一个业务容器，也可以放主容器加 sidecar；同 Pod 内的容器共享一个 Pod IP 和网络命名空间，也可以挂同一个 Volume，所以适合表达“必须一起部署、一起协作”的进程组。

例如业务容器将日志写进 `emptyDir`，日志 sidecar 从同一目录采集；它们必须被调度到同一 Node，且可用 `localhost` 通信。不要把两个可独立发布、独立扩缩容的微服务为了“通信方便”塞进同一 Pod——那会把故障域和伸缩策略错误绑定。

---

#### 5. Pod 创建流程怎么讲？

**答案：**

以 Deployment 为例：用户通过 `kubectl apply` 把 YAML 交给 API Server；API Server 完成认证、RBAC 鉴权、准入和校验后，把声明持久化进 etcd。Deployment / ReplicaSet 控制器 watch 到期望副本后创建 Pending Pod。Scheduler 为没有 `nodeName` 的 Pod 按资源 request、亲和性、污点容忍等筛选打分，只把它绑定到合适的 Node。

目标 Node 的 kubelet watch 到这个绑定结果，准备 ConfigMap、Secret 和可能的 PVC；再通过 **CRI（Container Runtime Interface，容器运行时接口）**调用 containerd 等 runtime。runtime 创建 Pod sandbox 并调用 **CNI（Container Network Interface，容器网络接口）**插件配置网络；需要持久卷时，**CSI（Container Storage Interface，容器存储接口）**Driver 负责挂载。随后拉镜像、依次执行 init container、启动业务容器。readiness probe 成功后，EndpointSlice 才将该 Pod 作为 Service 的可用后端。

最重要的纠错：Scheduler 只“选 Node”，不创建容器；etcd 只“存状态”，不下发命令；kubelet 才是节点上的执行者。完整时序图、状态和排障树见第三章第 9 节。

---

#### 6. Deployment 和 Pod 的关系是什么？

**答案：**

Deployment 是上层的“期望状态和发布策略”，Pod 是真正运行容器、被调度到 Node 的单位，中间由 ReplicaSet 负责维持某一版本的副本数：`Deployment -> ReplicaSet -> Pod`。例如 Deployment 声明 `replicas: 3`，一个 Pod 崩掉后，ReplicaSet 会创建替代 Pod，让实际副本重新回到 3。

改镜像版本时，Deployment 会创建新的 ReplicaSet，按 `maxSurge`、`maxUnavailable` 等策略逐步扩新、缩旧，等待新 Pod readiness 成功再继续；因此滚动升级、暂停、回滚属于 Deployment 层。不要通过 SSH 到机器手动重启容器，这会让实际状态与声明状态漂移。

---

#### 7. Service 为什么存在？

**答案：**

Pod 会重建、扩缩容和迁移，Pod IP 与后端数量都可能变化；Service 用 label selector 选择一组 Pod，为调用方提供稳定的 DNS 名和虚拟 IP。只有通过 readiness 的后端才应被记录进 EndpointSlice，Service 的转发实现再从这些地址中选择目标。

因此 Service 解决“稳定访问这一类后端”，不是固定某一个 Pod；调用方应访问 `order-api` 这样的 Service 名，而不是写死 `10.x.x.x`。它也不等同于 Ingress：Service 是服务稳定入口，Ingress / Ingress Controller 处理外部 HTTP / HTTPS 的 host、path、TLS 路由。

---

#### 8. Service 常见类型有哪些？

**答案：**

- `ClusterIP`：集群内访问，默认类型
- `NodePort`：通过每台 Node 的某个端口暴露
- `LoadBalancer`：借助云负载均衡对外暴露
- `ExternalName`：把服务映射到外部 DNS 名称

选择思路：服务只供集群内调用时先选 `ClusterIP`；临时测试可使用 `NodePort`，但要承担端口暴露和节点地址管理；云上对外服务通常用 `LoadBalancer` 或 Ingress；需要给外部已有域名取一个集群内别名时选 `ExternalName`。`ExternalName` 只返回 DNS CNAME，不会自动产生 Pod 转发和健康检查。

---

#### 9. liveness / readiness / startup probe 的区别是什么？

**答案：**

- `liveness probe`：判断容器是不是“活着”，失败了会重启
- `readiness probe`：判断容器是不是“准备好接流量”，失败了会从 Service Endpoints 里摘掉
- `startup probe`：给慢启动应用兜底，启动成功前不执行 liveness / readiness

一句话：

> liveness 管“要不要重启”，readiness 管“要不要接流量”，startup 管“启动慢时别太早误杀”。

典型配置方式是：慢启动的 Java / 模型服务先配 startup probe；启动成功前，liveness 和 readiness 不会过早判失败。应用已经启动但依赖未就绪时，让 readiness 失败以摘流量，而不是让 liveness 失败反复重启。探针地址不要只写“进程存在”，更不要在每次探测里访问会导致级联压力的重依赖；它应能表达本服务是否真的可安全接流量。

---

#### 10. OOMKilled 和 CrashLoopBackOff 是什么？

**答案：**

- `OOMKilled`：容器因为内存超限被系统杀掉
- `CrashLoopBackOff`：容器反复启动、反复崩，K8s 进入退避重试状态

两者关系也不同：OOMKilled 是一次明确的内核内存杀进程原因；CrashLoopBackOff 是“容器连续启动失败后，kubelet 逐渐延长重试间隔”的状态，OOMKilled、配置错误、端口冲突、依赖不可用都可能导致它。

排查顺序是：`kubectl describe pod` 看状态、退出码和 Events；`kubectl logs <pod> --previous` 看上一次崩溃日志；再核对 `requests/limits`、启动命令、ConfigMap / Secret、依赖连通性与健康检查。不要把增加内存当成所有 CrashLoop 的解法；先确认真正的退出原因。

---

#### 11. requests 和 limits 是什么？

**答案：**

`requests` 是容器申请并让 Scheduler 计入调度的资源量；Node 没有足够可分配 request 时，Pod 会 Pending。`limits` 是运行时上限，kubelet 将它交给 runtime，再由 Linux cgroups（控制组）执行。CPU 用超通常表现为 throttling（被限速），内存超过 limit 则可能被内核 OOM Kill。

例如一个 Pod 的容器 `request: 500m, 512Mi`、`limit: 1 CPU, 1Gi`：Scheduler 至少要为它找到还剩半核、512Mi 可分配资源的 Node；运行中可以短时使用到一核和 1Gi，超过 CPU 上限会变慢，内存继续膨胀则有被杀风险。request 不是“保证永远拿到的 CPU”，limit 也不是“内存到了就一定优雅报错”。

---

#### 12. Namespace 是什么？是强隔离吗？

**答案：**

它是 Kubernetes API 对象的逻辑管理边界，可按团队、环境配合 ResourceQuota、RBAC 管理；不是 Linux namespace，也不单独构成强安全隔离。多租户还要结合权限、网络策略、Pod 安全和必要时的强隔离运行时。

---

#### 13. PV / PVC 的关系怎么讲？

**答案：**

PV（PersistentVolume，持久卷）是集群可提供的真实存储资源；PVC（PersistentVolumeClaim，持久卷声明）是应用提出的容量、访问模式、存储类型需求；Pod 引用 PVC，而不是直接操作底层云盘。StorageClass（存储类）规定动态供给策略，例如由哪个 CSI Driver 创建什么性能等级的盘。

可按“PVC 申请 → StorageClass / CSI 供给或匹配 PV → PVC 与 PV 绑定 → kubelet 挂到 Pod”讲。常见排障不是只看 Pod：PVC 一直 Pending 先查 StorageClass、容量和 CSI provisioner；PVC 已 Bound 但容器起不来，再查 attach / mount Events、节点与卷的限制、文件权限。

---

#### 14. CSI 是什么？

**答案：**

CSI 是 **Container Storage Interface（容器存储接口）**。CSI Driver 把 Kubernetes 的 PVC / PV 需求对接到具体后端，例如云盘、NFS、Ceph，并完成动态创建、附着、节点挂载等动作。它解决“存储如何进入 Pod”，不解决网络。

与其成对记忆的是 CNI：CNI 是 **Container Network Interface（容器网络接口）**，给 Pod 配 IP、路由和网络能力。口诀仍是：**CRI 起容器，CNI 配网络，CSI 挂存储。**

---

#### 15. kubelet、scheduler、kube-proxy、etcd 分别干什么？

**答案：**

- `kubelet`：每个 Node 上的节点代理，watch 分给本机的 Pod，通过 CRI 调 runtime 落实运行、执行探针并上报状态。
- `scheduler`：为还没有 `nodeName` 的 Pod 选择合适 Node，依据 request、调度约束等过滤与打分；它不启动容器。
- `kube-proxy`：维护 Service 到 EndpointSlice 后端的转发规则；不同代理模式底层实现可以不同。
- `etcd`：控制面强一致状态存储，保存 API 对象；不是镜像仓库，也不是业务服务直接做注册发现的数据库。

顺序要讲对：期望状态经 API Server 写入 etcd，Controller 造出 Pod，Scheduler 绑定 Node，kubelet 再落地运行；组件不应绕过 API Server 直接改 etcd。

---

### C. etcd 与一致性

#### 16. etcd 是什么？它在 Kubernetes 中是不是服务注册中心？

**答案：**

etcd 是 Kubernetes 的强一致 KV（Key-Value，键值）状态存储，保存 Pod、Node、Deployment、Service、Secret 等 API 对象的持久化状态。它不是给业务服务直接查询实例地址的服务注册中心：集群内服务发现通常靠 Service 和 CoreDNS（集群 DNS 服务），实际流量由 kube-proxy 或 CNI 网络实现转发。

正常路径是 Controller、Scheduler、kubelet 都通过 API Server 间接读写 etcd；业务程序或普通组件绕过 API Server 直接改 etcd，会跳过认证、鉴权、准入和对象校验，可能破坏控制面一致性。

#### 17. etcd 为什么偏 CP？Raft 写入如何成功？

**答案：**

网络分区时 etcd 宁可让少数派不能写，也不接受两边冲突的状态。Raft Leader 收到写请求后追加日志、复制给 Follower，获得多数派确认才提交并返回成功；3 节点需要 2 个确认。这就是它为控制面元数据选择一致性而牺牲少数派可写性的原因。

#### 18. Watch 和 Lease 分别用来做什么？有什么坑？

**答案：**

Watch 是订阅 key 或前缀变更的机制，适合控制器及时感知对象变化；Lease（租约）给临时 key 附加 TTL（Time To Live，生存时间），客户端不再 keepalive 时 key 自动过期，适合实例临时注册、选主等协调状态。

Watch 不是 MQ（Message Queue，消息队列）：消费者要保存 revision（版本序号），连接断开或历史被 compaction 清理后，要重新 List 得到当前全量状态，再从新的 revision 继续 Watch；业务处理也要幂等。Lease 也不是万能分布式锁，若持锁者对外部系统有副作用，仍应配合版本校验或 fencing token（栅栏令牌）防止旧持有者“复活后继续写”。

### D. Docker、Kubernetes、etcd 如何协作

#### 19. 部署一个 Docker 镜像到 Kubernetes，etcd 在哪里参与？

**答案：**

Docker / CI 先构建并推送镜像；`kubectl apply` 将“运行几个副本、使用哪个镜像、资源多少”的声明提交 API Server，API Server 才把它持久化到 etcd。控制器、Scheduler、kubelet 通过 API Server Watch 和更新对象，最终由 kubelet 拉镜像、调用 runtime 创建容器。etcd 保存的是“应运行什么、已绑定哪台节点、当前状态”，不负责存镜像或直接启动容器。

#### 20. etcd 短暂不可用，正在运行的服务会立刻全挂吗？

**答案：**

通常不会。已在节点上运行的容器可以继续执行；但控制面难以可靠读写状态，新的创建、调度、扩缩容、滚动发布和故障收敛会受影响。回答时再补充高可用做法：奇数节点组成多数派、跨故障域部署、低延迟磁盘、监控与定期快照恢复演练。

---

### E. AI / 云原生进阶

#### 21. GPU 在 Kubernetes 里怎么调度？和 CPU / 内存有什么区别？

**答案：**

GPU 通常以扩展资源暴露，例如 `nvidia.com/gpu`。节点先安装厂商驱动，再部署厂商的 **Device Plugin**：Plugin 向 kubelet 上报可用设备，kubelet 将其写入 Node 的 `status.allocatable`；Pod 在 `resources.limits` 中申请 GPU 后，Scheduler 像核算 CPU / 内存一样选择还有足够 GPU 的节点。真正落到节点时，Device Plugin 再参与设备分配，container runtime 把对应设备和运行所需配置交给容器。

```text
GPU 驱动 + Device Plugin
  -> Node allocatable: nvidia.com/gpu = 8
  -> Pod limits: nvidia.com/gpu = 1
  -> Scheduler 选有空闲 GPU 的节点
  -> kubelet / Device Plugin Allocate
  -> 容器拿到被分配的 GPU 设备
```

区别在于：CPU、内存是可按 `500m`、`512Mi` 分割和 cgroups 管控的可压缩资源；普通 GPU 是离散设备，Kubernetes 按整数个扩展资源核算，通常不能超卖，GPU 的 request 和 limit 必须相等。设备是否能分片、如何隔离算力和显存，取决于硬件与厂商插件，不是通用 cgroup CPU 配额。

- **MIG**：支持 MIG 的 GPU 可被划成带独立显存 / 计算配额的硬件实例，并由厂商插件作为不同资源暴露；调度的是某类 MIG 实例，而非整卡。
- **共享 GPU**：时间切片、显存配额或 MPS 等通常是厂商 / 平台扩展，能提高利用率，但隔离强度、监控和故障影响面要单独评估，不能把它当成标准 Kubernetes 的整卡独占。

官方 GPU 调度依赖 Device Plugin，并要求节点安装厂商驱动和对应插件。[Kubernetes GPU 调度](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

#### 22. 训练任务和在线推理任务在调度上有什么不同？

**答案：**

训练通常是长时间、多 GPU、强通信的批处理任务：要考虑同机 / 同机架亲和性、NVLink / RDMA 拓扑、数据位置，并使用 gang scheduling（成组资源一次满足才启动），否则 8 卡任务只拿到 4 卡会白占资源却无法有效训练。训练更重视吞吐、checkpoint、可恢复和排队公平性。

在线推理更看首 token / P99、可用性和弹性：请求可能很短、负载波动大，常按 QPS、队列长度、GPU 利用率或 KV Cache 水位扩缩；需要模型副本、路由、batching 和限流。它可以容忍小粒度共享或动态批处理，但必须为突发流量预留余量。简记为：**训练追求成组拿齐资源和总吞吐；推理追求低延迟、弹性与稳定服务。**

#### 23. GPU 很贵，怎样提高利用率？

**答案：**

先量化 GPU 利用率、显存利用率、排队时间、空洞资源和单位请求成本，不能只看“集群有多少卡”。常见组合是：

- 建立统一资源池，用队列和优先级让高优在线推理优先，低优训练 / 批任务可抢占或在空闲时运行；
- 以 GPU 型号、显存、网络拓扑做匹配和 bin packing，减少“任务要 80GB 显存却被放到不合适节点”的碎片；
- 推理侧做连续 / 动态 batching、模型复用和 KV Cache 管理，提高每次 kernel 执行的有效工作量；
- 对可分片硬件使用 MIG，或在风险可控时使用时间切片等共享方案；
- 设置闲置超时回收、checkpoint 与自动暂停，防止 notebook、实验任务长期占卡。

注意：利用率不是越高越好。在线推理把 GPU 压到接近 100% 往往会使排队和 P99 恶化，需为 SLO 留出容量。

#### 24. Serverless AI 为什么会冷启动？怎样优化？

**答案：**

普通函数冷启动已经包括调度、镜像拉取、容器 / runtime 启动；AI 还多了模型权重下载与加载、GPU 分配、CUDA / 推理引擎初始化、KV Cache 预留，因此首请求会明显慢于热实例。

优化要分别处理每一段：镜像做多阶段构建、减小层并使用就近 registry / 节点缓存；预拉镜像、预热节点；权重放本地高速缓存或共享只读缓存；保留少量 warm pool；让模型进程常驻并按模型规格分池；请求侧做排队、并发控制与动态 batch。代价是更高的空闲成本，所以应按流量周期、模型大小和首请求 SLO 设预热容量，而非无限保活。

#### 25. 镜像怎样构建、分发？几千台机器同时拉镜像怎么办？

**答案：**

CI 中以 Dockerfile / BuildKit 等构建不可变镜像，做依赖缓存、多阶段构建、漏洞扫描和签名；推送到镜像 Registry 后，以 tag 之外的 **digest** 标识确定版本。节点运行时按镜像层拉取并缓存，Deployment 再按策略创建 Pod。

大规模发布不能让数千节点同一时刻直连一个 Registry：会造成 Registry、跨机房带宽和镜像源限流雪崩。可组合使用区域 Registry mirror / pull-through cache、P2P 分发或节点级缓存；先用 DaemonSet 在目标节点预拉热点镜像；分批发布并限制并发拉取；对大镜像使用层复用和懒加载。每层还要校验 digest，失败指数退避，Registry、节点磁盘和拉取时延必须有监控。镜像是包含应用及依赖的可执行软件包，通常先推送 Registry 再由 Pod 引用。[Kubernetes Images](https://kubernetes.io/docs/concepts/containers/images/)

#### 26. 怎样做灰度发布和故障回滚？

**答案：**

先保证版本可追溯：镜像 digest、配置版本、数据库迁移和发布批次都要能定位。发布时可按 1% → 5% → 25% → 100% 的流量或实例比例逐步放量，使用 readiness 确保新 Pod 未就绪前不接流量；按错误率、P99、CPU / 内存、核心业务成功率和业务指标设自动暂停阈值。复杂场景可通过 Ingress / Service Mesh 做按用户、Header、地域或流量比例的 canary，而不是只按 Pod 数量。

发现异常先停止继续放量，切回稳定版本 / 流量；无状态服务可以回滚 Deployment 镜像。若涉及数据库，迁移应优先采用向后兼容的 expand → backfill → contract：新旧版本可同时读写，确认回滚窗口结束后再删除旧字段，不能把不可逆 DDL 和应用全量切换绑成一次操作。

#### 27. K8s 滚动更新时，正在运行的 Agent 服务怎样避免被打断？

先区分两种“服务”：HTTP 请求型 Agent（一次 Run 很短）主要需要连接 drain；任务 / Run 型 Agent（一次调查、代码审查、批量处理可能持续数分钟到数小时）还需要**任务可恢复**。只把 `terminationGracePeriodSeconds` 调大只能延后中断，不能保证节点故障、驱逐、进程 crash 时任务不丢。

```text
新版本 Pod 启动 -> startup / readiness 通过 -> 加入 Service
                                      |
Deployment 缩旧 Pod                  v
旧 Pod: 先标记 Draining / readiness=false
      -> EndpointSlice 摘流，不接新 Run
      -> preStop / SIGTERM：等待当前 HTTP 请求或 Run 到 checkpoint
      -> 到完成 / 期限：持久化 checkpoint、释放 lease、可重试任务重新入队
      -> 进程退出；超过 grace period 才会被强制终止
```

**应用层必须先设计任务状态，不把真相放在内存。**一个 Agent Run 至少持久化 `run_id`、状态、输入版本、当前 plan step、已调用工具的幂等键、checkpoint、worker lease / heartbeat。Worker 收到 drain 或 SIGTERM 后：停止领取新任务；将正在执行的 Run 标记为 `DRAINING`；在安全点保存 checkpoint；有外部副作用的工具按幂等键或 fencing token 防重复；超时未完成则由另一个 worker 在 lease 到期后接管或重试。这样即使不是正常滚动更新而是 Node 直接丢失，也能恢复。

Deployment 侧的典型策略：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0  # 先保证旧副本不减少到不可用
    maxSurge: 1        # 容量允许时，先多起一个新副本
minReadySeconds: 30    # 新 Pod 连续就绪一段时间再继续推进
template:
  spec:
    terminationGracePeriodSeconds: 600
    containers:
    - name: agent-worker
      lifecycle:
        preStop:
          httpGet: {path: /drain, port: 8080}
```

- `/drain` 的语义应是“拒绝新 Run、让 readiness 失败、等待 / checkpoint 旧 Run”，而不是只 `sleep 10`；应用也必须正确处理 `SIGTERM`。
- `maxUnavailable: 0` 与 `maxSurge: 1` 是可用性优先的例子，代价是发布时需要额外资源；GPU 或昂贵模型无法多起副本时，可能只能分批迁移、预热备用容量或使用流量切换。
- **PDB（PodDisruptionBudget，Pod 中断预算）**保护的是 drain、维护等通过 eviction API 发起的自愿中断；它不能阻止 Deployment 按自己的滚动策略替换 Pod，也不能抵御节点宕机。因此发布可用性主要看 Deployment 的 `maxUnavailable` 和 readiness，PDB 是额外保护而不是万能开关。
- 需要给 `/drain`、checkpoint 和终止过程监控：draining Run 数、剩余最长执行时间、被强杀数、恢复成功率、重复副作用数。超过发布窗口应自动暂停 rollout，而不是无限等。

面试回答：**K8s 负责先起新副本、摘旧流量、给旧 Pod 优雅退出时间；Agent 自己负责 Run 的持久状态、lease、checkpoint 和幂等恢复。两者缺一个，长任务都可能在发布或故障时丢失或重复执行。**

#### 28. 在 K8s 部署推理服务，健康检查怎么设计？

推理服务不能只做 `GET /health == 200`。进程端口活着时，模型可能仍在加载、权重损坏、GPU 未分配、显存耗尽、队列已经爆满，或者推理线程死锁。应该把 **启动完成、是否接流量、进程是否需要重启、容量是否饱和** 分成不同信号。

```text
启动阶段：镜像 -> 权重 / tokenizer -> CUDA / 引擎 -> 模型加载
                  | startup probe
                  v
可服务阶段：模型可用 + GPU / KV Cache 有余量 + 队列未过载
                  | readiness probe
                  v
运行阶段：主事件循环 / worker 仍有进展，进程没有卡死
                  | liveness probe
                  v
容量阶段：GPU 利用率、显存、队列、inflight、TTFT / P99
                  | 指标告警与扩缩容，不应靠 liveness 重启解决
```

| 检查 | 它回答的问题 | 合理检查内容 | 不要这样做 |
| --- | --- | --- | --- |
| `startupProbe`（启动探针） | 模型服务是否已完成一次性初始化？ | 权重 / tokenizer 已加载、推理引擎初始化成功、必要 GPU device 可见。 | 用过短 liveness 在大模型加载时反复杀掉 Pod。 |
| `readinessProbe`（就绪探针） | 这个实例现在能否安全接新请求？ | 模型已 ready、关键依赖可用、没有 drain、队列 / 并发 / KV Cache 未过载。 | 只要 TCP 端口通就接流量；或每 5 秒跑一次昂贵完整生成。 |
| `livenessProbe`（存活探针） | 进程是否卡死到应该重启？ | 主 loop heartbeat、线程池进展、内部死锁检测；失败阈值要容忍短暂 GC / GPU 抖动。 | 因下游短暂慢、队列满或单个请求超时就重启整个模型。 |
| 外部合成探测 + 指标 | 用户实际是否拿到正确且及时的推理？ | 小流量固定 prompt、错误率、TTFT（首 token 时间）、端到端 P99、GPU / 显存 / 队列。 | 把容量不足误判为“进程死了”，造成重启风暴。 |

一个简化的 Probe 配置如下；数值必须来自模型大小、冷启动 p99 和压测，下面只展示职责划分：

```yaml
containers:
- name: inference
  startupProbe:
    httpGet: {path: /health/startup, port: 8080}
    periodSeconds: 5
    failureThreshold: 120 # 最长约 10 分钟，覆盖模型加载窗口
  readinessProbe:
    httpGet: {path: /health/ready, port: 8080}
    periodSeconds: 5
    failureThreshold: 2
  livenessProbe:
    httpGet: {path: /health/live, port: 8080}
    periodSeconds: 10
    failureThreshold: 3
```

`/health/ready` 最好返回结构化原因，例如 `model_not_loaded`、`draining`、`queue_over_limit`、`gpu_unavailable`，便于 Events、日志与自动化决策。GPU 驱动错误、Xid、节点温度和设备分配等通常还要从 Device Plugin / Node 监控采集；Kubernetes 探针只看到容器内进程，不能代替节点和 GPU 监控。

**和发布结合起来看**：新推理 Pod 只有 startup 和 readiness 成功才进入 Service；旧 Pod 收到 drain 后 readiness 立即失败、停止收新请求，剩余请求在终止宽限期内完成或按幂等键重试。若 readiness 因“过载”失败，要有上游排队、限流、扩容或降级，不要让所有实例同时被摘流量造成雪崩。

更多 Agent 长任务的 checkpoint、死循环与运行治理见 [Agent_场景题_面经合集.md](../ai-tools/Agent_场景题_面经合集.md)。

---

## 八、最近公开面经里，大厂常问哪些 Docker / K8s / etcd 点？

根据近期公开可见的牛客内容，重复出现比较多的是：

1. **Docker 隔离原理**：`namespace` / `cgroup`
2. **Pod 创建流程**
3. **K8s 核心组件作用**：`apiserver` / `scheduler` / `kubelet` / `etcd` / `kube-proxy`
4. **Deployment 怎么扩缩容 / 滚动更新**
5. **Service 类型与负载均衡原理**
6. **Pod 生命周期**
7. **PV / PVC / CSI**
8. **调度相关问题**：怎么让 Pod 调到指定节点
9. **容器云 / 云原生方向会追问 etcd、scheduler、kubelet 流程**

这说明大厂一面常常不是考你“会不会敲 `kubectl`”，而是看你知不知道：

> **这些对象到底是怎么协作起来，把一个容器从 YAML 变成线上运行实例的。**

---

## 九、你明天要面云原生 / Infra 组，最该背的 14 个题

1. Docker 和虚拟机区别
2. Docker 怎么实现隔离
3. 镜像和容器区别
4. Pod 是什么，为什么不是直接调度容器
5. Pod 创建流程
6. Deployment / ReplicaSet / Pod 关系
7. Service 为什么存在，常见类型有哪些
8. Ingress 是什么
9. requests / limits 是什么
10. liveness / readiness / startup probe 区别
11. PV / PVC / StorageClass 关系
12. OOMKilled / CrashLoopBackOff 怎么排查
13. etcd 为什么偏 CP，Raft 写入怎样提交
14. Docker、Kubernetes、etcd 怎样协作部署一个服务

如果你把这 14 个讲清楚，已经能覆盖很多容器 / K8s / etcd 的初中级面试。

---

## 十、来源（基础定义 + 近期公开面经）

### 官方文档 / 一手资料

1. Docker 官方文档（Overview）  
   https://docs.docker.com/get-started/docker-overview/
2. Kubernetes Pod Lifecycle  
   https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
3. Kubernetes Service  
   https://kubernetes.io/docs/concepts/services-networking/service/index.html
4. Kubernetes Liveness / Readiness / Startup Probes  
   https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
5. Kubernetes Persistent Volumes  
   https://kubernetes.io/docs/concepts/storage/persistent-volumes/
6. Kubernetes Deployments  
   https://v1-33.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/
7. Kubernetes Resource Management  
   https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
8. Kubernetes GPU Scheduling  
   https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
9. Kubernetes Container Runtime Interface  
   https://kubernetes.io/docs/concepts/containers/cri/
10. Kubernetes RuntimeClass  
    https://kubernetes.io/docs/concepts/containers/runtime-class/
11. Kubernetes Cluster Architecture  
    https://kubernetes.io/docs/concepts/architecture/
12. Kubernetes API Server Bypass Risks（etcd 访问边界）  
    https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/
13. etcd API Guarantees  
    https://etcd.io/docs/v3.5/learning/api_guarantees/
14. etcd Distributed Coordination  
   https://etcd.io/docs/v3.6/learning/why/
15. Kubernetes Controllers（控制器如何通过 API 收敛状态）  
   https://kubernetes.io/docs/concepts/architecture/controller/
16. Kubernetes Compute / Storage / Networking Extensions（CNI、CSI 等扩展）  
   https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/
17. Kubernetes Persistent Volumes（PV、PVC、StorageClass）  
   https://kubernetes.io/docs/concepts/storage/persistent-volumes/
18. etcd Disaster Recovery（快照、恢复与 quorum）  
   https://etcd.io/docs/v3.7/op-guide/recovery/
19. Kubernetes Rolling Update（`maxUnavailable` / `maxSurge`）  
   https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
20. Kubernetes Disruptions（PDB、优雅终止与滚动更新边界）  
   https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
21. [Docker：What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
22. [Docker Engine Security（namespace、cgroup、seccomp 等边界）](https://docs.docker.com/engine/security/)
23. [Linux Kernel：Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
24. [Kubernetes Pods（Pod 共享上下文与网络 namespace）](https://kubernetes.io/docs/concepts/workloads/pods/)
25. [Kubernetes cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/)
26. [Kubernetes Container Runtimes（CRI、cgroup driver、dockershim）](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
27. [Kubernetes：Share Process Namespace between Containers in a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)

### 公开面经 / 公开讨论（牛客为主）

1. Docker + Kubernetes(k8s) + Serverless详解  
   https://www.nowcoder.com/discuss/741063580249755648
2. go开 云原生开发  
   https://www.nowcoder.com/feed/main/detail/86961dcd11f645f3ba5faa6a5aee3f75?sourceSSR=post
3. 滴滴后端 oc 面经总结（云原生意向，话题页摘要）  
   https://www.nowcoder.com/creation/subject/26a863bcecd54fa3a2fed817c6029659?entranceType_var=%E5%86%85%E5%AE%B9%E6%9D%A1%E7%9B%AE
4. 面经｜快手云原生凉经（一面、二面挂）  
   https://www.nowcoder.com/discuss/413444430189834240
5. 面经 | 快手容器云一面  
   https://www.nowcoder.com/feed/main/detail/b99dfe1bef5249618926633c98097f59
6. 小黑盒开发一面 40min（含 Docker 隔离追问）  
   https://www.nowcoder.com/feed/main/detail/b0a6dae1887d48e88dddcd3daeec46d0
7. Service——负载均衡机制实现原理  
   https://www.nowcoder.com/discuss/490142295887486976
8. Docker——容器深入理解  
   https://www.nowcoder.com/feed/main/detail/1e6d63166a6849c9ae6dbecd5fb7972f

### 关于小红书

- 我尝试按公开 Web 检索 Docker / K8s / 云原生面经，但小红书公开搜索结果长期受登录墙影响，难以稳定抓取正文。
- 所以这份文档的“常问问题”主要以**牛客公开可验证内容 + 官方文档**为主。

---

## 十一、最后给你的学习顺序建议（按新手版）

### 第 1 步：先背清对象关系

```text
Docker: Image -> Container
K8s: Deployment -> ReplicaSet -> Pod -> Service -> Ingress
存储: PV <-> PVC
```

### 第 2 步：先会讲，不要求一开始就会配

先会说清：

- Pod 是什么
- Deployment 是什么
- Service 为什么存在
- Docker 为什么比虚拟机轻
- namespace/cgroup 干什么

### 第 3 步：再去补命令和排障

- `kubectl get/describe/logs/exec`
- `kubectl get pods -A`
- `kubectl describe pod xxx`
- `kubectl logs xxx`
- 看状态：`Pending` / `Running` / `CrashLoopBackOff` / `OOMKilled`

### 第 4 步：最后补进阶原理

- scheduler 调度
- kube-proxy 转发
- CSI
- CNI
- Ingress Controller
- etcd

如果时间有限，先过第二章的“容器、镜像、隔离”，再过第三章的“Pod、Deployment、Service、Pod 创建链路、资源隔离”，随后读第四、五章，确保能讲清 etcd 与控制面如何协作；最后用第七章 B～D 组自测。面 AI / Infra 岗再补 E 组的 GPU、冷启动、镜像分发和发布回滚。
