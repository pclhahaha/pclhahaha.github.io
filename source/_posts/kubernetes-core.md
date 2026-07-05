---
title: Kubernetes 核心概念
date: 2026-07-05
updated: 2026-07-05
tags:
  - Kubernetes
  - K8s
  - Pod
  - Service
  - Ingress
categories:
  - 容器化
---

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


```yaml
`
```
