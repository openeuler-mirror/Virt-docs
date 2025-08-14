# Skylark

- [Skylark](#skylark)
    - [Skylark Introduction](#skylark-introduction)
    - [Architecture and Features](#architecture-and-features)
    - [Skylark Installation](#skylark-installation)
    - [Skylark Configuration](#skylark-configuration)
    - [Skylark Usage](#skylark-usage)
    - [Best Practices](#best-practices)

## Skylark Introduction

### Scenario

With the rapid growth of the cloud computing market, cloud vendors are increasing their investment in cloud infrastructure. However, the industry still faces the problem of low resource utilization. Improving resource utilization has become an important technical subject. This document describes openEuler Skylark, as well as how to install and use it.

### Overview

Hybrid deployment of services of different priorities is a typical and effective method to improve resource utilization. Services can be classified into high-priority and low-priority services based on latency sensitivity. When high-priority services compete with low-priority services for resources, resources are preferentially provided for high-priority services. Therefore, the core technology of service hybrid deployment is resource isolation control, which involves kernel-mode basic resource isolation and user-mode QoS control.

This document describes the user-mode QoS control technology provided by Skylark of openEuler 22.09. In Skylark, the priority granularity is VMs. That is, a priority attribute is added to each VM. Resources are isolated and controlled based on VMs. Skylark is a QoS-aware resource scheduler in hybrid deployment scenarios. It improves physical machine resource utilization while ensuring the QoS of high-priority VMs.

For details about how to better use the priority feature of Skylark in actual application scenarios, see [Best Practices](#best-practices).

## Architecture and Features

### Overall Architecture 

The core class of Skylark is `QoSManager`. Class members include data collection class instances, QoS analysis class instances, QoS control class instances, and task scheduling class instances.

- `DataCollector`: data collection class. It has the `HostInfo` and `GuestInfo` members, which collect host information and VM information, respectively.
- `PowerAnalyzer`: power consumption analysis class, which analyzes power consumption interference and low-priority VMs to be restricted.
- `CpuController`: CPU bandwidth control class, which limits the CPU bandwidth of low-priority VMs.
- `CacheMBWController`: last-level cache (LLC) and memory bandwidth control class, which limits the LLC and memory bandwidth of low-priority VMs.
- `BackgroundScheduler`: task scheduling class, which periodically drives the preceding modules to continuously manage QoS.

After checking the host environment, Skylark creates a daemon process. The daemon has a main scheduling thread and one or more job threads.

- The main scheduling thread is unique. It connects to libvirt, creates and initializes the `QosManager` class instance, and then starts to drive the Job threads.
- Each Job thread periodically executes a QoS management task.

### Power Consumption Interference Control

Compared with non-hybrid deployment, host resource utilization is higher in hybrid deployment scenarios. High utilization means high power consumption. When the power consumption exceeds the thermal design power (TDP) of the server, CPU frequency reduction is triggered. When the power consumption exceeds the preset TDP (that is, a TDP hotspot occurs), Skylark limits the CPU bandwidth of low-priority VMs to reduce the power consumption of the entire system and ensure the QoS of high-priority VMs.

During initialization, Skylark sets the power consumption interference control attributes based on the related configuration values in [Skylark Configuration](#skylark-configuration). In each control period, host information and control attributes are comprehensively analyzed to determine whether TDP hotspots occur. If a hotspot occurs, Skylark analyzes the low-priority VMs whose CPU bandwidth needs to be limited based on the VM information.

### LLC/MB Interference Control

Skylark can limit the LLC and memory bandwidth of low-priority VMs. Currently, only static allocation is supported. Skylark uses the **/sys/fs/resctrl** interface provided by the OS to implement the limitation.

1. Skylark creates the **low_prio_machine** folder in the **/sys/fs/resctrl** directory and writes the PID of the low-priority VM to the **/sys/fs/resctrl/low_prio_machine/tasks** file.
2. Skylark allocates LLC ways and memory bandwidth for low-priority VMs based on the LLC/MB configuration items in [Skylark Configuration](#skylark-configuration). The configuration items are written into the **/sys/fs/resctrl/low_prio_machine/schemata** file.

### CPU Interference Control

In hybrid deployment scenarios, low-priority VMs generate CPU time slice interference and hardware hyper-threading (SMT) interference on high-priority VMs.

- When threads of high- and low-priority VMs are running on the same minimum CPU topology unit (core or SMT execution unit), they compete for CPU time slices.
- When threads of high- and low-priority VMs are running on different SMT execution units of the same CPU core at the same time, they compete for resources in the core shared by the SMT execution units.

CPU interference control includes CPU time slice interference control and SMT interference control, which are implemented based on the **QOS_SCHED** and **SMT_EXPELLER** features provided by the kernel, respectively.

- The **QOS_SCHED** feature enables high-priority VM threads on a single CPU core or SMT execution unit to suppress low-priority VM threads, eliminating CPU time slice interference.
- The **SMT_EXPELLER** feature enables high-priority VM threads to suppress low-priority VM threads on different SMT execution units of the same CPU core, eliminating SMT interference.

During initialization, Skylark sets the **cpu.qos_level** field of the slice level corresponding to the low-priority VM under the cgroup CPU subcontroller to -1 to enable the preceding kernel features. By doing this, the kernel controls CPU-related interference without the intervention of Skylark.

## Skylark Installation

### Hardware Requirements

Processor architecture: AArch64 or x86_64

- For Intel processors, the RDT function must be supported.
- For the AArch64 architecture, only Kunpeng 920 processor is supported, and the BIOS must be upgraded to 1.79 or later to support the MPAM function.

### Software Requirements

- python3, python3-APScheduler, and python3-libvirt
- systemd 249-32 or later
- libvirt 1.0.5 or later
- openEuler kernel 5.10.0 or later.

### Installation Procedure

You are advised to install the Skylark component using Yum for automatic processing of the software dependencies:

```shell
# yum install -y skylark
```

Check whether the Skylark is successfully installed. If the installation is successful, the skylarkd background service status is displayed:

```shell
# systemctl status skylarkd
```

(Optional) Enable the Skylark service to automatically start upon system startup:

```shell
# systemctl enable skylarkd
```

## Skylark Configuration

After the Skylark component is installed, you can modify the configuration file if the default configuration does not meet your requirements. The Skylark configuration file is stored in **/etc/sysconfig/skylarkd**. The following describes the configuration items in the configuration file.

### Logs

- The **LOG_LEVEL** parameter is a character string used to set the minimum log level. The supported log levels are **critical > error > warning > info > debug**. Logs whose levels are lower than **LOG_LEVEL** are not recorded in the log file **/var/log/skylark.log**. Skylark backs up logs every seven days for a maximum of four times. (When the number of backup times reaches the limit, the oldest logs are deleted.) The backup log is saved as **/var/log/skylark.log. %Y- %m- %d**.

### Power Consumption Interference Control

- **POWER_QOS_MANAGEMENT** is a boolean value used to control whether to enable power consumption QoS management. Only x86 processors support this function. This function is useful if the CPU usage of VMs on the host can be properly limited.

- **TDP_THRESHOLD** is a floating point number used to control the maximum power consumption of a VM. When the power consumption of the host exceeds **TDP * TDP_THRESHOLD**, a TDP hotspot occurs, and a power consumption control operation is triggered. The value ranges from 0.8 to 1, with the default value being 0.98.

- **FREQ_THRESHOLD** is a floating point number used to control the minimum CPU frequency when a TDP hotspot occurs on the host. The value ranges from 0.8 to 1, with the default value being 0.98.
    1. When the frequency of some CPUs is lower than **max_freq * FREQ_THRESHOLD**, Skylark limits the CPU bandwidth of low-priority VMs running on these CPUs.
    2. If such a CPU does not exist, Skylark limits the CPU bandwidth of some low-priority VMs based on the CPU usage of low-priority VMs.

- **QUOTA_THRESHOLD** is a floating point number used to control the CPU bandwidth that a restricted low-priority VM can obtain (CPU bandwidth before restriction x **QUOTA_THRESHOLD**). The value ranges from 0.8 to 1, with the default value being 0.9.

- **ABNORMAL_THRESHOLD** is an integer used to control the number of low-priority VM restriction periods. The value ranges from 1 to 5, with the default value being 3.
    1. In each power consumption control period, if a low-priority VM is restricted, its number of remaining restriction periods is updated to **ABNORMAL_THRESHOLD**.
    2. Otherwise, its number of remaining restriction periods decreases by 1. When the number of remaining restriction periods of the VM is 0, the CPU bandwidth of the VM is restored to the value before the restriction.

### LLC/MB Interference Control

Skylark's interference control on LLC/MB depends on the RDT/MPAM function provided by hardware. For Intel x86_64 processors, **rdt=cmt,mbmtotal,mbmlocal,l3cat,mba** needs to be added to kernel command line parameters. For Kunpeng 920 processors, **mpam=acpi** needs to be added to kernel command line parameters.

- **MIN_LLC_WAYS_LOW_VMS** is an integer used to control the number of LLC ways that can be accessed by low-priority VMs. The value ranges from 1 to 3, with the default value being 2. During initialization, Skylark limits the numfer of accessible LLC ways for low-priority VMs to this value.

- **MIN_MBW_LOW_VMS** is a floating point number used to control the memory bandwidth ratio available to low-priority VMs. The value ranges from 0.1 to 0.2, with the default value being 0.1. Skylark limits the memory bandwidth of low-priority VMs based on this value during initialization.

## Skylark Usage

### Starting the Service

Start Skylark for the first time:

```shell
# systemctl start skylarkd
```

Restart Skylark (a service restart is required after modifying the configuration file):

```shell
# systemctl restart skylarkd
```

### Creating VMs

Skylark uses the **partition** tag in the XML configuration file of a VM to identify the VM priority.

To create a low-priority VM, configure the XML file as follows:

```xml
<domain>
  ...
  <resource>
    <partition>/low_prio_machine</partition>
  </resource>
  ...
</domain>
```

To create a high-priority VM, configure the XML file as follows:

```xml
<domain>
  ...
  <resource>
    <partition>/high_prio_machine</partition>
  </resource>
  ...
</domain>
```

The subsequent VM creation process is the same as the normal process.

### Running VMs

Skylark detects VM creation events, manages VMs of different priorities, and performs automatic QoS management based on CPU, power consumption, and LLC/MB resources.

## Best Practices

### VM Service Recommendation

- High-priority VMs are suitable for latency-sensitive services, such as web services, high-performance databases, real-time rendering, and AI inference.
- Low-priority VMs are suitable for non-latency-sensitive services, such as video encoding, big data processing, offline rendering, and AI training.

### CPU Binding Configuration

To ensure optimal performance of high-priority VMs, you are advised to bind each vCPU of high-priority VMs to a physical CPU. To enable low-priority VMs to make full use of idle physical resources, you are advised to bind vCPUs of low-priority VMs to CPUs that are bound to high-priority VMs.

To ensure that low-priority VMs are scheduled when high-priority VMs occupy CPU resources for a long time, you are advised to reserve a small number of for low-priority VMs.
