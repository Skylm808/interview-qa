# Docker / Kubernetes（K8s）基础与面经合集（答案版）

> 适用人群：刚接触容器 / K8s 的后端同学。  
> 你的当前状态我按这个假设来整理：**会用 `kubectl` 连公司集群、能看部署和服务，但还没有系统学过 Docker / K8s 原理。**  
> 目标：先建立整体地图，再补大厂高频面试题。

---

## 一、阅读顺序：先 Docker，后 Kubernetes，再做面试题

```text
Docker：应用 + Dockerfile -> Image -> Registry -> Container -> Volume
K8s：Node / Control Plane -> Pod -> Deployment -> Service / Ingress
                                 -> ConfigMap / Secret / PV / PVC
                                 -> Scheduler / kubelet / CNI / CSI
```

- **Docker** 解决“应用如何连同依赖被一致地构建、分发和运行”。
- **Kubernetes（K8s）** 解决“许多容器如何部署、调度、扩缩容、联网、存储与自愈”。

因此不要先背 K8s 名词：先弄清镜像和容器，再理解 Pod 为什么是调度单位，最后才看 Deployment、Service 和节点组件。

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

这是非常高频的大厂问题。

#### `namespace`

`namespace` 负责“**看起来隔离**”。

常见的 namespace：

- `PID namespace`：进程号隔离
- `NET namespace`：网络隔离
- `MNT namespace`：挂载点隔离
- `IPC namespace`：进程间通信隔离
- `UTS namespace`：主机名隔离
- `USER namespace`：用户/权限映射

#### `cgroup`

`cgroup`（`control groups`）负责“**资源限制**”。

比如限制：

- CPU
- 内存
- 磁盘 IO
- 进程数

所以最常见一句话答法是：

> **namespace 负责隔离视图，cgroup 负责限制资源。**

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

---

### 2A. etcd 是不是分布式？它在 K8s 里是做服务发现的吗？

可以先记一句：

> **etcd 本身就是分布式强一致 KV（Key-Value，键值）存储，不是项目大才叫分布式；在 K8s 里，它更像“集群总账本”，不是业务直接使用的服务发现组件。**

更准确地说：

- 小环境里 etcd 可能单节点部署
- 大环境里常见 3 节点 / 5 节点部署
- 但它的本质定位一直都是分布式一致性存储

在 Kubernetes 里，etcd 主要保存：

- Pod
- Node
- Deployment
- Service
- Endpoint / EndpointSlice
- ConfigMap / Secret
- 其他集群状态

所以它和服务发现的关系是：

- **服务发现依赖的状态数据会存在 etcd 里**
- 但真正更直接负责服务发现的是：
  - `Service`
  - `CoreDNS`
  - `kube-proxy`

一句压缩版：

> **etcd 是 K8s 的分布式状态存储，不是直接做业务服务发现的；服务发现依赖的状态数据会存到 etcd 里。**

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

---

### 3. Pod 是什么？为什么不是直接调度容器？

Pod 是 K8s 里的**最小调度单元**。

它不是“某个容器”，而是“**一组共享网络和存储的容器**”。

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

---

### 7. ConfigMap 和 Secret 是什么？

- `ConfigMap`：放普通配置
- `Secret`：放敏感信息，比如密码、token、证书

它们通常会被挂到：

- 环境变量
- 文件

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

### 9. 一个 Pod 怎样从 YAML 变成真正运行的容器？

这条链路把前面的组件串起来：

```text
客户端 / Controller
  -> API Server（认证、鉴权、准入校验）
  -> etcd（持久化期望状态；Pod 尚未绑定 Node）
  -> Scheduler（按 requests、可用资源、亲和/反亲和、污点容忍等过滤和打分）
  -> 写回 Pod.spec.nodeName
  -> 目标 kubelet（watch 到分配给本机的 Pod）
  -> CRI / container runtime（创建 Pod sandbox、拉镜像、创建容器）
  -> CNI（创建网络命名空间、配置网卡 / IP / 路由）
  -> CSI（若使用 PVC，则挂载存储卷）
  -> 探针通过；readiness 就绪后才接入 Service 流量
```

