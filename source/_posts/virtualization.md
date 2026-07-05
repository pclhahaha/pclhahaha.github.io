---
title: "虚拟化与容器化: 从 Hypervisor 到 K8s"
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 虚拟化
  - Docker
  - Kubernetes
  - 容器
  - 云原生
categories:
  - 基础设施
---
# 虚拟化与容器化：从 Hypervisor 到 Kubernetes 的深度解析

虚拟化能够充分利用硬件资源，在同一套硬件上运行多个虚拟机相比只运行一个操作系统显然更具经济优势。在此基础上，云计算产业得以快速发展。下表列出了虚拟化的主要类型：

| 虚拟化类型     | 特点                                                         |
| -------------- | ------------------------------------------------------------ |
| 桌面虚拟化     | Virtual desktop infrastructure 服务端提供虚拟化桌面、Local desktop virtualization 本地虚拟化不同操作系统桌面 |
| 网络虚拟化     | software-defined networking (SDN)、network function virtualization (NFV) |
| 存储虚拟化     | 云存储                                                       |
| 数据虚拟化     | 将应用层与数据存储层间的传输透明化，应用层不用关心底层是如何存储的 |
| 数据中心虚拟化 | 可以在一个数据中心上虚拟化出多个数据中心                     |
| CPU虚拟化      | 可以理解为 Hypervisor 虚拟化                                 |
| GPU虚拟化      | Pass-through GPUs、Shared vGPUs                              |

## 一、Hypervisor 虚拟化

Hypervisor 是介于虚拟机操作系统与硬件层之间的一层薄薄的软件层，是对硬件层的虚拟化，它允许多个操作系统共享相同的硬件资源。运行在 Hypervisor 之上的操作系统就是 VM (virtual machine)，而 Hypervisor 又称为 VMM (virtual machine monitor)。

![显示常用硬件虚拟化的简单分层架构](/infra/virtualization/figure1.gif)

### 1.1 Type 1 vs. Type 2

Hypervisor 分为如下两类：

| 类型   | 特点                                                         | 优势                         | 劣势                                                         |
| ------ | ------------------------------------------------------------ | ---------------------------- | ------------------------------------------------------------ |
| Type 1 | 又称为 bare-metal hypervisors，直接运行在底层硬件之上，与 CPU，内存，磁盘等交互 | 性能高，安全性高             | 需要独立的管理机器来管理虚拟机以及硬件                       |
| Type 2 | 作为一个应用运行在底层操作系统上，通常用于个人 PC 而不是服务器 | 便于个人开发者安装多个虚拟机 | 由于 Hypervisor 和硬件中间隔了一层操作系统，存在延迟；底层操作系统一旦被侵入，上层的虚拟机也容易被侵入 |

**架构示意（文字描述）：**
- **Type 1**：`硬件 → Hypervisor → VM1/VM2... → 各VM内Guest OS + App`
- **Type 2**：`硬件 → Host OS → Hypervisor应用 → VM1/VM2... → 各VM内Guest OS + App`

Type 1 去掉了 Host OS 这一层，VM 内的 Guest OS 通过 Hypervisor 直接与硬件交互，没有中间的上下文切换损耗，因此性能更优。这也是数据中心和生产环境几乎全部使用 Type 1 的原因。

### 1.2 Hypervisor 的主流实现

| 实现               | 类型   | 特点                                                         |
| ------------------ | ------ | ------------------------------------------------------------ |
| ESXi hypervisor    | Type 1 | VMware 出品，适用于数据中心的服务器虚拟化，管理 VMware 虚拟机 |
| vSphere            | Type 1 | VMware 出品，ESXi 的完整版，需要付费，附带 vSphere 环境的管理服务器 |
| VMware Fusion      | Type 2 | VMware 出品，适用于 Mac 操作系统                             |
| Workstation        | Type 2 | VMware 出品，适用于 Linux、Windows 操作系统                  |
| VirtualBox         | Type 2 | 免费，适用于 Mac、Linux、Windows 操作系统，目前属于 Oracle   |
| Hyper-V            | Type 1 | 微软出品，安装后直接运行在硬件之上，原 Windows 操作系统对硬件资源优先级较高 |
| Citrix XenServer   | Type 1 | 适用于 Linux、Windows 操作系统                               |
| KVM                | Type 1 | 开源 Hypervisor，嵌入 Linux 内核，主流的 Linux 虚拟化技术都基于此。KVM 补全了 QEMU（纯软件实现的虚拟化），当处理器支持时使用 KVM，否则 fallback 到 QEMU |

![KVM hypervisor的高级别视图](/infra/virtualization/figure4-20200205001204791.gif)

| 实现               | 类型   | 特点                                                         |
| ------------------ | ------ | ------------------------------------------------------------ |
| Xen                | Type 1 | 支持 Intel 和 ARM 架构，支持 Paravirtualization 半虚拟化，支持硬件辅助的全虚拟化 |
| Red Hat Hypervisor | Type 1 | 基于 KVM，在此基础上添加了一系列基础设施                     |

### 1.3 核心虚拟化技术原理

