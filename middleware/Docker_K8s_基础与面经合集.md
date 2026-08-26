# Docker / Kubernetes（K8s）基础与面经合集（答案版）

> 适用人群：刚接触容器 / K8s 的后端同学。  
> 你的当前状态我按这个假设来整理：**会用 `kubectl` 连公司集群、能看部署和服务，但还没有系统学过 Docker / K8s 原理。**  
> 目标：先建立整体地图，再补大厂高频面试题。

---

## 一、先建立最小认知地图

### 1. 先一句话理解 Docker 和 K8s

- **Docker**：解决“应用怎么打包、怎么在不同环境里一致运行”。
- **Kubernetes / K8s**：解决“很多容器怎么批量部署、调度、扩缩容、服务发现和自愈”。

你可以先把它们理解成：

```text
Docker = 把应用装进标准集装箱
K8s    = 管很多集装箱的大型调度系统
```

---

### 2. 你现在最容易混淆的几个概念

| 概念               | 一句话理解                           |
| ---------------- | ------------------------------- |
| **镜像 Image**     | 应用运行模板，里面有代码、运行时、依赖、配置          |
| **容器 Container** | 镜像运行起来后的实例                      |
| **仓库 Registry**  | 存镜像的地方，比如 Docker Hub、Harbor     |
| **Pod**          | K8s 里最小调度单元，通常装一个主容器，也可以多个容器一起跑 |
| **Deployment**   | 管一组 Pod，负责副本数、滚动更新、回滚           |
| **Service**      | 给 Pod 提供稳定访问入口                  |
| **Ingress**      | 负责外部 HTTP/HTTPS 流量进集群后的路由       |
| **Node**         | K8s 集群中的机器                      |
| **Namespace**    | 资源隔离视图，不是强安全边界                  |
| **PV / PVC**     | 持久化存储资源和存储申请单                   |

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

## 五、大厂很爱问的 Docker / K8s 问题（适合新手版）

### 1. Docker 和虚拟机的区别是什么？

**答案：**

虚拟机是虚拟出一整套硬件和 Guest OS，隔离强但开销大；容器本质上还是宿主机上的进程，主要通过 namespace 和 cgroup 做隔离与资源限制，启动快、资源利用率高。\
一句话就是：**虚拟机隔离的是机器，容器隔离的是进程。**

---

### 2. Docker 是怎么实现轻量级隔离的？

**答案：**

核心靠两类 Linux 机制：

- `namespace`：隔离进程视图，比如 PID、网络、挂载点、用户空间
- `cgroup`：限制资源，比如 CPU、内存、磁盘 IO

所以标准答法是：

> Docker 容器本质上是宿主机上的一组特殊进程，namespace 负责隔离，cgroup 负责资源限制。

---

### 3. 镜像和容器的区别是什么？

**答案：**

镜像是只读模板，容器是镜像运行起来后的实例。镜像里有应用和依赖，容器里则有运行时状态和可写层。

---

### 4. Pod 和容器的关系是什么？

**答案：**

Pod 是 K8s 最小调度单元，里面可以有一个或多个容器。多个容器共享同一个网络命名空间和 Volume，所以适合把强关联组件放在一个 Pod 里运行。

---

### 5. Pod 创建流程怎么讲？

**答案：**

最简化版本你可以这样答：

1. 用户提交 YAML 给 `kube-apiserver`
2. `apiserver` 写入 `etcd`
3. `scheduler` 发现有新的 Pod 未绑定节点，选择一个合适的 Node
4. 目标节点上的 `kubelet` 发现这个 Pod 被分配给自己
5. `kubelet` 调用容器运行时拉镜像、创建容器、挂载卷、配置网络
6. Pod 进入运行状态，并持续上报状态

这已经够应对大多数一面。

---

### 6. Deployment 和 Pod 的关系是什么？

**答案：**

Deployment 不直接跑业务，而是声明“我希望有多少个 Pod、怎么升级、怎么回滚”；底层由 ReplicaSet 帮它维持副本数。  
所以一般是：**Deployment 管副本和发布，Pod 真正承载容器。**

---

### 7. Service 为什么存在？

**答案：**

因为 Pod 是不稳定的，IP 会变、数量会变，不能直接依赖 Pod IP。Service 用来给一组 Pod 提供稳定的访问入口和服务发现能力。

---

### 8. Service 常见类型有哪些？

**答案：**

- `ClusterIP`：集群内访问，默认类型
- `NodePort`：通过每台 Node 的某个端口暴露
- `LoadBalancer`：借助云负载均衡对外暴露
- `ExternalName`：把服务映射到外部 DNS 名称

---

### 9. liveness / readiness / startup probe 的区别是什么？

**答案：**

- `liveness probe`：判断容器是不是“活着”，失败了会重启
- `readiness probe`：判断容器是不是“准备好接流量”，失败了会从 Service Endpoints 里摘掉
- `startup probe`：给慢启动应用兜底，启动成功前不执行 liveness / readiness

一句话：

> liveness 管“要不要重启”，readiness 管“要不要接流量”，startup 管“启动慢时别太早误杀”。

---

### 10. OOMKilled 和 CrashLoopBackOff 是什么？

**答案：**

- `OOMKilled`：容器因为内存超限被系统杀掉
- `CrashLoopBackOff`：容器反复启动、反复崩，K8s 进入退避重试状态

排查时先看：

- `kubectl describe pod`
- `kubectl logs`
- 资源限制 `requests/limits`
- 程序启动参数、配置、依赖是否正常

---

### 11. requests 和 limits 是什么？

**答案：**

- `requests`：调度时保证给你的最小资源
- `limits`：你最多能用到的上限资源

调度器主要参考 request，运行时限制主要看 limit。

---

### 12. Namespace 是什么？是强隔离吗？

**答案：**

Namespace 是 K8s 里的资源隔离视图，用来把同一集群里的资源按环境、团队、项目分开。它是很重要的管理边界，但一般不应简单理解成“强安全隔离”。

---

### 13. PV / PVC 的关系怎么讲？

**答案：**

PV 是真实存储资源，PVC 是存储申请。Pod 通常不是直接绑 PV，而是声明 PVC，再由系统把 PVC 和合适的 PV 绑定。

---

### 14. CSI 是什么？

**答案：**

`CSI` 全称是 `Container Storage Interface`，中文可以理解成**容器存储接口标准**。它让 K8s 可以通过统一接口对接不同存储系统。  
面试里你不用讲太深，知道：**PV/PVC 背后的很多现代存储接入都靠 CSI Driver。**

---

### 15. kubelet、scheduler、kube-proxy、etcd 分别干什么？

**答案：**

- `kubelet`：节点上的执行管家
- `scheduler`：给 Pod 选 Node
- `kube-proxy`：维护 Service 转发规则
- `etcd`：保存集群状态

这是非常高频的基础题。

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

如果你现在时间有限，优先把 **第五章的 15 个题 + 第七章的 12 个高频题** 过一遍，收益最高。