- `scheduler` 只选 Node，不负责拉镜像或启动容器；节点执行者是 `kubelet`。
- `CRI` 是 kubelet 与 containerd、CRI-O 等 container runtime 的接口。
- `CNI` 管网络，`CSI` 管存储；CSI 仅在需要卷时参与，二者不能混为一谈。

### 10. Pod 的 CPU、内存怎样隔离？

完整路径是：**`requests / limits → scheduler → kubelet / runtime → Linux cgroups`。**

- `requests` 是调度预留量。Scheduler 将一个 Pod 中各容器的 request 汇总，只有 Node 剩余可分配资源足够才会放置它；CPU request 在竞争时通常对应更高的 CPU 时间权重。
- `limits` 是运行上限。kubelet 将数值交给 runtime，Linux 节点一般由 cgroups 执行：CPU 到上限会被 throttling；内存超过 limit 时，内存压力下可能被 OOM Kill。
- 例如 `request: 500m / 512Mi`、`limit: 1 CPU / 1Gi`：调度需要节点至少有 0.5 核和 512Mi 余量；运行时超过 1 CPU 会被限速，内存失控则可能被杀后重启。

这里的 Linux `namespace` 解决“进程能看见哪些 PID、网络、挂载点”；`cgroups` 解决“最多能用多少 CPU、内存、IO”。不要把它和 Kubernetes Namespace（API 对象的逻辑管理边界）混为一谈。

### 11. CNI、CSI 和 RuntimeClass 分别解决什么？

- `CNI`：给 Pod 配网络，如网卡、IP、路由与网络策略落地。
- `CSI`：让 PVC 对接云盘、NFS、Ceph 等存储，并完成挂载。
- `RuntimeClass`：为 Pod 选择不同 container runtime 配置；需要更强隔离的工作负载可以选择基于轻量虚拟化或 sandbox 的运行时，调度器也能计入额外开销。

它们分别位于网络、存储和运行时隔离三层，不能把“装了 CSI”理解成网络打通，或把“用了容器”理解成天然强安全隔离。

---

## 四、你现在会用 kubectl，但要知道它到底在做什么

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

## 五、高频面试题：用基础知识组织成短答

> 第二、三章负责把概念讲透；本章不再重复教材，而是给出面试时的回答顺序、易错点和进阶追问。复习时先读基础，再用这一章自测。

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

Pod 是最小调度单元，通常包含一个主容器，也可包含强关联 sidecar；它们共享 Pod 网络和可共享 Volume。回答时说清“Pod 是调度边界，不等于一个容器”即可。

---

#### 5. Pod 创建流程怎么讲？

**答案：**

按“API Server → etcd → Scheduler → kubelet → runtime → CNI / CSI → probes”复述即可。最容易失分的是说 Scheduler 在节点上创建容器；实际上 Scheduler 只绑定 Node，kubelet 通过 CRI 落实运行。

---

#### 6. Deployment 和 Pod 的关系是什么？

**答案：**

Deployment 声明副本数与发布策略，底层通过 ReplicaSet 维持 Pod 数量；Pod 才是实际承载容器的单位。回答时可补充滚动升级、暂停与回滚都是 Deployment 层的职责。

---

#### 7. Service 为什么存在？

**答案：**

Pod IP 与副本数会变化，Service 以 label selector 选择一组就绪 Pod，并提供稳定的虚拟 IP / DNS 名称；它解决“稳定访问谁”，不直接替代 Ingress 的 HTTP 路由能力。

---

#### 8. Service 常见类型有哪些？

**答案：**

- `ClusterIP`：集群内访问，默认类型
- `NodePort`：通过每台 Node 的某个端口暴露
- `LoadBalancer`：借助云负载均衡对外暴露
- `ExternalName`：把服务映射到外部 DNS 名称

---

#### 9. liveness / readiness / startup probe 的区别是什么？

**答案：**