**全虚拟化 (Full Virtualization)**：Guest OS 无需任何修改即可运行。Hypervisor 通过**二进制翻译 (Binary Translation)** 截获 Guest OS 的特权指令，将其翻译为安全指令后再在物理硬件上执行。VMware ESXi 早期就采用这种方式，兼容性最好但性能开销较大。

**半虚拟化 (Paravirtualization)**：Guest OS 需要修改内核，通过 **hypercall** 接口直接与 Hypervisor 通信，不再需要二进制翻译。代表性的有 Xen PV 模式。性能优于全虚拟化，但需要对 Guest OS 进行修改。

**硬件辅助虚拟化**：Intel VT-x 和 AMD-V 在 CPU 中增加了新的特权级（VMX root/non-root mode），使得 Guest OS 执行特权指令时会触发 **VM-Exit**，CPU 切换到 root mode 由 Hypervisor 处理，处理完成后通过 VMLAUNCH/VMRESUME 恢复 Guest OS 执行。KVM 和 Xen HVM 都基于此技术。

**内存虚拟化**：Hypervisor 维护 Guest 物理地址 (GPA) 到 Host 物理地址 (HPA) 的映射，即**影子页表 (Shadow Page Table)**。现代 CPU 支持的 **EPT (Intel) / NPT (AMD)** 将二次地址翻译交给硬件完成，大幅降低了内存虚拟化的开销。

**I/O 虚拟化**：从纯软件模拟 (QEMU) → 半虚拟化驱动 (virtio，通过共享内存 vring 与 Hypervisor 通信) → 设备直通 (SR-IOV / PCI Passthrough，物理设备直接分配给 VM)，性能逐步接近裸机。

### 1.4 参考文献

https://www.ibm.com/cloud/learn/hypervisors

---

## 二、容器化：Docker 深度解析

### 2.1 容器 vs 虚拟机

容器化是指将应用及其所有依赖打包在一起，使其能够运行在任何平台上。容器间共享操作系统内核，而虚拟化则是多个操作系统共享硬件，因此容器是比虚拟化更高一个层次的虚拟化。

**架构对比（文字描述）：**

```
虚拟机:   硬件 → Host OS → Hypervisor → [Guest OS → Bins/Libs → App] × N
容器:     硬件 → Host OS → Docker Engine → [Bins/Libs → App] × N
```

| 对比维度     | 虚拟机 (VM)                    | 容器 (Container)          |
| ------------ | ------------------------------ | ------------------------- |
| 隔离层级     | OS 级别，每个 VM 有独立内核    | 进程级别，共享 Host 内核  |
| 启动速度     | 分钟级                         | 秒级 / 毫秒级             |
| 镜像大小     | GB 级 (含完整 OS)              | MB 级                     |
| 资源开销     | 高 (Guest OS + Hypervisor 开销)| 低 (直接复用 Host 内核)   |
| 安全隔离     | 强 (独立内核，硬件级隔离)      | 弱 (共享内核，namespace 隔离) |
| 部署密度     | 一台物理机几十个 VM            | 一台物理机数百个容器      |

容器带来的好处：可移植性、敏捷、启动快、错误隔离、资源利用率高、便于编排管理。

### 2.2 Docker 架构

Docker 采用经典的 C/S 架构：

```
Docker CLI (docker) ──REST API──→ dockerd (Daemon) ──gRPC──→ containerd ──shim──→ runc → 容器进程
                                        │                       (OCI Runtime)
                                    containerd(内置)
```

- **Docker Client** (`docker`)：用户交互入口，通过 Unix Socket (`/var/run/docker.sock`) 发送命令到 Daemon
- **Docker Daemon** (`dockerd`)：核心守护进程，负责接收请求、管理镜像/容器/网络/存储
- **containerd**：管理容器生命周期的核心运行时，从 dockerd 中独立出来，通过 gRPC 通信
- **containerd-shim**：作为容器进程的父进程，解决 "runc 退出但容器不退出" 的问题，同时收集退出状态
- **runc**：OCI Runtime 参考实现，基于 Linux namespace + cgroup 创建容器进程，创建完成后即退出

### 2.3 镜像 (Image)

#### UnionFS 与分层文件系统

Docker 镜像采用 **Union File System (联合文件系统)** 实现分层存储。每一层都是只读的，只有最顶层的容器层是可写的：

```
┌──────────────────────┐
│   容器层 (Thin R/W)   │ ← 容器运行时创建，仅修改写入此层
├──────────────────────┤
│   镜像层 3 (只读)     │ ← COPY app.jar /app/
├──────────────────────┤
│   镜像层 2 (只读)     │ ← RUN apt-get install openjdk-17
├──────────────────────┤
│   镜像层 1 (只读)     │ ← FROM ubuntu:22.04
└──────────────────────┘
```

#### Copy-on-Write (COW) 写时复制

容器需要修改文件时：从镜像只读层查找文件 → 复制到容器可写层 → 在可写层修改。此后的读操作从可写层返回（上层文件会**掩盖**下层同名文件）。删除操作通过 **whiteout 文件**实现：在容器层创建 `.wh.<filename>` 隐藏文件，UnionFS 遇到它时会将该文件标记为已删除。

#### Overlay2 存储驱动

Overlay2 是 Docker 默认存储驱动（替代了早期的 AUFS/DeviceMapper），通过合并多个目录层实现：

