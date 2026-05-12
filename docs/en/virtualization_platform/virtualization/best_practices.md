# Best Practices

## Performance Best Practices

### halt-polling

#### Overview

If compute resources are sufficient, the halt-polling feature can be used to enable VMs to obtain performance similar to that of physical machines. If the halt-polling feature is not enabled, the host allocates CPU resources to other processes when the vCPU exits due to idle timeout. When the halt-polling feature is enabled on the host, the vCPU of the VM performs polling when it is idle. The polling duration depends on the actual configuration. If the vCPU is woken up during the polling, the vCPU can continue to run without being scheduled from the host. This reduces the scheduling overhead and improves the VM system performance.

>![NOTE](./public_sys_resources/icon_note.gif)
> The halt-polling mechanism ensures that the vCPU thread of the VM responds in a timely manner. However, when the VM has no load, the host also performs polling. As a result, the host detects that the CPU usage of the vCPU is high, but the actual CPU usage of the VM is not high. 

#### Operation Guide

The halt-polling feature is enabled by default, and the default polling time is 500000 ns. You can dynamically change the halt-polling time of vCPU by modifying the `halt\_poll\_ns` file.

For example, to set the polling time to 400000, run the following command as the `root` user:

```Shell
# echo 400000 > /sys/module/kvm/parameters/halt_poll_ns
```

### I/O Thread Configuration

#### Overview

On the KVM platform, QEMU main threads process read and write operations on virtual disks at the backend by default. This causes the following issues:

