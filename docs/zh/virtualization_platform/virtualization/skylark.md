# Skylark

<!-- TOC -->

- [Skylark概述](#skylark概述)
- [架构及特性](#架构及特性)
- [安装Skylark](#安装skylark)
- [配置Skylark](#配置skylark)
- [使用Skylark](#使用skylark)
- [最佳实践](#最佳实践)

<!-- /TOC -->

## Skylark概述

### 问题背景

随着云计算市场规模的快速增长，各云厂商基础设施投入也不断增加。资源利用率低是行业普遍存在的问题，在上述背景下，提升资源利用率已经成为了一个重要的技术课题。本文档介绍 openEuler Skylark 组件，并给出安装方法及使用指导。

### 总体介绍

将业务区分优先级混合部署（下文简称混部）是典型有效的资源利用率提升手段。业务可根据时延敏感性分为高优先级业务和低优先级业务。当高优先级业务和低优先级业务发生资源竞争时，需优先保障高优先级业务的资源供给。因此，业务混部的核心技术是资源隔离控制，主要涉及内核态基础资源隔离技术及用户态 QoS 控制技术。

本文描述的对象为用户态 QoS 控制技术，由 openEuler Skylark 组件承载，首发于 openEuler 22.09 版本。在 Skylark 视角下，优先级粒度为虚拟机级别，即给虚拟机新增高低优先级属性，以虚拟机为粒度进行资源的隔离和控制。Skylark 是一种混部场景下的 QoS 感知的资源调度器，在保障高优先级虚拟机 QoS 前提下提升物理机资源利用率。

在实际应用场景中如何更好地利用 Skylark 的高低优先级特性，请参考[最佳实践](#最佳实践)章节。

## 架构及特性

### 总体实现框架

Skylark 核心类为`QoSManager`，类成员包括数据收集类实例、QoS 分析类实例、QoS 控制类实例、以及任务调度类实例：

- `DataCollector`：数据收集类，有`HostInfo`和`GuestInfo`两个成员，分别用于收集主机信息和虚拟机信息。
- `PowerAnalyzer`：功耗分析类，用于分析功耗干扰以及需要限制的低优先级虚拟机。
- `CpuController`：CPU 带宽控制类，用于限制低优先级虚拟机的 CPU 带宽。
- `CacheMBWController`：LLC 及内存带宽控制类，用于限制低优先级虚拟机的 LLC 和内存带宽。
- `BackgroundScheduler`：任务调度类，用于周期性驱动以上模块，持续进行 QoS 管理。

Skylark 检查主机环境后，创建守护进程。守护进程有两种线程：主调度线程和 Job 线程：

- 主调度线程是唯一的，首先连接 Libvirt，然后创建并初始化`QosManager`类实例，最后开始驱动 Job 线程。
- Job 线程可能不止一个，每个 Job 线程负责周期性执行某个 QoS 管理任务。

### 功耗干扰控制

相比非混部情况，混部后主机利用率更高，高利用率意味着高功耗，服务器功耗在超过 TDP 时会触发 CPU 降频。Skylark 支持当功耗超过预设的 TDP 阈值（即出现 TDP 热点）时，通过对低优先级虚拟机的 CPU 带宽进行限制，以此达到降低整机功耗的同时保障高优先级虚拟机 QoS。

Skylark 初始化时，根据[配置Skylark](#配置skylark)中相关配置值，设置功耗干扰控制属性。在每个控制周期，综合分析主机信息和控制属性，判断是否出现 TDP 热点。如果出现热点，进一步根据虚拟机信息分析出需要对哪些低优先级虚拟机进行 CPU 带宽的限制。

### LLC/MB干扰控制

Skylark 支持对低优先级虚拟机的 LLC 和内存带宽进行限制，当前仅支持静态分配。Skylark 通过操作系统提供的`/sys/fs/resctrl`接口来限制低优先级虚拟机的 LLC 和内存带宽。

1. Skylark 在`/sys/fs/resctrl`目录下建立`low_prio_machine`文件夹，并将低优先级虚拟机的 pid 写入`/sys/fs/resctrl/low_prio_machine/tasks`文件中。
2. Skylark 根据[配置Skylark](#配置skylark)章节中 LLC/MB 相关配置项对低优先级虚拟机的 LLC ways 和内存带宽进行分配，配置项写入`/sys/fs/resctrl/low_prio_machine/schemata`文件中。

### CPU干扰控制

混部场景下，低优先级虚拟机会对高优先级虚拟机产生 CPU 时间片干扰和 SMT（硬件超线程）干扰。

- 当高低优先级虚拟机相关线程在同一个最小 CPU 拓扑单元（core 或 SMT）上同时处于可运行状态时，会竞争 CPU 时间片。
- 当高低优先级虚拟机相关线程在同一个 CPU core 的不同 SMT 上同时处于可运行状态时，会竞争 SMT 共享的 core 内资源。

CPU 干扰控制分为 CPU 时间片干扰控制及 SMT 干扰控制，分别基于内核提供的 `QOS_SCHED` 及 `SMT_EXPELLER` 特性实现。

- `QOS_SCHED` 特性实现了单个 CPU core 或 SMT 上高优先级虚拟机对低优先级虚拟机的绝对压制，解决了 CPU 时间片干扰问题。
- `SMT_EXPELLER` 特性实现了同一个 CPU core 的不同 SMT 上高优先级虚拟机对低优先级虚拟机的绝对压制，解决了 SMT 干扰问题。

Skylark 初始化时，会把 Cgroup CPU 子控制器下低优先级虚拟机对应 slice 层级的`cpu.qos_level`字段设置为 -1，以使能上述内核特性，后续就由内核实现对 CPU 相关干扰的控制，Skylark 无需介入。

## 安装Skylark

### 硬件要求

处理器架构：仅支持 AArch64 和 Intel x86_64 处理器架构。

- Intel 处理器需支持 RDT 功能。
- AArch64 当前仅支持 Kunpeng920，且需将 bios 升级到 1.79 及以上以支持 MPAM 功能。

### 软件要求

- 依赖 python3、python3-APScheduler、python3-libvirt 等 python 组件。
- 依赖 systemd 组件，版本 >= 249-32
- 依赖 libvirt 组件，版本 >= 1.0.5
- 依赖 openEuler 内核，版本 >= 5.10.0

### 安装方法

推荐使用 yum 安装 Skylark 组件，因为 yum 会自动处理上述软件依赖：

```shell
# yum install -y skylark
```

检查 Skylark 是否安装成功，若安装成功则会显示 skylarkd 后台服务状态：

```shell
# systemctl status skylarkd
```

设置 Skylark 服务开机自启动（可选）：

```shell
# systemctl enable skylarkd
```

## 配置Skylark

安装好 Skylark 组件后，若默认配置不满足需求，可修改配置文件。Skylark 的配置文件路径为`/etc/sysconfig/skylarkd`，下面对该配置文件包含的配置项作详细说明。

### 日志

- `LOG_LEVEL`用于设置最小日志级别，类型为字符串。所有可设置的日志级别及其关系为`critical > error > warning > info > debug`。级别小于`LOG_LEVEL`的日志将不会输出到日志文件。日志文件路径为`/var/log/skylark.log`。Skylark 会每 7 天备份一次日志，最多备份 4 次（当次数超限时，会删除最旧的日志）。备份的日志路径为`/var/log/skylark.log.%Y-%m-%d`。

### 功耗干扰控制

- `POWER_QOS_MANAGEMENT`用于控制是否打开功耗 QoS 管理功能，类型为布尔。当前仅 x86 支持该功能。如果主机上虚拟机的 CPU 利用率能被很好地限制，该功能可选。

- `TDP_THRESHOLD`用于控制虚拟机可达到的最大功耗。当主机功耗超过`TDP * TDP_THRESHOLD`时，将判断为出现 TDP 热点，触发功耗控制操作。类型为 float，可接受的输入范围为 0.8-1，默认值为 0.98。

- `FREQ_THRESHOLD`用于控制当主机出现 TDP 热点时，CPU 运行的最低频率。类型为 float，可接受的输入范围为 0.9-1，默认值为 0.98。
  1. 当存在某些 CPU 的频率低于`max_freq * FREQ_THRESHOLD`时，Skylark 会限制在这些 CPU 上运行的低优先级虚拟机的 CPU 带宽。
  2. 当找不到这样的 CPU，则 Skylark 也会根据低优先级虚拟机的 CPU 利用率情况，选择性限制某些低优先级虚拟机的 CPU 带宽。

- `QUOTA_THRESHOLD`用于控制低优先级虚拟机被限制后所能获得的 CPU 带宽（限制前的 CPU 带宽 * `QUOTA_THRESHOLD`）。类型为 float，可接受的输入范围为 0.8-1，默认值为 0.9。

- `ABNORMAL_THRESHOLD`用于控制低优先级虚拟机被限制的周期。类型为 int，可接受的输入范围为 1-5，默认值为 3。
  1. 在每个功耗控制周期内，如果某个低优先级虚拟机被限制，其剩余被限制周期刷新为`ABNORMAL_THRESHOLD`。
  2. 否则其剩余被限制周期减 1。当虚拟机的剩余被限制周期等于 0 时，其 CPU 带宽恢复为被限制前的值。

### LLC/MB干扰控制

Skylark 对 LLC/MB 的干扰控制依赖于硬件使能 RDT/MPAM 功能，Intel x86_64 架构处理器需在内核 cmdline 配置`rdt=cmt,mbmtotal,mbmlocal,l3cat,mba`，Kunpeng920 处理器需在内核 cmdline 配置`mpam=acpi`。

- `MIN_LLC_WAYS_LOW_VMS`用于控制低优先级虚拟机可访问的 LLC ways。类型为 int，可接受的输入范围为 1-3，默认值为 2。Skylark 会在初始化时，限制低优先级虚拟机的 LLC ways 为该值。

- `MIN_MBW_LOW_VMS`用于控制低优先级虚拟机可访问的内存带宽比例。类型为 float，可接受的输入范围为 0.1～0.2，默认值为 0.1。Skylark 会在初始化时，限制低优先级虚拟机的内存带宽为该值。

## 使用Skylark

### 启动服务

初次启动：

```shell
# systemctl start skylarkd
```

重新启动（修改配置文件后需重启）：

```shell
# systemctl restart skylarkd
```

### 创建虚拟机

Skylark 借助虚拟机 XML 配置文件的`partition`标签标识虚拟机优先级属性。

创建低优先级虚拟机，其 XML 需做如下配置：

```xml
<domain>
  ...
  <resource>
    <partition>/low_prio_machine</partition>
  </resource>
  ...
</domain>
```

创建高优先级虚拟机，其 XML 需做如下配置：

```xml
<domain>
  ...
  <resource>
    <partition>/high_prio_machine</partition>
  </resource>
  ...
</domain>
```

后续创建虚拟机流程和一般流程无异。

### 虚拟机运行

Skylark 能感知到虚拟机创建事件，纳管所有高、低优先级虚拟机，并围绕 CPU、功耗、LLC/MB 等资源做自动化 QoS 管理。

## 最佳实践

### 虚拟机业务推荐

- 高优先级虚拟机业务推荐：时延敏感类业务，如 web 服务、高性能数据库、实时渲染、机器学习推理等。
- 低优先级虚拟机业务推荐：非时延敏感类业务，如视频编码、大数据处理、离线渲染、机器学习训练等。

### 虚拟机绑核配置

为了让高优先级虚拟机达到最佳性能，推荐高优先级虚拟机 vCPU 与物理 CPU 一对一绑核。为了让低优先级虚拟机充分利用空闲物理资源，推荐低优先级虚拟机 vCPU 范围绑核，且绑核范围覆盖高优先级虚拟机绑核范围。

同时为了防止出现因高优先级虚拟机长时间占满 CPU 导致低优先级虚拟机无法被调度的情况，需要预留少量低优先级虚拟机专用的 CPU，该部分 CPU 不可让高优先级虚拟机绑定，且要求让低优先级虚拟机绑定。