```
lowerdir=/var/lib/docker/overlay2/l/<layer-id-1>:<layer-id-2>:...  ← 所有只读层
upperdir=/var/lib/docker/overlay2/<container-id>/diff              ← 容器可写层
workdir=/var/lib/docker/overlay2/<container-id>/work               ← 内部工作目录
merged =/var/lib/docker/overlay2/<container-id>/merged             ← 联合挂载点（容器看到的根文件系统）
```

#### 多阶段构建 (Multi-Stage Build)

解决 "构建时依赖臃肿" 的问题，将编译环境和运行环境分离：

```dockerfile
FROM maven:3.9 AS builder
WORKDIR /app
COPY pom.xml . && COPY src/ src/
RUN mvn package -DskipTests

FROM openjdk:17-slim
COPY --from=builder /app/target/app.jar /app/app.jar
CMD ["java", "-jar", "/app/app.jar"]
```

最终镜像只包含运行阶段的内容，不含 Maven、源码等编译依赖，体积大幅缩小。

### 2.4 容器隔离机制

#### Linux Namespace (命名空间隔离)

Namespace 是容器隔离的核心，每种 namespace 隔离一类系统资源：

| Namespace | 隔离内容                                       | 系统调用常量 |
| --------- | ---------------------------------------------- | ------------ |
| PID       | 进程 PID 编号，容器内 PID=1 是容器内首个进程   | CLONE_NEWPID |
| NET       | 网络设备、IP 地址、路由表、防火墙规则          | CLONE_NEWNET |
| MNT       | 文件系统挂载点，容器拥有独立的 / 根目录        | CLONE_NEWNS  |
| UTS       | 主机名 (hostname) 和 NIS 域名                  | CLONE_NEWUTS |
| IPC       | System V IPC 和 POSIX 消息队列                 | CLONE_NEWIPC |
| USER      | 用户和组 ID，容器内 uid=0 可映射到宿主机 uid=1000 | CLONE_NEWUSER |
| Cgroup    | cgroup 根目录视图，限制容器可见的 cgroup 层级  | CLONE_NEWCGROUP |

#### Cgroup (控制组资源限制)

cgroup v2 是当前推荐版本，Docker 为每个容器创建一个子 cgroup：

```
/sys/fs/cgroup/docker/<container-id>/
├── cpu.max       ← CPU 配额 ("200000 100000" = 2核)
├── memory.max    ← 内存上限 ("536870912" = 512MB)
├── memory.high   ← 内存软限制
├── io.max        ← 磁盘 IO 限制
├── pids.max      ← 进程数上限
└── cgroup.procs  ← 属于该 cgroup 的进程 PID 列表
```

| 子系统 | Docker 参数                         | 控制内容        |
| ------ | ----------------------------------- | --------------- |
| cpu    | `--cpus=2`, `--cpu-shares`          | CPU 配额与权重  |
| memory | `--memory=512m`, `--memory-swap`    | 内存硬/软限制   |
| io     | `--blkio-weight`, `--device-read-bps` | 磁盘 IO 限制  |
| pids   | `--pids-limit=100`                  | 最大进程数      |

#### 其他安全机制

- **Capabilities**：将 root 权限拆分为细粒度能力（如 `NET_BIND_SERVICE`），容器默认仅保留安全子集，推荐 `--cap-drop=ALL --cap-add=NET_BIND_SERVICE` 最小权限运行
- **Seccomp**：限制容器进程可调用的系统调用，默认 profile 禁用了约 44 个危险系统调用
- **AppArmor / SELinux**：基于路径或标签的强制访问控制，限制容器对文件系统的访问

### 2.5 网络模式

| 模式      | 参数                             | 行为                                           | 适用场景       |
| --------- | -------------------------------- | ---------------------------------------------- | -------------- |
| bridge    | `--network bridge` (默认)        | 容器连接 docker0 网桥，通过 NAT 访问外部      | 单机容器通信   |
| host      | `--network host`                 | 直接使用宿主机网络栈，无隔离                   | 高性能网络     |
| container | `--network container:NAME`       | 共享指定容器的网络 namespace（含 localhost）   | Sidecar 模式   |
| none      | `--network none`                 | 无网络，仅 lo 回环接口                         | 安全敏感场景   |
| overlay   | (Swarm 模式)                     | 跨主机通信，VXLAN 隧道封装                     | 多主机容器网络 |

**bridge 模式 iptables 规则原理：**
- 外网访问容器 (DNAT)：`PREROUTING → DOCKER chain → DNAT <宿主机IP:8080> → <容器IP:80>`
- 容器访问外网 (SNAT/MASQUERADE)：`容器 → docker0 → POSTROUTING → 源IP改为宿主机IP → eth0 → 外网`

### 2.6 存储

| 类型        | 生命周期   | 存放位置                         | 特点                              |
| ----------- | ---------- | -------------------------------- | --------------------------------- |
| Volume      | 独立于容器 | `/var/lib/docker/volumes/<name>/` | Docker 管理，推荐方式             |
| Bind Mount  | 独立于容器 | 宿主机任意路径                   | 直接挂载宿主机目录，权限管理复杂  |
| tmpfs       | 随容器     | 内存中                           | 临时存储，不写磁盘                |

