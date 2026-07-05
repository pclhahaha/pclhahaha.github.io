---
title: Docker 深度解析
date: 2026-07-05
updated: 2026-07-05
tags:
  - Docker
  - 容器
  - 镜像
  - Dockerfile
categories:
  - 容器化
---

### 1.1 容器 vs 虚拟机

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

### 1.2 Docker 架构

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

### 1.3 镜像 (Image)

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

### 1.4 容器隔离机制

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

### 1.5 网络模式

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

### 1.6 存储

| 类型        | 生命周期   | 存放位置                         | 特点                              |
| ----------- | ---------- | -------------------------------- | --------------------------------- |
| Volume      | 独立于容器 | `/var/lib/docker/volumes/<name>/` | Docker 管理，推荐方式             |
| Bind Mount  | 独立于容器 | 宿主机任意路径                   | 直接挂载宿主机目录，权限管理复杂  |
| tmpfs       | 随容器     | 内存中                           | 临时存储，不写磁盘                |

```yaml
`