- `liveness probe`：判断容器是不是“活着”，失败了会重启
- `readiness probe`：判断容器是不是“准备好接流量”，失败了会从 Service Endpoints 里摘掉
- `startup probe`：给慢启动应用兜底，启动成功前不执行 liveness / readiness

一句话：

> liveness 管“要不要重启”，readiness 管“要不要接流量”，startup 管“启动慢时别太早误杀”。

---

#### 10. OOMKilled 和 CrashLoopBackOff 是什么？

**答案：**

- `OOMKilled`：容器因为内存超限被系统杀掉
- `CrashLoopBackOff`：容器反复启动、反复崩，K8s 进入退避重试状态

排查时先看：

- `kubectl describe pod`
- `kubectl logs`
- 资源限制 `requests/limits`
- 程序启动参数、配置、依赖是否正常

---

#### 11. requests 和 limits 是什么？

**答案：**

按“request 决定能否调度，limit 决定运行上限，cgroups 真正执行”回答。CPU 超限通常是 throttling，内存超限风险是 OOM Kill；不要把 Kubernetes Namespace 误说成 Linux namespace 或强安全隔离。数值例子和完整链路见第三章第 10 节。

---

#### 12. Namespace 是什么？是强隔离吗？

**答案：**

它是 Kubernetes API 对象的逻辑管理边界，可按团队、环境配合 ResourceQuota、RBAC 管理；不是 Linux namespace，也不单独构成强安全隔离。多租户还要结合权限、网络策略、Pod 安全和必要时的强隔离运行时。

---

#### 13. PV / PVC 的关系怎么讲？

**答案：**

PV 是集群可供给的存储资源，PVC 是工作负载的存储申请，Pod 引用 PVC 而非直接绑定底层盘。StorageClass 可提供动态供给策略；三者的类比和基础定义见第三章第 8 节。

---

#### 14. CSI 是什么？

**答案：**

CSI 是容器存储接口，Driver 将 PVC 的请求落到具体云盘、NFS、Ceph 等存储，并负责挂载。它与负责 Pod 网络的 CNI 是两条不同链路，见第三章第 11 节。

---

#### 15. kubelet、scheduler、kube-proxy、etcd 分别干什么？

**答案：**

- `kubelet`：节点执行者，负责让本机 Pod 实际运行并上报状态。
- `scheduler`：为未绑定 Node 的 Pod 选择合适节点。
- `kube-proxy`：维护 Service 转发相关规则。
- `etcd`：控制面的一致性状态存储，不是直接给业务服务做注册发现。

顺序要讲对：期望状态经 API Server 写入 etcd，Scheduler 绑定 Node，kubelet 再落地运行。

---

### C. AI / 云原生进阶

#### 16. GPU 在 Kubernetes 里怎么调度？和 CPU / 内存有什么区别？

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

#### 17. 训练任务和在线推理任务在调度上有什么不同？

**答案：**

训练通常是长时间、多 GPU、强通信的批处理任务：要考虑同机 / 同机架亲和性、NVLink / RDMA 拓扑、数据位置，并使用 gang scheduling（成组资源一次满足才启动），否则 8 卡任务只拿到 4 卡会白占资源却无法有效训练。训练更重视吞吐、checkpoint、可恢复和排队公平性。

在线推理更看首 token / P99、可用性和弹性：请求可能很短、负载波动大，常按 QPS、队列长度、GPU 利用率或 KV Cache 水位扩缩；需要模型副本、路由、batching 和限流。它可以容忍小粒度共享或动态批处理，但必须为突发流量预留余量。简记为：**训练追求成组拿齐资源和总吞吐；推理追求低延迟、弹性与稳定服务。**

#### 18. GPU 很贵，怎样提高利用率？

**答案：**

先量化 GPU 利用率、显存利用率、排队时间、空洞资源和单位请求成本，不能只看“集群有多少卡”。常见组合是：