```yaml
# docker-compose.yml 存储配置示例
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data             # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # bind mount
    tmpfs:
      - /tmp
volumes:
  pgdata:
    driver: local
```

### 2.7 Dockerfile 最佳实践

1. **选择官方最小基础镜像**：优先 `alpine` 或 `distroless`，避免臃肿的 `ubuntu:latest`
2. **固定镜像版本**：`FROM node:20.11-alpine` 而非 `FROM node:latest`，确保构建可重复
3. **合并 RUN 指令减少层数**：多个命令用 `&&` 连接，最后 `rm -rf /var/lib/apt/lists/*` 清理缓存
4. **先 COPY 依赖文件再 COPY 源码**：利用层缓存，依赖不变时跳过重新安装
5. **使用 .dockerignore**：排除 `node_modules`、`.git`、`*.log` 等，减小构建上下文
6. **多阶段构建**：编译和运行分离，最终镜像只含运行时必需文件
7. **使用非 root 用户**：`USER 1000`，遵循最小权限原则
8. **设置 WORKDIR**：使用绝对路径 `WORKDIR /app`，避免 `RUN cd /app && ...`
9. **优先 COPY 而非 ADD**：`ADD` 有自动解压 tar 的功能，容易引入意外行为
10. **合理排序分层**：不易变动的指令放前面（系统依赖安装），频繁变动的放后面（`COPY . .`）
11. **使用 HEALTHCHECK**：`HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost:8080/health || exit 1`
12. **使用 exec 形式的 CMD/ENTRYPOINT**：`CMD ["java", "-jar", "app.jar"]` 而非 shell 形式 `CMD java -jar app.jar`，确保信号转发正确

### 2.8 Docker Compose

Docker Compose 用于定义和运行多容器应用，通过单个 YAML 文件声明服务、网络、存储：

```yaml
version: "3.9"
services:
  app:
    build: .
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
    depends_on:
      db:
        condition: service_healthy    # 等待 db 健康检查通过后启动
    restart: unless-stopped
    deploy:
      resources:
        limits: { cpus: "2", memory: 512M }
        reservations: { cpus: "0.5", memory: 256M }

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 10s
      timeout: 5s
      retries: 5
    secrets:
      - db_password

volumes:
  pgdata:
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## 三、容器编排：Kubernetes 深度解析

### 3.1 架构全景

```
┌─────────────────── Control Plane ───────────────────┐
│  ┌───────────┐   ┌────────┐   ┌────────────┐        │
│  │API Server │←→ │  etcd  │   │ Controller │        │
│  │(REST API) │   │(Raft KV)│   │  Manager   │        │
│  └─────┬─────┘   └────────┘   │(调谐循环)   │        │
│        │                      └────────────┘        │
│  ┌─────┴─────┐                                      │
│  │ Scheduler │ 为 Pod 选择最优 Node                  │
│  └───────────┘                                      │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS (6443)
  ┌────────────────────┼────────────────────┐
  │  Worker Node 1     │                    │  Worker Node 2
  │  ┌───────┐ ┌──────┐│                    │  ┌───────┐ ┌──────┐
  │  │kubelet│ │kube- ││                    │  │kubelet│ │kube- │
  │  │CRI→runc│ │proxy ││                    │  └───────┘ │proxy │
  │  └───────┘ └──────┘│                    │  ┌────────────────┐
  │  ┌────────────────┐│                    │  │ Pod: [pause]   │
  │  │ Pod: [pause]   ││                    │  │ + [app]+[sidecar]│
  │  │ + [app container]│                   │  └────────────────┘
  │  └────────────────┘│
```

| 组件               | 位置    | 职责                                                         |
| ------------------ | ------- | ------------------------------------------------------------ |
| **kube-apiserver** | Master  | 集群统一入口 (REST API)，认证/授权/准入控制，所有组件通过它交互 |
| **etcd**           | Master  | 分布式 KV 存储 (Raft 共识)，保存集群所有状态数据              |
| **kube-scheduler** | Master  | 监听未绑定 Node 的 Pod，根据调度策略选择最优节点              |
| **controller-manager** | Master | 运行各类 Controller（Deployment/ReplicaSet/Node 等），通过调谐循环将当前状态驱动到期望状态 |
| **kubelet**        | Node    | 节点代理，接收 PodSpec，通过 CRI 调用容器运行时管理容器       |
| **kube-proxy**     | Node    | 维护节点上的 iptables/IPVS 规则，实现 Service 到 Pod 的流量转发 |

### 3.2 Pod

Pod 是 K8s 最小调度单元，包含一个或多个共享 namespace 的容器。

**Pause 容器**：每个 Pod 启动时首先创建 pause 容器（~700KB 的 sandbox 容器），作用：
1. 创建并持有 Pod 的 NET / IPC namespace
2. 后续业务容器加入 pause 容器的 namespace，实现网络和 IPC 共享
3. 极为精简，几乎不会崩溃，保证 Pod 网络 namespace 不丢失

Pod 内容器通过 localhost 互相访问，共享同一 IP 和端口空间。PID namespace 可选共享，MNT namespace 各自独立。

**探针 (Probes)：**

| 探针              | 作用                              | 失败后果                          |
| ----------------- | --------------------------------- | --------------------------------- |
| **livenessProbe** | 检查容器是否**存活**（死锁/卡死） | 杀死容器，按 restartPolicy 重启   |
| **readinessProbe**| 检查容器是否**就绪**接受流量      | 从 Service Endpoints 中移除       |
| **startupProbe**  | 检查容器是否**启动完成**（慢启动）| 杀死容器，按 restartPolicy 重启   |

```yaml
spec:
  containers:
    - name: app
      startupProbe:                          # 慢启动保护
        httpGet: { path: /healthz, port: 8080 }
        failureThreshold: 30                 # 最多等 30×10s = 300s
        periodSeconds: 10
      livenessProbe:                         # 存活检查
        httpGet: { path: /healthz, port: 8080 }
        periodSeconds: 15
        failureThreshold: 3
      readinessProbe:                        # 就绪检查
        httpGet: { path: /ready, port: 8080 }
        periodSeconds: 5
