---
title: "虚拟化与容器化: 从 Hypervisor 到 K8s"
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 虚拟化
  - Docker
  - Kubernetes
  - 容器
  - 云原生
categories:
  - 基础设施
---

---

> 📖 本文前六节（Docker 基础、容器 vs 虚拟机、镜像分层、网络模式、存储卷、多阶段构建）已独立为 [Docker 深度解析](/docker-deep-dive/)，建议先阅读。以下从 Dockerfile 实践开始。

### 1.7 Dockerfile 最佳实践

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

### 1.8 Docker Compose

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

## 二、容器编排——Kubernetes 深度解析

### 2.1 架构全景

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

### 2.2 Pod

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

### 2.3 Controller

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

### 2.4 Service

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

### 2.5 Ingress / Gateway API

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

### 2.6 存储

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

### 2.7 配置管理

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

### 2.8 调度

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

### 2.9 自动伸缩

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

### 2.10 网络模型 (CNI)

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

### 2.11 核心控制器工作原理

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

## 三、云原生生态云原生生态

### 3.1 Service Mesh — Istio

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

### 3.2 Serverless — Knative

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

### 3.3 Helm

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

### 3.4 GitOps — ArgoCD / Flux

GitOps 核心理念：**Git 仓库是声明式基础设施的唯一事实来源**。

```
开发者 Push K8s Manifests → Git 仓库 → ArgoCD 检测变更 (轮询/Webhook)
  → 对比 Git 期望 vs 集群实际 → 自动/手动 Sync → 集群状态与 Git 一致
```

ArgoCD 核心概念：
- **Application**：一组 K8s 资源集合，由 Git 仓库定义
- **Sync**：将集群状态驱动到 Git 期望状态的过程

---

## 四、生产实践生产实践

### 4.1 资源配额

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

### 4.2 日志收集

#### EFK Stack (Elasticsearch + Fluentd + Kibana)

```
Pod stdout/stderr → /var/log/containers/*.log → Fluentd (DaemonSet)
  → parse + enrich → Elasticsearch → Kibana (Dashboard查询)
```

**日志最佳实践**：输出到 stdout/stderr（不写文件）、使用 JSON 结构化日志、添加 `timestamp/level/service/traceId/message` 标准字段。

#### Loki (Grafana Loki)

相比 EFK 更轻量，只索引标签（不索引日志内容），日志数据存对象存储 (S3/GCS)。

### 4.3 监控 (Prometheus + Grafana)

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

### 4.4 常见问题排查

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

### 4.5 生产环境检查清单

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

## 五、参考文献参考文献

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