- 建立统一资源池，用队列和优先级让高优在线推理优先，低优训练 / 批任务可抢占或在空闲时运行；
- 以 GPU 型号、显存、网络拓扑做匹配和 bin packing，减少“任务要 80GB 显存却被放到不合适节点”的碎片；
- 推理侧做连续 / 动态 batching、模型复用和 KV Cache 管理，提高每次 kernel 执行的有效工作量；
- 对可分片硬件使用 MIG，或在风险可控时使用时间切片等共享方案；
- 设置闲置超时回收、checkpoint 与自动暂停，防止 notebook、实验任务长期占卡。

注意：利用率不是越高越好。在线推理把 GPU 压到接近 100% 往往会使排队和 P99 恶化，需为 SLO 留出容量。

#### 19. Serverless AI 为什么会冷启动？怎样优化？

**答案：**

普通函数冷启动已经包括调度、镜像拉取、容器 / runtime 启动；AI 还多了模型权重下载与加载、GPU 分配、CUDA / 推理引擎初始化、KV Cache 预留，因此首请求会明显慢于热实例。

优化要分别处理每一段：镜像做多阶段构建、减小层并使用就近 registry / 节点缓存；预拉镜像、预热节点；权重放本地高速缓存或共享只读缓存；保留少量 warm pool；让模型进程常驻并按模型规格分池；请求侧做排队、并发控制与动态 batch。代价是更高的空闲成本，所以应按流量周期、模型大小和首请求 SLO 设预热容量，而非无限保活。

#### 20. 镜像怎样构建、分发？几千台机器同时拉镜像怎么办？

**答案：**

CI 中以 Dockerfile / BuildKit 等构建不可变镜像，做依赖缓存、多阶段构建、漏洞扫描和签名；推送到镜像 Registry 后，以 tag 之外的 **digest** 标识确定版本。节点运行时按镜像层拉取并缓存，Deployment 再按策略创建 Pod。

大规模发布不能让数千节点同一时刻直连一个 Registry：会造成 Registry、跨机房带宽和镜像源限流雪崩。可组合使用区域 Registry mirror / pull-through cache、P2P 分发或节点级缓存；先用 DaemonSet 在目标节点预拉热点镜像；分批发布并限制并发拉取；对大镜像使用层复用和懒加载。每层还要校验 digest，失败指数退避，Registry、节点磁盘和拉取时延必须有监控。镜像是包含应用及依赖的可执行软件包，通常先推送 Registry 再由 Pod 引用。[Kubernetes Images](https://kubernetes.io/docs/concepts/containers/images/)

#### 21. 怎样做灰度发布和故障回滚？

**答案：**

先保证版本可追溯：镜像 digest、配置版本、数据库迁移和发布批次都要能定位。发布时可按 1% → 5% → 25% → 100% 的流量或实例比例逐步放量，使用 readiness 确保新 Pod 未就绪前不接流量；按错误率、P99、CPU / 内存、核心业务成功率和业务指标设自动暂停阈值。复杂场景可通过 Ingress / Service Mesh 做按用户、Header、地域或流量比例的 canary，而不是只按 Pod 数量。

发现异常先停止继续放量，切回稳定版本 / 流量；无状态服务可以回滚 Deployment 镜像。若涉及数据库，迁移应优先采用向后兼容的 expand → backfill → contract：新旧版本可同时读写，确认回滚窗口结束后再删除旧字段，不能把不可逆 DDL 和应用全量切换绑成一次操作。

---

## 六、最近公开面经里，大厂常问哪些 Docker / K8s 点？

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

## 七、你明天要面云原生 / Infra 组，最该背的 12 个题

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

如果你把这 12 个讲清楚，已经能覆盖很多容器 / K8s 初中级面试。

---

## 八、来源（基础定义 + 近期公开面经）

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

## 九、最后给你的学习顺序建议（按新手版）

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

如果时间有限，先过第二章的“容器、镜像、隔离”，再过第三章的“Pod、Deployment、Service、Pod 创建链路、资源隔离”，最后用第五章 B 组的 4～15 题自测；面 AI / Infra 岗再补 C 组的 GPU、冷启动、镜像分发和发布回滚。