```

探针执行方式：**exec**（容器内执行命令）、**httpGet**、**tcpSocket** 三种。

### 3.3 Controller

Controller 是 K8s 核心设计哲学：**声明式 API + 调谐循环 (Reconcile Loop)**。

```
调谐循环伪代码:
for {
    currentState = getCurrentState()
    desiredState = getDesiredState()
    if currentState != desiredState {
        executeActions(computeDiff(currentState, desiredState))
    }
}
```

| Controller     | 用途                                     | 核心机制                                    |
| -------------- | ---------------------------------------- | ------------------------------------------- |
| **Deployment** | 无状态应用，滚动更新/回滚                | 管理 ReplicaSet，RS 管理 Pod                 |
| **ReplicaSet** | 保证指定数量的 Pod 副本运行              | 通过 label selector 匹配 Pod                 |
| **StatefulSet**| 有状态应用，稳定标识/有序部署/持久存储   | Pod 固定名称 (app-0, app-1...)，PVC 模板     |
| **DaemonSet**  | 每个节点运行一个 Pod 副本                | 日志收集 (Fluentd)、监控 (Node Exporter)     |
| **Job**        | 一次性任务                               | completions 成功次数，parallelism 并行度     |
| **CronJob**    | 定时任务，cron 表达式                    | 在指定时间创建 Job                           |

#### Deployment 滚动更新原理

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 更新时最多多创建 1 个 Pod
      maxUnavailable: 0    # 更新时不可用 Pod 最多 0 个
```

**更新过程 (replicas=3, maxSurge=1, maxUnavailable=0)：**
```
Step 1: [v1][v1][v1]         → 创建 1 个 v2 → [v1][v1][v1][v2]
Step 2: [v1][v1][v1][v2]     → v2 就绪，删 1 个 v1 → [v1][v1][v2]
Step 3: [v1][v1][v2]         → 创建 1 个 v2 → [v1][v1][v2][v2]
Step 4: 删 1 个 v1 → [v1][v2][v2] → 创建 v2 → 删 v1 → [v2][v2][v2]  完成！
```

Deployment 通过逐步伸缩新旧 ReplicaSet 实现滚动更新，旧 RS 保留以便回滚 (`kubectl rollout undo`)。

#### StatefulSet 特性

- **稳定网络标识**：`<statefulset>-<序号>.<headless-svc>.<ns>.svc.cluster.local`
- **稳定存储**：每个 Pod 绑定独立 PVC（通过 `volumeClaimTemplates`），Pod 重建后仍挂载相同存储
- **有序部署与逆序删除**：`podManagementPolicy: OrderedReady` → 0→1→2 启动，2→1→0 删除

### 3.4 Service

Service 为 Pod 提供稳定入口，解决 Pod IP 不固定的问题。

| 类型           | 行为                                                | 适用场景                |
| -------------- | --------------------------------------------------- | ----------------------- |
| **ClusterIP**  | 默认类型，集群内部 VIP，仅集群内可访问              | 内部服务间调用          |
| **NodePort**   | 在每个 Node 上开放固定端口 (30000-32767)            | 开发测试 / 简单外部访问 |
| **LoadBalancer**| 通过云厂商 LB 暴露公网或内网地址                    | 生产环境外部入口        |
| **ExternalName**| 返回 CNAME，将 Service 映射到外部 DNS 名称         | 代理集群外部服务        |
| **Headless**   | `clusterIP: None`，DNS 直接返回 Pod IP             | StatefulSet 对等发现    |

#### kube-proxy 模式对比

| 模式      | 原理                                   | 特点                                 |
| --------- | -------------------------------------- | ------------------------------------ |
| iptables  | 随机负载均衡，规则数随 Service/Pod 增长 | 简单，中小规模集群                   |
| IPVS      | 内核 L4 负载均衡器，支持 rr/lc/dh/sh   | 性能优，大规模集群 (>1000 Service)   |

```yaml
apiVersion: v1
kind: Service
spec:
  selector: { app: myapp }
  ports:
    - protocol: TCP
      port: 80          # Service 端口
      targetPort: 8080  # Pod 端口
  type: ClusterIP
  sessionAffinity: ClientIP   # 同一客户端 IP 路由到同一 Pod
```