- VM I/O requests are processed by a QEMU main thread. Therefore, the single-thread CPU usage becomes the bottleneck of VM I/O performance.
- The QEMU global lock \(`qemu\_global\_mutex\`) is used when VM I/O requests are processed by the QEMU main thread. If the I/O processing takes a long time, the QEMU main thread will occupy the global lock for a long time. As a result, the VM vCPU cannot be scheduled properly, affecting the overall VM performance and user experience.

You can configure the I/O thread attribute for the virtio-blk disk or virtio-scsi controller. At the QEMU backend, an I/O thread is used to process read and write requests of a virtual disk. The mapping relationship between the I/O thread and the virtio-blk disk or virtio-scsi controller can be a one-to-one relationship to minimize the impact on the QEMU main thread, enhance the overall I/O performance of the VM, and improve user experience.

#### Configuration Description

To use I/O threads to process VM disk read and write requests, you need to modify VM configurations as follows:

- Configure the total number of high-performance virtual disks on the VM. For example, set `<iothreads\>` to `4` to control the total number of I/O threads.

    ```Conf
    <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>   
         <name>VMName</name>
         <memory>4194304</memory>
         <currentMemory>4194304</currentMemory>
         <vcpu>4</vcpu>
         <iothreads>4</iothreads>
    ```

- Configure the I/O thread attribute for the virtio-blk disk. `<iothread\>` indicates I/O thread IDs. The IDs start from 1 and each ID must be unique. The maximum ID is the value of `<iothreads\>`. For example, to allocate I/O thread 2 to the virtio-blk disk, set parameters as follows:

    ```Conf
    <disk type='file' device='disk'>
          <driver name='qemu' type='raw' cache='none' io='native' iothread='2'/>
          <source file='/path/test.raw'/>
          <target dev='vdb' bus='virtio'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
    ```

- Configure the I/O thread attribute for the virtio-scsi controller. For example, to allocate I/O thread 2 to the virtio-scsi controller, set parameters as follows:

    ```Conf
    <controller type='scsi' index='0' model='virtio-scsi'>
          <driver iothread='2'/>
          <alias name='scsi0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </controller>
    ```

- Bind I/O threads to a physical CPU.

    Binding I/O threads to specified physical CPUs does not affect the resource usage of vCPU threads. `<iothread\>` indicates I/O thread IDs, and `<cpuset\>` indicates IDs of the bound physical CPUs.

    ```Conf
    <cputune>
    <iothreadpin iothread='1' cpuset='1-3,5,7-12' />
    <iothreadpin iothread='2' cpuset='1-3,5,7-12' />
    </cputune>
    ```

### Raw Device Mapping

#### Overview

When configuring VM storage devices, you can use configuration files to configure virtual disks for VMs, or connect block devices (such as physical LUNs and logical volumes) to VMs for use to improve storage performance. The latter configuration method is called raw device mapping (RDM). Through RDM, a virtual disk is presented as a small computer system interface (SCSI) device to the VM and supports most SCSI commands.

RDM can be classified into virtual RDM and physical RDM based on backend implementation features. Compared with virtual RDM, physical RDM provides better performance and more SCSI commands. However, for physical RDM, the entire SCSI disk needs to be mounted to a VM for use. If partitions or logical volumes are used for configuration, the VM cannot identify the disk.

#### Configuration Example

VM configuration files need to be modified for RDM. The following are configuration examples.

- Virtual RDM

    The following is an example of mounting the SCSI disk `/dev/sdc` on the host to the VM as a virtual raw device:

    ```Conf
    <domain type='kvm'>
     <devices>
        ...
        <controller type='scsi' model='virtio-scsi' index='0'/>
        <disk type='block' device='disk'>
            <driver name='qemu' type='raw' cache='none' io='native'/>
            <source dev='/dev/sdc'/>
            <target dev='sdc' bus='scsi'/>
            <address type='drive' controller='0' bus='0' target='0' unit='0'/>
        </disk>
        ...
     </devices>
    </domain>
    ```

- Physical RDM

    The following is an example of mounting the SCSI disk `/dev/sdc` on the host to the VM as a physical raw device:

    ```Conf
    <domain type='kvm'>
     <devices>
        ...
        <controller type='scsi' model='virtio-scsi' index='0'/>
        <disk type='block' device='lun' rawio='yes'>
            <driver name='qemu' type='raw' cache='none' io='native'/>
            <source dev='/dev/sdc'/>
            <target dev='sdc' bus='scsi'/>
            <address type='drive' controller='0' bus='0' target='0' unit='0'/>
        </disk>
        ...
     </devices>
    </domain>
    ```

### kworker Isolation and Binding

#### Overview

kworker is a per-CPU thread implemented by the Linux kernel. It is used to execute workqueue requests in the system. kworker threads will compete for physical core resources with vCPU threads, resulting in virtualization service performance jitter. To ensure that the VM can run stably and reduce the interference of kworker threads on the VM, you can bind kworker threads on the host to a specific CPU.

#### Procedure

You can modify the `/sys/devices/virtual/workqueue/cpumask` file to bind tasks in the workqueue to the CPU specified by `cpumask`. Masks in `cpumask` are in hexadecimal format. For example, if you need to bind kworker to CPU0 to CPU7, run the following command as the `root` user to change the mask to `ff`:

```Shell
# echo ff > /sys/devices/virtual/workqueue/cpumask
```

### HugePage Memory

#### Overview

Compared with traditional 4 KB memory paging, openEuler also supports 2 MB/1 GB memory paging. HugePage memory can effectively reduce TLB misses and significantly improve the performance of memory-intensive services. openEuler uses two technologies to implement HugePage memory.

- Static HugePage

    The static HugePage requires that a static HugePage pool be reserved before the host OS is loaded. When creating a VM, you can modify the XML configuration file to specify that the VM memory is allocated from the static HugePage pool. The static HugePage ensures that all memory of a VM exists on the host as the HugePage to ensure physical continuity. However, the deployment difficulty is increased. After the page size of the static HugePage pool is changed, the host needs to be restarted for the change to take effect. The size of a static HugePage can be 2 MB or 1 GB.

- Transparent HugePage

    If the transparent HugePage (THP) mode is enabled, the VM automatically selects available 2 MB consecutive pages and automatically splits and combines HugePages when allocating memory. When no 2 MB consecutive pages are available, the VM selects available 64 KB (AArch64 architecture) or 4 KB (x86_64 architecture) pages for allocation. By using THP, users do not need to be aware of it and 2 MB HugePages can be used to improve memory access performance.

If VMs use static HugePages, you can disable THP to reduce the overhead of the host OS and ensure stable VM performance.

#### Operation Guide

- Configure static HugePages.

    Before creating a VM, modify the XML file to configure a static HugePage for the VM.

    ```Conf
      <memoryBacking>
        <hugepages>
          <page size='1' unit='GiB'/>
        </hugepages>
      </memoryBacking>
    ```

    The preceding XML segment indicates that a 1 GB static HugePage is configured for the VM.

    ```Conf
      <memoryBacking>
        <hugepages>
          <page size='2' unit='MiB'/>
        </hugepages>
      </memoryBacking>
    ```

    The preceding XML segment indicates that a 2 MB static HugePage is configured for the VM.

- Configure the THP.

    Dynamically enable the THP through sysfs.

    ```Shell
    # echo always > /sys/kernel/mm/transparent_hugepage/enabled
    ```

    Dynamically disable the THP.

    ```Shell
    # echo never > /sys/kernel/mm/transparent_hugepage/enabled
    ```

### Memory Bandwidth Monitoring

#### Overview

When VMs of different tenants run on the same host and large-specification memory-intensive VMs occupy a large amount of memory bandwidth, the memory bandwidth of other VMs cannot meet service requirements. The MPAM of Kunpeng 920B and the resctrl function provided by the system can be used to detect and control the memory bandwidth usage for a certain number of VMs (up to 30).

To enable this function, add the boot parameter `mpam=acpi` to the `grub.cfg` file of the host.

#### Operation Guide

- Memory bandwidth control

    Configure L3PRI.

    ```Conf
      <cputune>
        ...
        <cachetune vcpus='0-3'>
          <cache id='0' level='3' type='priority' size='1'/>
        </cachetune>
        ...
      </cputune>
    ```

    - The preceding XML file is used to configure the L3 cache priority. A larger value indicates a higher priority. The default value of L3PRI is 0. The valid value range is [0, 3].

    Configure the bandwidth limit.

    ```Conf
      <cputune>
        ...
        <memorytune>
          <node id='0' bandwidth='100' min_bandwidth='50' hardlimit='1' priority='3'/>
        </memorytune>
        ...
      </cputune>
    ```

    - `bandwidth`: upper limit of the memory bandwidth. The value range is [0, 100].
    - `min_bandwidth`: lower limit of the memory bandwidth. When the actual proportion of a shared resource is lower than the configured value, the priority of using the resource automatically increases. The value range is [0, 100].
    - `hardlimit`: The value can be 0 or 1. When MBHDL is set to 1, the usage of MB shared resources cannot exceed the configured MB value, that is, `bandwidth`. When MBHDL is set to 0, the usage of MB shared resources is allowed to exceed the configured MB value in idle scenarios.
    - `priority`: A larger value indicates a higher priority. The default value of MBPRI is 3. The valid value range is [0, 7].
    - The preceding four fields can be configured separately.

    After the preceding XML file is configured for a VM, a control group for the VM is created in the `resctrl` directory. The schemata in the control group records the corresponding configuration.

- Memory bandwidth monitoring

    Memory bandwidth monitoring depends on the MPAM feature. After the preceding XML file is configured, you can run the following command to monitor the VM memory bandwidth.

    ```Shell
    # vmtop -G
    ```

### PV-qspinlock

#### Overview

PV-qspinlock is an optimization of spinlocks in virtualization CPU overcommitment scenarios. It allows a hypervisor to block a vCPU in the lock context and wake up the corresponding vCPU after the lock is released. In overcommitment scenarios, PV-qspinlock can better utilize pCPU resources and optimize the compilation process, reducing the time required for compiling applications.

#### Operation Guide

Modify the `/boot/efi/EFI/openEuler/grub.cfg` configuration file of a VM by adding `arm_pvspin` to the command-line startup parameter. The modification takes effect after the VM is restarted. After PV-qspinlock takes effect, you can run the `dmesg` command on the VM to find the following log:

```text
[    0.000000] arm-pv: PV qspinlocks enabled
```

>![NOTE](./public_sys_resources/icon_note.gif)
>PV-qspinlock is supported only when both the host and VM run openEuler 20.09 or later and the VM kernel compilation option is set as `CONFIG_PARAVIRT_SPINLOCKS=y` (default configuration on openEuler).

### Guest-Idle-Haltpoll

#### Overview

To ensure fairness and reduce power consumption, when the vCPUs of a VM are idle, the VM executes the WFx/HLT instruction to exit the host machine and triggers a context switch. The host machine determines whether to schedule other processes or vCPUs on the physical CPU or enter the energy saving mode. However, switching between the VM and the host machine, additional context switches, and IPI interrupt wakeup cause relatively high overhead, and this problem is particularly prominent in a service of frequent sleep and wakeup. The Guest-Idle-Haltpoll technology means that when a VM vCPU is idle, it does not immediately execute WFx/HLT and trigger a VM-exit, but instead, the vCPU performs polling for a period of time within the VM. During this period, tasks of other vCPUs that share the LLC are woken up on the vCPU without the need to send IPI interrupts, reducing the overhead of sending and receiving IPIs and the VM-exit overhead. This reduces the task wakeup latency.

>![NOTE](./public_sys_resources/icon_note.gif)
>Enabling idle-haltpoll for a vCPU within a VM increases the CPU overhead of the vCPU on the host machine. Therefore, it is recommended that the vCPU exclusively occupy a physical core on the host machine when this feature is enabled.

#### Operation Guide

The Guest-Idle-Haltpoll feature is disabled by default. The following describes how to enable this feature.

1. Enable the Guest-Idle-Haltpoll feature.
    - If the host machine uses the x86 processor architecture, you can enable this feature by configuring `hint-dedicated` in the VM XML file of the host machine. The VM XML configuration transfers the status of the vCPU exclusively occupying a physical core to the VM. The host machine ensures that the vCPU exclusively occupies a physical core.

        ```Conf
        <domain type='kvm'>
         ...
         <features>
           <kvm>
             ...
             <hint-dedicated state='on'/>
           </kvm>
         </features>
          ...
        </domain>
        ```

        Alternatively, you can configure `cpuidle\_haltpoll.force=Y` in the VM kernel startup parameters to forcibly enable this feature. This method does not require you to configure the vCPU to exclusively occupy a physical core on the host machine.

        ```Conf
        cpuidle_haltpoll.force=Y
        ```

    - If the host machine uses the AArch64 processor architecture, you can enable this feature only by configuring `cpuidle\_haltpoll.force=Y haltpoll.enable=Y` in the VM kernel startup parameters.

        ```Conf
        cpuidle_haltpoll.force=Y haltpoll.enable=Y
        ```

2. Check whether the Guest-Idle-Haltpoll feature has taken effect. Run the following command on the VM. If `haltpoll` is displayed, the feature has taken effect.

    ```Shell
    # cat /sys/devices/system/cpu/cpuidle/current_driver
    ```

3. (Optional) Configure Guest-Idle-Haltpoll parameters.
    The following configuration files are provided in the `/sys/module/haltpoll/parameters/` path of the VM to adjust configuration parameters. You can adjust the parameters based on service characteristics.

    - `guest\_halt\_poll\_ns`: a global parameter that specifies the maximum polling duration after a vCPU is idle. The default value is 200000 ns.
    - `guest\_halt\_poll\_shrink`: a divisor used to shrink `guest\_halt\_poll\_ns` of the current vCPU when a wakeup event occurs after the global `guest\_halt\_poll\_ns`. The default value is 2.
    - `guest\_halt\_poll\_grow`: a multiplier used to extend `guest\_halt\_poll\_ns` of the current vCPU when a wakeup event occurs after `guest\_halt\_poll\_ns` of the current vCPU and before the global `guest\_halt\_poll\_ns`. The default value is 2.
    - `guest\_halt\_poll\_grow\_start`: When the system is idle, `guest\_halt\_poll\_ns` of each vCPU eventually reaches zero. This parameter is used to set the initial value of `guest\_halt\_poll\_ns` of the current vCPU so that the vCPU polling duration can be shrunk or extended. The default value is 50000 ns.
    - `guest\_halt\_poll\_allow\_shrink`: whether to allow `guest\_halt\_poll\_ns` of each vCPU to be shrunk. The default value is `Y` (`Y` indicates that shrink is allowed, and `N` indicates that shrink is not allowed).

    You can run the following command as the `root` user to change the parameter value: In the command, _value_ indicates the parameter value to be set, and _configFile_ indicates the corresponding configuration file.

    ```Shell
    # echo value > /sys/module/haltpoll/parameters/configFile
    ```

    For example, to set the global `guest\_halt\_poll\_ns` to 200000 ns, run the following command:

    ```Shell
    # echo 200000 > /sys/module/haltpoll/parameters/guest_halt_poll_ns
    ```

### NVMe Drive Passthrough

#### Overview

The device passthrough technology is a hardware-based virtualization solution. With this technology, VMs can be directly connected to specified physical passthrough devices. To improve VM storage performance, you can use the PCI passthrough technology to pass through NVMe drives to VMs.

#### Operation Guide

1. Prepare for the use.

    - Ensure that the driver provided by the NVMe drive vendor is installed in the guest OS. Otherwise, the NVMe drive cannot work properly.
    - Ensure that the VT-d and VT-x support of the CPU is enabled on the host OS.
    - Ensure that the IOMMU function of the kernel is enabled on the host OS.
    - Ensure that the interrupt remapping function of the kernel is enabled on the host OS.

2. Obtain the PCI BDF information of an NVMe drive.

    Run the `lspci` command on the host to obtain the resource list of PCI devices on the host.

    ```Shell
    # lspci -vmm
    Slot: 81:00.1
    Class: Non-Volatile memory controller
    ...
    ```

    In the command output, `Slot` indicates the PCI BDF number of the NVMe drive, `81` indicates the bus number, `00` indicates the slot number, and `1` indicates the function number.

3. Mount a PCI passthrough NVMe drive to a VM.

    When creating a VM, add the PCI NVMe drive passthrough configuration option to the corresponding XML configuration file. The following is an example of the configuration file:

    ```Conf
    <hostdev mode='subsystem' type='pci' managed='yes'>
        <source>
            <address domain='0x0000' bus='0x81' slot='0x00' function='0x1' />
        </source>
    </hostdev>
    ```

    - `hostdev.source.address.domain`: domain number of the PCI device on the host OS.
    - `hostdev.source.address.bus`: bus number of the PCI device on the host OS.
    - `hostdev.source.address.slot`: slot number of the PCI device on the host OS.
    - `hostdev.source.address.function`: function number of the PCI device on the host OS.

4. Specify a PCI BAR of the NVMe drive.

    To further maximize the performance of the NVMe drive, you need to specify a BAR for PCI MSI-X interrupts of the passthrough NVMe drive in the guest OS. The configuration is as follows:

    ```Conf
    <hostdev mode='subsystem' type='pci' managed='yes'>
        <source>
            <address domain='0x0000' bus='0x01' slot='0x00' function='0x0' />
        </source>
        <alias name='ua-sm2262'/>
                <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
    </hostdev>
        <qemu:commandline>
            <qemu:arg value='-set'/>
            <qemu:arg value='device.ua-sm2262.x-msix-relocation=bar2'/>
        </qemu:commandline>
    ```

    In the preceding XML configuration, the interrupt information of the passthrough NVMe drive is processed on BAR 2. After this configuration is added, the performance of the NVMe drive in the guest OS is almost the same as the performance of the NVMe drive in the host OS.

### Transparent Transmission of Hardware Topology

#### Overview

CPU topology information shows how CPU cores are organized hierarchically on hardware, such as sockets, clusters, cores, and threads. The CPU topology information is provided to the kernel through the Advanced Configuration and Power Interface (ACPI) or Device Tree (DT). In virtualization scenarios, the ACPI or DT of a VM is generated by a virtualization component. The virtualization component generates the ACPI or DT based on the user-defined CPU topology, and loads them, together with the VM kernel, to the VM memory address space. In this way, the VM can detect the CPU topology information to make better task scheduling decisions.

#### Operation Guide

Add the CPU topology information to the XML file of the VM.

```Conf
<vcpu placement='static' current='4'>32</vcpu>
    <cpu mode='host-passthrough' check='none'>
    <topology sockets='1' clusters='4' cores='4' threads='2'/>
</cpu>
```

According to the XML `<vcpu>` tag, `32` indicates the maximum number of CPUs on the VM. The `topology` tag specifies the vCPU topology information. The value of `sockets * clusters * cores * threads` must be equal to the maximum number of CPUs on the VM.

After the VM is started, you can view the CPU topology information of the VM in the `/sys/devices/system/cpu` path.

### vCPU Core Binding

#### Overview

A vCPU is pinned to a physical CPU and can be scheduled only on this physical CPU. This improves VM performance in some scenarios. Otherwise, the vCPU can run on any physical CPU by default, which may cause interference across VMs or across vCPUs on the same VM. In the many-core scenario, you can bind each vCPU to a physical CPU to optimize vCPU performance.

#### Operation Guide

```Conf
<cputune>
    <vcpupin vcpu='0' cpuset='20'/>
    <vcpupin vcpu='1' cpuset='21'/>
    ......
</cputune>
```

In the XML example, `vcpu` indicates the vCPU ID, and `cpuset` indicates the ID of the physical CPU to which the vCPU is to be pined.

You can run the `virsh vpuinfo vmname` command on the host to check the mapping between vCPUs and physical CPUs.

### NUMA Affinity

#### Overview

Before starting the VM, you can specify NUMA nodes for VM memory in the VM configuration file. This improves VM performance by preventing remote memory access. You can also configure virtual NUMA to expose multiple NUMA nodes to the VM, so that the VM can recognize NUMA differences and prevent cross-node access.

#### Operation Guide

```Conf
<cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
</cputune>
<numatune>
    <memnode cellid="0" mode="strict" nodeset="0"/>
    <memnode cellid="1" mode="strict" nodeset="1"/>
</numatune>
```

In `<numatune>`, `cellid` indicates the NUMA ID of a VM. `mode` can be set to: `strict` (which means that memory must be allocated exclusively from specified node. If the node cannot satisfy the request, the allocation fails.); `preferred` (which means that memory is preferably allocated from the specified node, but may fall back to other nodes if necessary); or `interleave` (which means that memory is allocated across specified nodes.). `nodeset` indicates a specified physical NUMA node. In `<cputune>`, the vCPUs within the same `cellid` must be pinned to the physical NUMA node specified by `memnode`.

### WFI-no-trap

#### Overview

Based on the Virtual Software Generated Interrupt (vSGI) passthrough feature of GICv4.1, a Kernel-based VM (KVM) is configured to avoid trapping Wait For Interrupt (WFI) instructions. As a result, when a vCPU thread enters an idle state and executes WFI, it no longer traps into KVM. This eliminates VM exits and VM entries, thereby reducing virtualization overhead and in-guest latency.

#### Operation Guide

Configure the interrupt passthrough parameter in `cmdline`.

```Shell
kvm-arm.vgic_v4_enable=1
```

Disable the KVM WFI trap switch.

```Shell
echo N > /sys/module/kvm/parameters/force_wfi_trap
```

After creating a VM, use `vmtop` to check whether any WFI traps have occurred.

### NIC Passthrough

#### Overview

NIC passthrough is an application of the PCI passthrough technology. The PCI passthrough is a hardware-assisted virtualization solution. It allows VMs to directly access physical PCI devices, reducing virtualization overhead.

#### Operation Guide

To enable PCI passthrough for devices like Huawei Hi1822 NIC on a VM, follow these steps:

1. Obtain PCI BDF information of a device.
   You can run the `lspci | grep Eth` command on the host to obtain the NIC resource list of the current board. For example, the PCI BDF number `03:00.0` identifies a port on the Huawei Hi1822 4 x 25GE NIC.

   ```Shell
    03:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
    04:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
    05:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
    06:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
    ```

2. Assign PCI passthrough NICs to a VM.
   When creating a VM, add a PCI passthrough entry for the NICs to the VM configuration file:

   ```Conf
    <devices>
    ...
    <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
        <address domain='0x0000' bus='0x03' slot='0x10' function='0x00'/>
    </source>
    <rom bar='on'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </hostdev>
    ...
    </devices>
    ```

### NUMA Exposure for Passthrough Devices

#### Overview

On a VM, you can use the `sysfs` interface to view the NUMA node where a passthrough device resides. This allows you to deploy service applications based on the NUMA node where a device resides, reducing performance loss caused by cross-NUMA resource access and improving the performance of service applications on the VM.

#### Operation Guide

- XML configuration for NUMA information of passthrough devices:

   ```Conf
    <devices>
    ...
    <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
        <address domain='0x0000' bus='0x03' slot='0x10' function='0x00'/>
    </source>
    <numa node='0'>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </hostdev>
    ...
    </devices>
    ```

- View the NUMA node where a passthrough device resides in a VM.

    ```Shell
    # cat /sys/bus/pci/devices/bdf/numa_node
    ```

## Security Best Practices

### Libvirt Authentication

#### Overview

When a user uses libvirt remote invocation but no authentication is performed, any third-party program that connects to the host's network can operate VMs through the libvirt remote invocation mechanism. This poses security risks. To improve system security, openEuler provides the libvirt authentication function. That is, users can remotely invoke a VM through libvirt only after identity authentication. Only specified users can access the VM, thereby protecting VMs on the network.

#### Enabling Libvirt Authentication

By default, the libvirt remote invocation function is disabled on openEuler. The following describes how to enable the libvirt remote invocation and libvirt authentication functions.

1. Log in to a host as the `root` user.
2. Modify the libvirt service configuration file `/etc/libvirt/libvirtd.conf` to enable the libvirt remote invocation and libvirt authentication functions. For example, to enable the TCP remote invocation that is based on the Simple Authentication and Security Layer (SASL) framework, configure parameters by referring to the following:

    ```Conf
    # Transport layer security protocol. `0` indicates that the protocol is disabled, and `1` indicates that the protocol is enabled. You can set the value as needed.
    listen_tls = 0
    # Enable the TCP remote invocation. To enable the libvirt remote invocation and libvirt authentication functions, set the parameter to `1`.    
    listen_tcp = 1
    # User-defined protocol configuration for TCP remote invocation. The following uses `sasl` as an example.   
    auth_tcp = "sasl" 
    ```

3. Modify the `/etc/sasl2/libvirt.conf` configuration file to set the SASL mechanism and SASLDB.

    ```Conf
    # Authentication mechanism of the SASL framework
    mech_list: digest-md5
    # Database for storing usernames and passwords
    sasldb_path: /etc/libvirt/passwd.db
    ```

4. Add the user for SASL authentication and set the password. Take the user `userName` as an example. The command is as follows:

    ```Shell
    # saslpasswd2 -a libvirt userName
    Password:
    Again (for verification):
    ```

5. Modify the `/etc/sysconfig/libvirtd` configuration file to enable the libvirt listening option.

    ```Conf
    LIBVIRTD_ARGS="--listen"
    ```

6. Restart the libvirtd service to make the modification take effect.

    ```Shell
    # systemctl restart libvirtd
    ```

7. Check whether the authentication function for libvirt remote invocation takes effect. Enter the username and password as prompted. If the libvirt service is successfully connected, the function is successfully enabled.

    ```Shell
    # virsh -c qemu+tcp://192.168.0.1/system
    Please enter your authentication name: openeuler
    Please enter your password:
    Welcome to virsh, the virtualization interactive terminal.
    
    Type:  'help' for help with commands
           'quit' to quit
    
    virsh #
    ```

#### Managing SASL

The following describes how to manage SASL users. Perform the operations as the `root` user.

- Query an existing user in the database.

    ```Shell
    # sasldblistusers2 -f /etc/libvirt/passwd.db
    user@localhost.localdomain: userPassword
    ```

- Delete a user from the database.

    ```Shell
    # saslpasswd2 -a libvirt -d user
    ```

### qemu-ga

#### Overview

QEMU guest agent (qemu-ga) is a daemon running within VMs. It allows users on a host OS to perform various management operations on the guest OS through outband channels provided by QEMU. The operations include file operations (open, read, write, close, seek, and flush), internal shutdown, VM suspend (suspend-disk, suspend-ram, and suspend-hybrid), and obtaining of VM internal information (including the memory, CPU, NIC, and OS information).

In some scenarios with high security requirements, qemu-ga provides the blacklist function to prevent internal information leakage of VMs. You can use a blacklist to selectively shield some functions provided by qemu-ga.

>![NOTE](./public_sys_resources/icon_note.gif)
>The qemu-ga installation package is `qemu-guest-agent-xx.rpm`. It is not installed on openEuler by default. `xx` indicates the actual version number. 

#### Operation Method

To add a qemu-ga blacklist, perform the following steps as the `root` user:

1. Log in to the VM and ensure that the qemu-guest-agent service exists and is running.

    ```Shell
    # systemctl status qemu-guest-agent |grep Active
       Active: active (running) since Wed 2018-03-28 08:17:33 CST; 9h ago
    ```

2. Query which `qemu-ga` commands can be added to the blacklist:

    ```Shell
    # qemu-ga --blacklist ?
    guest-sync-delimited
    guest-sync
    guest-ping
    guest-get-time
    guest-set-time
    guest-info
    ...
    ```

3. Set the blacklist. Add the commands to be shielded to `--blacklist` in the `/usr/lib/systemd/system/qemu-guest-agent.service` file. Use spaces to separate different commands. For example, to add the `guest-file-open` and `guest-file-close` commands to the blacklist, configure the file by referring to the following:

    ```Conf
    [Service]
    ExecStart=-/usr/bin/qemu-ga \
          --blacklist=guest-file-open guest-file-close
    ```

4. Restart the qemu-guest-agent service.

    ```Shell
    # systemctl daemon-reload
    # systemctl restart qemu-guest-agent
    ```

5. Check whether the qemu-ga blacklist function takes effect on the VM, that is, whether the `--blacklist` parameter configured for the qemu-ga process is correct.

    ```Shell
    # ps -ef|grep qemu-ga|grep -E "blacklist=|b="
    root       727     1  0 08:17 ?        00:00:00 /usr/bin/qemu-ga --method=virtio-serial --path=/dev/virtio-ports/org.qemu.guest_agent.0 --blacklist=guest-file-open guest-file-close guest-file-read guest-file-write guest-file-seek guest-file-flush -F/etc/qemu-ga/fsfreeze-hook
    ```

    >![NOTE](./public_sys_resources/icon_note.gif)
    >For more information about qemu-ga, visit [https://wiki.qemu.org/Features/GuestAgent](https://wiki.qemu.org/Features/GuestAgent). 

### sVirt Protection

#### Overview

In a virtualization environment that uses the discretionary access control (DAC) policy only, malicious VMs running on hosts may attack the hypervisor or other VMs. To improve security in virtualization scenarios, openEuler uses sVirt for protection. sVirt is a security protection technology based on SELinux. It is applicable to KVM virtualization scenarios. A VM is a common process on the host OS. In the hypervisor, the sVirt mechanism labels QEMU processes corresponding to VMs with SELinux labels. In addition to types which are used to label virtualization processes and files, different categories (in the seclevel range) are used to label different VMs. Each VM can access only file devices of the same category. This prevents VMs from accessing files and devices on unauthorized hosts or other VMs, thereby preventing VM escape and improving host and VM security.

#### Enabling sVirt Protection

##### I. Perform the following steps as the root user to enable SELinux on the host

1. Log in to the host.
2. Enable the SELinux function on the host.
    1. Modify the system startup parameter file `grub.cfg` to set `selinux` to `1`.

        ```Conf
        selinux=1
        ```

    2. Modify `/etc/selinux/config` to set the `SELINUX` to `enforcing`.

        ```Conf
        SELINUX=enforcing
        ```

3. Restart the host.

    ```Shell
    # reboot
    ```

##### II. Create a VM with the sVirt function enabled

1. Add the following information to the VM configuration file:

    ```Conf
    <seclabel type='dynamic' model='selinux' relabel='yes'/>
    ```

    Or check whether the following configuration exists in the file:

    ```Conf
    <seclabel type='none' model='selinux'/>
    ```

2. Create a VM.

    ```Shell
    # virsh define openEulerVM.xml
    ```

##### III. Verify that sVirt is enabled

Run the following command to check whether sVirt protection has been enabled for the QEMU process of the running VM. If `svirt\_t:s0:c` exists, sVirt protection has been enabled.

```Shell
# ps -eZ|grep qemu |grep "svirt_t:s0:c"
system_u:system_r:svirt_t:s0:c200,c947 11359 ? 00:03:59 qemu-kvm
system_u:system_r:svirt_t:s0:c427,c670 13790 ? 19:02:07 qemu-kvm
```

### Trusted VM Boot

#### Overview

Trusted boot includes measured boot and remote attestation. The virtualization component mainly provides the measured boot function. Remote attestation is enabled by users by installing related software (RA client) on the VM and setting up a remote attestation server (RA server).

The two basic elements of measured boot are the root of trust (RoT) and chain of trust. The fundamental idea is to establish a RoT in the computer system to act as the Core Root of Trust for Measurement (CRTM). The credibility of the RoT is ensured from the aspects of physical security, technical security, and management security. Then, a chain of trust is established, starting from the RoT, through the BIOS/BootLoader and operating system, to applications. In this way, measurement, authentication, and trust are implemented level by level to extend trust throughout the system. This process is like a chain, so it is called a chain of trust.

The CRTM is the root of measured boot and the first component to start in the system. There is no other code to check the integrity of the CRTM itself. Therefore, as the starting point in the chain of trust, it must be an absolutely trusted source. Therefore, the CRTM needs to be designed as read-only code or code with strictly limited updates to defend against BIOS attacks and prevent remote injection of malicious code or modification of the boot code at the upper layer of the operating system. In a physical host, the microcode in the CPU is usually used as the CRTM. In a virtualization environment, the SEC section of the vBIOS is usually used as the CRTM.

During the boot process, the previous component measures (calculates the hash value) the next component and then extends the measurement value to a trusted storage area, such as the PCR of the TPM. The CRTM measures the BootLoader and extends the measurement value to the PCR. The BootLoader measures the OS and extends the measurement value to the PCR.

#### Configuring a vTPM Device and Enabling Measured Boot

##### I. Install the swtpm and libtpms software

swtpm provides a TPM emulator (TPM 1.2 or TPM 2.0) that can be integrated into a virtualization environment. So far, it has been integrated into QEMU and also used as a prototype system in RunC. swtpm uses libtpms to provide the simulation functions of TPM 1.2 and TPM 2.0.
Currently, openEuler 21.03 provides the sources of libtpms and swtpm, which can be installed using yum commands.

```Shell
# yum install libtpms swtpm swtpm-devel swtpm-tools
```

##### II. Configure a vTPM Device for a VM

1. Add the following information to the VM configuration file:

    ```Conf
    <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
    ...
    <devices>
        ...
        <tpm model='tpm-tis'>
        <backend type='emulator' version='2.0'/>
        </tpm>
        ...
    </devices>
            ...
    </domain>
    ```

2. Create a VM.

    ```Shell
    # virsh define MeasuredBoot.xml
    ```

3. Start the VM.

    Before starting the VM, run the `chmod` command to grant the following permissions to the `/var/lib/swtpm-localca/` directory. Otherwise, libvirt cannot start swtpm.

    ```Shell
        # chmod -R 777 /var/lib/swtpm-localca/
        #
    # virsh start MeasuredbootVM
    ```

##### III. Verify that measured boot is enabled successfully

Whether measured boot is enabled is determined by the vBIOS. Currently, the vBIOS in openEuler 21.03 supports measured boot. If the host uses the edk2 component of another version, check whether it supports measured boot.

Log in to the VM as the `root` user and check whether the TPM driver, tpm2-tss protocol stack, and tpm2-tools tool are installed on the VM.
In openEuler 21.03, the TPM driver (`tpm_tis.ko`), tpm2-tss protocol stack, and tpm2-tools tool are installed by default. If another OS is used, run the following commands to check whether the driver and related tools are installed:

```Shell
# lsmod |grep tpm
# tpm_tis          16384   0
#
# yum list installed | grep -E 'tpm2-tss|tpm2-tools'
#
# yum install tpm2-tss tpm2-tools
```

You can run the `tpm2_pcrread` command (or the `tpm2_pcrlist` command in earlier versions of tpm2_tools) to list all PCR values.

```Shell
# tpm2_pcrread
sha1 :
  0  : fffdcae7cef57d93c5f64d1f9b7f1879275cff55
  1  : 5387ba1d17bba5fdadb77621376250c2396c5413
  2  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
  3  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
  4  : e5d40ace8bb38eb170c61682eb36a3020226d2c0
  5  : 367f6ea79688062a6df5f4737ac17b69cd37fd61
  6  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
  7  : 518bd167271fbb64589c61e43d8c0165861431d8
  8  : af65222affd33ff779780c51fa8077485aca46d9
  9  : 5905ec9fb508b0f30b2abf8787093f16ca608a5a
  10 : 0000000000000000000000000000000000000000
  11 : 0000000000000000000000000000000000000000
  12 : 0000000000000000000000000000000000000000
  13 : 0000000000000000000000000000000000000000
  14 : 0000000000000000000000000000000000000000
  15 : 0000000000000000000000000000000000000000
  16 : 0000000000000000000000000000000000000000
  17 : ffffffffffffffffffffffffffffffffffffffff
  18 : ffffffffffffffffffffffffffffffffffffffff
  19 : ffffffffffffffffffffffffffffffffffffffff
  20 : ffffffffffffffffffffffffffffffffffffffff
  21 : ffffffffffffffffffffffffffffffffffffffff
  22 : ffffffffffffffffffffffffffffffffffffffff
  23 : 0000000000000000000000000000000000000000
sha256 :
  0  : d020873038268904688cfe5b8ccf8b8d84c1a2892fc866847355f86f8066ea2d
  1  : 13cebccdb194dd916f2c0c41ec6832dfb15b41a9eb5229d33a25acb5ebc3f016
  2  : 3d458cfe55cc03ea1f443f1562beec8df51c75e14a9fcf9a7234a13f198e7969
  3  : 3d458cfe55cc03ea1f443f1562beec8df51c75e14a9fcf9a7234a13f198e7969
  4  : 07f9074ccd4513ef1cafd7660f9afede422b679fd8ad99d25c0659eba07cc045
  5  : ba34c80668f84407cd7f498e310cc4ac12ec6ec43ea8c93cebb2a688cf226aff
  6  : 3d458cfe55cc03ea1f443f1562beec8df51c75e14a9fcf9a7234a13f198e7969
  7  : 65caf8dd1e0ea7a6347b635d2b379c93b9a1351edc2afc3ecda700e534eb3068
  8  : f440af381b644231e7322babfd393808e8ebb3a692af57c0b3a5d162a6e2c118
  9  : 54c08c8ba4706273f53f90085592f7b2e4eaafb8d433295b66b78d9754145cfc
  10 : 0000000000000000000000000000000000000000000000000000000000000000
  11 : 0000000000000000000000000000000000000000000000000000000000000000
  12 : 0000000000000000000000000000000000000000000000000000000000000000
  13 : 0000000000000000000000000000000000000000000000000000000000000000
  14 : 0000000000000000000000000000000000000000000000000000000000000000
  15 : 0000000000000000000000000000000000000000000000000000000000000000
  16 : 0000000000000000000000000000000000000000000000000000000000000000
  17 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  18 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  19 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  20 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  21 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  22 : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  23 : 0000000000000000000000000000000000000000000000000000000000000000
```

## Service Application Best Practices

### High-Density Many-Core Computing

#### Overview

As the number of cores in a single server continues to increase, the universal scalability law (USL) shows that serial and synchronization overheads prevent linear performance gains. This limitation remains a key challenge in the industry. In practical many-core containerized environments, the serial access and synchronization overheads stem from contention over shared hardware and software resources.

1. Software shared resources: Contention over shared management data structures (such as inode, syslog, and lock).
2. Hardware shared resources: Contention over shared hardware components including memory, caches, buses, and hardware devices.

Resource isolation is a practical method to reduce hardware and software interference on many-core servers. However, full virtualization can introduce overhead. Therefore, lightweight virtualization technologies are used to reduce the impact of virtualization overhead on container services.

#### Lightweight Virtualization Practices

In high-density many-core scenarios, the Kunpeng-V key technologies provide a lightweight and low-overhead virtualization isolation solution. This is achieved through technologies such as transparent transmission of hardware topology, interrupt passthrough, NUMA affinity, vCPU core binding, SR-IOV passthrough, NUMA exposure for passthrough devices, HugePage memory, and memory bandwidth monitoring. By improving the isolation between VMs, the Redis container deployment density can be increased by 100%. For details about the operations of the preceding key technologies, see [Performance Best Practices](#performance-best-practices).