### 3.5 Ingress / Gateway API

**Ingress** 负责将集群外部的 HTTP(S) 流量路由到集群内部的 Service：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: [example.com]
      secretName: tls-secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api-service, port: { number: 8080 } }
```

Ingress 的局限性推动了 **Gateway API** 的出现，它将角色拆分为 GatewayClass（基础设施层）、Gateway（网关实例）和 HTTPRoute（路由规则），支持灰度发布（权重路由）、请求匹配、跨 namespace 路由等高级功能。

### 3.6 存储

```
PV (PersistentVolume) ←──绑定──→ PVC (PersistentVolumeClaim)
  集群存储资源                     用户请求 (大小/访问模式)
       ↑                                    
StorageClass (动态供应) → provisioner + 参数 (AWS EBS / NFS / Ceph RBD)
```

| 访问模式            | 含义                            |
| ------------------- | ------------------------------- |
| RWO (ReadWriteOnce) | 单节点读写挂载                  |
| ROX (ReadOnlyMany)  | 多节点只读挂载                  |
| RWX (ReadWriteMany) | 多节点读写挂载 (需 NFS/CephFS)  |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer   # 延迟绑定，避免跨区问题
allowVolumeExpansion: true                # 允许在线扩容
```

**CSI (Container Storage Interface)** 是 K8s 与存储系统的标准接口，包含三个 gRPC 服务：
- **Identity Service**：返回插件信息和能力
- **Controller Service**：创建/删除/快照/扩容卷（由外部 provisioner 调用）
- **Node Service**：挂载/卸载卷到 Pod（由 kubelet 调用）

### 3.7 配置管理

#### ConfigMap

存储非敏感配置，支持环境变量注入和文件挂载：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    server:
      port: 8080
    database:
      pool-size: 20
---
# 环境变量注入
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef: { name: app-config }
---
# 文件挂载 (支持热更新，约60s生效)
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap: { name: app-config }
```

#### Secret

存储敏感信息（密码/Token/证书），数据 base64 编码（非加密），需配合 etcd 加密和 RBAC 使用：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: dXNlcm5hbWU=          # echo -n "username" | base64
  password: cGFzc3dvcmQ=
---
spec:
  containers:
    - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-secret, key: password }
```

**安全实践**：使用 External Secrets Operator 对接 Vault；启用 etcd encryption at rest；配合 RBAC 限制 Secret 访问。

### 3.8 调度

#### 亲和性 (Affinity) 与反亲和性 (Anti-Affinity)

```yaml
spec:
  affinity:
    podAntiAffinity:                       # 高可用：Pod 分散到不同节点
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: { app: myapp }
          topologyKey: kubernetes.io/hostname
    nodeAffinity:                          # 选择 GPU 节点
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - { key: node-type, operator: In, values: [gpu, compute] }
```

`requiredDuringScheduling...` 是硬要求；`preferredDuringScheduling...` 是软偏好。

#### 污点 (Taint) 与容忍 (Toleration)

污点在 Node 上，容忍在 Pod 上。没有对应容忍的 Pod 无法被调度到该 Node。

```bash
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule     # 专用节点
```

```yaml
spec:
  tolerations:
    - { key: gpu, operator: Equal, value: "true", effect: NoSchedule }
```

污点效果：**NoSchedule**（不调度）、**PreferNoSchedule**（尽量不调度）、**NoExecute**（不调度+驱逐已有无容忍Pod）。

#### 资源请求与限制

```yaml
spec:
  containers:
    - resources:
        requests: { cpu: "500m", memory: "256Mi" }   # 调度据此选节点
        limits:   { cpu: "2",    memory: "512Mi" }   # 运行时上限（超内存→OOMKilled）
```

**QoS 类别**：
- **Guaranteed**：CPU/Memory 的 `requests == limits`，最高优先级，最后被驱逐
- **Burstable**：`requests < limits`，中优先级
- **BestEffort**：未设置 resources，最低优先级，最先被驱逐

#### 拓扑分布约束 (Topology Spread Constraints)

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                              # 两 zone 间 Pod 数差 ≤ 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app: myapp }
```

### 3.9 自动伸缩

#### HPA (Horizontal Pod Autoscaler)

基于 CPU/Memory 或自定义指标自动调整副本数：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # 缩容冷静期 5 分钟
```

**HPA 公式**：`期望副本数 = ceil(当前副本数 × 当前指标值 / 目标指标值)`
例如：3 副本，CPU 90%，目标 70% → `ceil(3 × 90/70) = 4`

#### VPA (Vertical Pod Autoscaler)

自动调整 Pod 的 CPU/Memory requests 和 limits，调整时会重启 Pod（需配合 PDB）。

#### Cluster Autoscaler

当 Pod 因资源不足 pending 时自动申请新节点加入集群；节点利用率偏低时自动释放节点。

### 3.10 网络模型 (CNI)

K8s 网络基本要求：每个 Pod 独立 IP、同/跨节点 Pod 直接通信（无需 NAT）、Pod 与 Service 可通信。

| 对比维度     | Calico                                    | Flannel                         |
| ------------ | ----------------------------------------- | ------------------------------- |
| 数据平面     | BGP 三层路由 或 IPIP/VXLAN 封装           | VXLAN / Host-GW / UDP           |
| 性能         | BGP 模式：无封装开销，接近原生网络        | VXLAN：轻微封装开销             |
| 网络策略     | 原生支持 NetworkPolicy                    | 不支持（需配合 Calico/Cilium）  |
| 复杂度       | 中等                                      | 低                              |
| 适用场景     | 高性能、网络策略、大型集群                | 简单场景、快速部署              |

**Flannel VXLAN 路径**：Pod A → cni0 网桥 → flannel.1 (VTEP 封装 VXLAN) → 物理网卡 → Node2 flannel.1 (解封装) → cni0 → Pod B

**Calico BGP 路径**：Pod A → caliXXXX (veth) → 查路由表: 目标子网 via Node2 IP → 物理网卡 → 物理网络纯 IP 路由 → Node2 → Pod B

### 3.11 核心控制器工作原理

#### Informer 模式

Controller 基于 etcd Watch 实现事件驱动 + 水平触发：

```
API Server → Watch → Reflector → DeltaFIFO → Indexer (本地缓存)
                                                   ↓
                                             Lister (读本地缓存)
                                                   ↓
                                             WorkQueue (去重+排队)
                                                   ↓
                                             Worker 协程 (调谐)
```

Informer 监听 API Server 变更，缓存到本地 Indexer，Lister 从本地缓存读取（避免频繁查 API Server），WorkQueue 去重后交给 Worker 执行调谐逻辑。

#### Deployment Controller 调谐循环

1. Watch Deployment 变更 → 获取对应 ReplicaSet 列表
2. 比较期望副本数 vs 当前副本数
3. 如果期望 Template 与当前 RS 不同 → 创建新 RS (滚动更新)
4. 按 maxSurge/maxUnavailable 逐步伸缩新旧 RS
5. 旧 RS 副本数缩为 0 后保留 (用于回滚)

#### ReplicaSet Controller 调谐循环

1. Watch RS / Pod 变更 → Selector 匹配所属 Pod
2. `diff = desiredReplicas - currentReplicas`
3. `diff > 0` → 根据 Template 创建 Pod
4. `diff < 0` → 按创建时间和状态排序删除 Pod

---

## 四、云原生生态

### 4.1 Service Mesh — Istio

Service Mesh 将流量管理、安全、可观测性从应用代码剥离到 Sidecar 代理层：

```
Pod: [App Container] ←──localhost──→ [Envoy Sidecar]
                                          │ mTLS
                                   ┌──────┴──────┐
                                   │  istiod     │ (控制面)
                                   │  Pilot: 流量管理     │
                                   │  Citadel: 证书签发   │
                                   └─────────────┘
```

**核心能力：**

**流量管理 — 灰度发布：**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
          weight: 90           # 90% 流量到 v1
        - destination: { host: reviews, subset: v2 }
          weight: 10           # 10% 到 v2 (金丝雀发布)
```

**安全**：Envoy 自动为服务间通信提供 mTLS 加密，无需修改应用代码。

**可观测性**：Envoy 自动生成分布式追踪 (Jaeger/Zipkin)、Prometheus 指标、访问日志。

**故障注入**：支持延迟注入和错误注入，用于韧性测试。

### 4.2 Serverless — Knative

Knative 基于 K8s 构建 Serverless 平台，核心组件：
- **Serving**：请求驱动的容器部署和自动伸缩（支持缩容到 0）
- **Eventing**：事件驱动的松耦合架构（CloudEvents 标准）

**冷启动优化**：缩容到 0 节省成本 → `minScale: 1` 保持预热 → 小镜像加速拉取 → 合理 startupProbe 快速标记就绪。

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
spec:
  template:
    spec:
      containerConcurrency: 100     # 每容器最多 100 并发
      containers:
        - image: myapp:latest
```

### 4.3 Helm

Kubernetes 的包管理器，通过 Chart（K8s 资源模板）实现应用一键部署和生命周期管理：

```yaml
# values.yaml (用户配置)
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
service:
  type: ClusterIP
---
# templates/deployment.yaml (模板)
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

关键功能：模板渲染 (Go Template)、Release 管理 (`helm upgrade/rollback`)、Hooks (安装前后执行 Job)。

### 4.4 GitOps — ArgoCD / Flux

GitOps 核心理念：**Git 仓库是声明式基础设施的唯一事实来源**。

```
开发者 Push K8s Manifests → Git 仓库 → ArgoCD 检测变更 (轮询/Webhook)
  → 对比 Git 期望 vs 集群实际 → 自动/手动 Sync → 集群状态与 Git 一致
```

ArgoCD 核心概念：
- **Application**：一组 K8s 资源集合，由 Git 仓库定义
- **Sync**：将集群状态驱动到 Git 期望状态的过程

---

## 五、生产实践

### 5.1 资源配额

**ResourceQuota** 限制 namespace 总资源：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    services.loadbalancers: "2"
```

**LimitRange** 限制单个容器资源范围并设置默认值：

```yaml
apiVersion: v1
kind: LimitRange
spec:
  limits:
    - type: Container
      default: { cpu: "500m", memory: "256Mi" }       # 未设 limits 的默认值
      defaultRequest: { cpu: "200m", memory: "128Mi" } # 未设 requests 的默认值
      max: { cpu: "4", memory: "8Gi" }                 # 单容器上限
      min: { cpu: "50m", memory: "64Mi" }
```

### 5.2 日志收集

#### EFK Stack (Elasticsearch + Fluentd + Kibana)

```
Pod stdout/stderr → /var/log/containers/*.log → Fluentd (DaemonSet)
  → parse + enrich → Elasticsearch → Kibana (Dashboard查询)
```

**日志最佳实践**：输出到 stdout/stderr（不写文件）、使用 JSON 结构化日志、添加 `timestamp/level/service/traceId/message` 标准字段。

#### Loki (Grafana Loki)

相比 EFK 更轻量，只索引标签（不索引日志内容），日志数据存对象存储 (S3/GCS)。

### 5.3 监控 (Prometheus + Grafana)

```
Service Discovery → Prometheus Server (抓取+存储TSDB) → Grafana Dashboard
                                            ↓
                                      Alertmanager (告警路由)
```

**四大黄金信号 (The Four Golden Signals)：**

| 信号        | PromQL 示例                                                   |
| ----------- | ------------------------------------------------------------- |
| **延迟**    | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| **流量**    | `rate(http_requests_total[5m])`                               |
| **错误**    | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| **饱和度**  | `100 - rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100`  |

**USE 方法（资源层面）**：Utilization（使用率）、Saturation（饱和度）、Errors（错误数）。

### 5.4 常见问题排查

#### Pod 一直 Pending
```bash
kubectl describe pod <pod-name>
```
常见原因：资源不足（扩容节点或调小 requests）、节点选择器不匹配（检查 `nodeSelector`/`affinity`）、污点不容忍（检查 taint/toleration）、PVC 无法绑定。

#### Pod 一直 CrashLoopBackOff
```bash
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```
常见原因：启动命令错误（应用立即退出）、依赖不可用（加 `initContainers`/`startupProbe`）、OOMKilled（内存超限）、健康检查过严。

#### Service 无法访问
排查步骤：
1. `kubectl get endpoints <svc-name>` → 应有 Pod IP:Port
2. 检查 Service selector 是否匹配 Pod labels
3. 检查 `targetPort` 与容器监听端口一致
4. 检查 NetworkPolicy 是否阻止流量
5. 检查 kube-proxy 是否正常

#### 节点 NotReady
```bash
kubectl describe node <node-name>
```
常见原因：kubelet 异常、`MemoryPressure`/`DiskPressure` 为 True、CNI 插件异常、与 API Server 网络不通。

#### DNS 解析异常
检查 CoreDNS Pod (`kube-system`) 状态、Pod 的 `dnsPolicy` 和 `/etc/resolv.conf`、网络策略（UDP 53）。

#### 快速诊断脚本

```bash
kubectl get deploy,pod,svc,ep -n <namespace>
kubectl describe pod -l app=<app-name> -n <namespace> | grep -A5 Events
kubectl top pod -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

### 5.5 生产环境检查清单

| 检查项               | 说明                                              |
| -------------------- | ------------------------------------------------- |
| ✅ 资源 requests/limits | 所有容器必须设置，避免 OOM / CPU 饥饿            |
| ✅ 存活/就绪探针       | 合理配置 livenessProbe 和 readinessProbe          |
| ✅ PDB                | PodDisruptionBudget 保证自愿中断时最小可用数      |
| ✅ HPA                | 关键服务配置自动伸缩                              |
| ✅ 反亲和性            | 配置 podAntiAffinity 实现跨节点/跨区高可用         |
| ✅ NetworkPolicy      | 默认拒绝，按需放行（零信任网络）                  |
| ✅ RBAC               | 最小权限原则，ServiceAccount 按需绑定             |
| ✅ Pod安全标准         | 使用 restricted 策略禁止特权容器                  |
| ✅ 优雅关闭            | 应用处理 SIGTERM + `terminationGracePeriodSeconds` |
| ✅ 监控+告警           | 四大黄金信号覆盖，告警渠道畅通                    |
| ✅ 结构化日志           | JSON 格式输出到 stdout/stderr                     |
| ✅ 备份               | etcd 定期备份，PV 数据快照策略                    |

---

## 六、参考文献

### Hypervisor 虚拟化
https://www.ibm.com/cloud/learn/hypervisors

### 容器化
https://www.ibm.com/cloud/learn/containers
https://www.ibm.com/cloud/learn/containerization

### Docker
https://docs.docker.com/get-started/overview/
https://docs.docker.com/storage/storagedriver/overlayfs-driver/
https://docs.docker.com/build/building/best-practices/

### Kubernetes
https://kubernetes.io/docs/concepts/overview/components/
https://kubernetes.io/docs/concepts/workloads/pods/
https://kubernetes.io/docs/concepts/services-networking/service/
https://kubernetes.io/docs/concepts/scheduling-eviction/

### 云原生生态
https://istio.io/latest/docs/concepts/what-is-istio/
https://knative.dev/docs/
https://argo-cd.readthedocs.io/

### 监控
https://prometheus.io/docs/introduction/overview/
