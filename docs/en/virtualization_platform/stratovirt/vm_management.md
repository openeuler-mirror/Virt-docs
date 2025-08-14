# VM Management

## Overview

StratoVirt allows you to query VM information and manage VM resources and lifecycle with QMP. To query the information about a VM, connect to the VM first.

## Querying VM Information

### Introduction

StratoVirt can be used to query the VM status, vCPU topology, and vCPU online status.

### Querying VM Status

Run the **query-status** command to query the running status of a VM.

- Usage:

  **{ "execute": "query-status" }**

- Example:

```shell
<- { "execute": "query-status" }
-> { "return": { "running": true,"singlestep": false,"status": "running" } 
```

### Querying Topology Information

Run the **query-cpus** command to query the topologies of all CPUs.

- Usage:

   **{ "execute": "query-cpus" }**

- Example:

```shell
<- { "execute": "query-cpus" }
-> {"return":[{"CPU":0,"arch":"x86","current":true,"halted":false,"props":{"core-id":0,"socket-id":0,"thread-id":0},"qom_path":"/machine/unattached/device[0]","thread_id":8439},{"CPU":1,"arch":"x86","current":true,"halted":false,"props":{"core-id":0,"socket-id":1,"thread-id":0},"qom_path":"/machine/unattached/device[1]","thread_id":8440}]}
```

### Querying vCPU Online Status

Run the **query-hotpluggable-cpus** command to query the online/offline statuses of all vCPUs.

- Usage:

**{ "execute": "query-hotpluggable-cpus" }**

- Example:

```shell
<- { "execute": "query-hotpluggable-cpus" }
-> {"return":[{"props":{"core-id":0,"socket-id":0,"thread-id":0},"qom-path":"/machine/unattached/device[0]","type":"host-x86-cpu","vcpus-count":1},{"props":{"core-id":0,"socket-id":1,"thread-id":0},"qom-path":"/machine/unattached/device[1]","type":"host-x86-cpu","vcpus-count":1}]}
```

Online vCPUs have the `qom-path` item, while offline vCPUs do not.

## Managing VM Lifecycle

### Introduction

StratoVirt can manage the lifecycle of a VM, including starting, stopping, resuming, and exiting the VM.

### Creating and Starting a VM

Use the command line parameters to specify the VM configuration, and create and start a VM.

- When using the command line parameters to specify the VM configuration, run the following command to create and start the VM:

```shell
$/path/to/stratovirt - *[Parameter 1] [Parameter option] - [Parameter 2] [Parameter option]*...
```

> [!NOTE]NOTE
>
> After the lightweight VM is started, there are two NICs: eth0 and eth1. The two NICs are reserved for hot plugging: eth0 first and then eth1. Currently, only two virtio-net NICs can be hot plugged.

### Connecting to a VM

StratoVirt uses QMP to manage VMs. To stop, resume, or exit a VM, connect it the StratoVirt through QMP first.

Open a new CLI (CLI B) on the host and run the following command to connect to the api-channel as the **root** user:

```shell
# ncat -U /path/to/socket
```

After the connection is set up, you will receive a greeting message from StratoVirt, as shown in the following:

```shell
{"QMP":{"version":{"qemu":{"micro":1,"minor":0,"major":4},"package":""},"capabilities":[]}}
```

You can now manage the VM by entering the QMP commands in CLI B.

> [!NOTE]NOTE
>
> QMP provides **stop**, **cont**, **quit**, and **query-status** commands to manage and query VM statuses.
>
> All QMP commands for managing VMs are entered in CLI B. `<-` indicates the command input, and `->` indicates the QMP returned result.

### Stopping a VM

QMP provides the **stop** command to stop a VM, that is, to stop all vCPUs of the VM. The command syntax is as follows:

**{"execute":"stop"}**

**Example:**

The **stop** command and the command output are as follows:

```shell
<- {"execute":"stop"}
-> {"event":"STOP","data":{},"timestamp":{"seconds":1583908726,"microseconds":162739}}
-> {"return":{}}
```

### Resuming a VM

QMP provides the **cont** command to resume a stopped VM, that is, to resume all vCPUs of the VM. The command syntax is as follows:

**{"execute":"cont"}**

**Example:**

The **cont** command and the command output are as follows:

```shell
<- {"execute":"cont"}
-> {"event":"RESUME","data":{},"timestamp":{"seconds":1583908853,"microseconds":411394}}
-> {"return":{}}
```

### Exiting a VM

QMP provides the **quit** command to exit a VM, that is, to exit the StratoVirt process. The command syntax is as follows:

**{"execute":"quit"}**

**Example:**

```shell
<- {"execute":"quit"}
-> {"return":{}}
-> {"event":"SHUTDOWN","data":{"guest":false,"reason":"host-qmp-quit"},"timestamp":{"ds":1590563776,"microseconds":519808}}
```

## Managing VM Resources

### Hot-Pluggable Disks

StratoVirt allows you to adjust the number of disks when a VM is running. That is, you can add or delete VM disks without interrupting services.

**Note**

- For a standard VM, the **CONFIG_HOTPLUG_PCI_PCIE=y** configuration must be enabled for the VM kernel.

- For a standard VM, devices can be hot added to the root port. The root port device must be configured before the VM is started.

- You are not advised to hot swap a device when the VM is being started, stopped, or under high internal pressure. Otherwise, the VM may become abnormal because the drivers on the VM cannot respond in a timely manner.

#### Hot Adding Disks

**Usage:**

Lightweight VM:

```shell
{"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
{"execute": "device_add", "arguments": {"id": "drive-0", "driver": "virtio-blk-mmio", "addr": "0x1"}}
```

Standard VM:

```shell
{"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
{"execute":"device_add", "arguments":{"id":"drive-0", "driver":"virtio-blk-pci", "drive": "drive-0", "addr":"0x0", "bus": "pcie.1"}}
```

**Parameters:**

- For a lightweight VM, the value of **node-name** in **blockdev-add** must be the same as that of **id** in **device_add**. For example, the values of **node-name** and **id** are both **drive-0** as shown above.

- For a standard VM, the value of **drive** must be the same as that of **node-name** in **blockdev-add**.

- **/path/to/block** is the image path of the hot added disks. It cannot be the path of the disk image that boots the rootfs.

- For a lightweight VM, the value of **addr**, starting from **0x0**, is mapped to a virtio device on the VM. **0x0** is mapped to **vda**, **0x1** is mapped to **vdb**, and so on. To be compatible with the QMP protocol, **addr** can be replaced by **lun**, but **lun=0** is mapped to the **vdb** of the guest machine. For a standard VM, the value of **addr** must be **0x0**.

- For a standard VM, **bus** indicates the name of the bus to mount the device. Currently, the device can be hot added only to the root port device. The value of **bus** must be the ID of the root port device.

- For a lightweight VM, StratoVirt supports a maximum of six virtio-blk disks. Note this when hot adding disks. For a standard VM, the maximum number of hot added disks depends on the number of root port devices.

**Example:**

Lightweight VM:

```shell
<- {"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
-> {"return": {}}
<- {"execute": "device_add", "arguments": {"id": "drive-0", "driver": "virtio-blk-mmio", "addr": "0x1"}}
-> {"return": {}}
```

Standard VM:

```shell
<- {"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"drive-0", "driver":"virtio-blk-pci", "drive": "drive-0", "addr":"0x0", "bus": "pcie.1"}}
-> {"return": {}}
```

#### Hot Removing Disks

**Usage:**

Lightweight VM:

```shell
{"execute": "device_del", "arguments": {"id":"drive-0"}}
```

Standard VM:

```shell
{"execute": "device_del", "arguments": {"id":"drive-0"}}
{"execute": "blockdev-del", "arguments": {"node-name": "drive-0"}}
```

**Parameters:**

**id** indicates the ID of the disk to be hot removed.

- **node-name** indicates the backend name of the disk.

**Example:**

Lightweight VM:

```shell
<- {"execute": "device_del", "arguments": {"id": "drive-0"}}
-> {"event":"DEVICE_DELETED","data":{"device":"drive-0","path":"drive-0"},"timestamp":{"seconds":1598513162,"microseconds":367129}}
-> {"return": {}}
```

Standard VM:

```shell
<- {"execute": "device_del", "arguments": {"id":"drive-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"drive-0","path":"drive-0"},"timestamp":{"seconds":1598513162,"microseconds":367129}}
<- {"execute": "blockdev-del", "arguments": {"node-name": "drive-0"}}
-> {"return": {}}
```

A **DEVICE_DELETED** event indicates that the device is removed from StratoVirt.

### Hot-Pluggable NICs

StratoVirt allows you to adjust the number of NICs when a VM is running. That is, you can add or delete VM NICs without interrupting services.

**Note**

- For a standard VM, the **CONFIG_HOTPLUG_PCI_PCIE=y** configuration must be enabled for the VM kernel.

- For a standard VM, devices can be hot added to the root port. The root port device must be configured before the VM is started.

- You are not advised to hot swap a device when the VM is being started, stopped, or under high internal pressure. Otherwise, the VM may become abnormal because the drivers on the VM cannot respond in a timely manner.

#### Hot Adding NICs

**Preparations (Requiring the root Permission)**

1. Create and enable a Linux bridge. For example, if the bridge name is **qbr0**, run the following command:

    ```shell
    # brctl addbr qbr0
    # ifconfig qbr0 up
    ```

2. Create and enable a tap device. For example, if the tap device name is **tap0**, run the following command:

    ```shell
    # ip tuntap add tap0 mode tap
    # ifconfig tap0 up
    ```

3. Add the tap device to the bridge.

    ```shell
    # brctl addif qbr0 tap0
    ```

**Usage:**

Lightweight VM:

```shell
{"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
{"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-mmio", "addr":"0x0"}}
```

Standard VM:

```shell
{"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
{"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-pci", "addr":"0x0", "netdev": "net-0", "bus": "pcie.1"}}
```

**Parameters:**

- For a lightweight VM, **id** in **netdev_add** must be the same as that in **device_add**. **ifname** is the name of the backend tap device.

- For a standard VM, the value of **netdev** must be the value of **id** in **netdev_add**.

- For a lightweight VM, the value of **addr**, starting from **0x0**, is mapped to an NIC on the VM. **0x0** is mapped to **eth0**, **0x1** is mapped to **eth1**. For a standard VM, the value of **addr** must be **0x0**.

- For a standard VM, **bus** indicates the name of the bus to mount the device. Currently, the device can be hot added only to the root port device. The value of **bus** must be the ID of the root port device.

- For a lightweight VM, StratoVirt supports a maximum of two virtio-net NICs. Therefore, pay attention to the specification restrictions when hot adding in NICs. For a standard VM, the maximum number of hot added disks depends on the number of root port devices.

**Example:**

Lightweight VM:

```shell
<- {"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-mmio", "addr":"0x0"}} 
-> {"return": {}}
```

**addr:0x0** corresponds to **eth0** in the VM.

Standard VM:

```shell
<- {"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-pci", "addr":"0x0", "netdev": "net-0", "bus": "pcie.1"}}
-> {"return": {}}
```

#### Hot Removing NICs

**Usage:**

Lightweight VM:

```shell
{"execute": "device_del", "arguments": {"id": "net-0"}}
```

Standard VM:

```shell
{"execute": "device_del", "arguments": {"id":"net-0"}}
{"execute": "netdev_del", "arguments": {"id": "net-0"}}
```

**Parameters:**

**id**: NIC ID, for example, **net-0**.

- **id** in **netdev_del** indicates the backend name of the NIC.

**Example:**

Lightweight VM:

```shell
<- {"execute": "device_del", "arguments": {"id": "net-0"}}
-> {"event":"DEVICE_DELETED","data":{"device":"net-0","path":"net-0"},"timestamp":{"seconds":1598513339,"microseconds":97310}}
-> {"return": {}}
```

Standard VM:

```shell
<- {"execute": "device_del", "arguments": {"id":"net-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"net-0","path":"net-0"},"timestamp":{"seconds":1598513339,"microseconds":97310}}
<- {"execute": "netdev_del", "arguments": {"id": "net-0"}}
-> {"return": {}}
```

A **DEVICE_DELETED** event indicates that the device is removed from StratoVirt.

### Hot-swappable Pass-through Devices

You can add or delete the passthrough devices of a StratoVirt standard VM when it is running.

**Note**

- The **CONFIG_HOTPLUG_PCI_PCIE=y** configuration must be enabled for the VM kernel.

- Devices can be hot added to the root port. The root port device must be configured before the VM is started.

- You are not advised to hot swap a device when the VM is being started, stopped, or under high internal pressure. Otherwise, the VM may become abnormal because the drivers on the VM cannot respond in a timely manner.

#### Hot Adding Pass-through Devices

**Usage:**

```shell
{"execute":"device_add", "arguments":{"id":"vfio-0", "driver":"vfio-pci", "bus": "pcie.1", "addr":"0x0", "host": "0000:1a:00.3"}}
```

**Parameters:**

- **id** indicates the ID of the hot added device.

- **bus** indicates the name of the bus to mount the device.

- **addr** indicates the slot and function numbers to mount the device. Currently, **addr** must be set to **0x0**.

- **host** indicates the domain number, bus number, slot number, and function number of the passthrough device on the host machine.

**Example:**

```shell
<- {"execute":"device_add", "arguments":{"id":"vfio-0", "driver":"vfio-pci", "bus": "pcie.1", "addr":"0x0", "host": "0000:1a:00.3"}}
-> {"return": {}}
```

#### Hot Removing Pass-through Devices

**Usage:**

```shell
{"execute": "device_del", "arguments": {"id": "vfio-0"}}
```

**Parameters:**

- **id** indicates the ID of the device to be hot removed, which is specified when the device is hot added.

**Example:**

```shell
<- {"execute": "device_del", "arguments": {"id": "vfio-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"vfio-0","path":"vfio-0"},"timestamp":{"seconds":1614310541,"microseconds":554250}}
```

A **DEVICE_DELETED** event indicates that the device is removed from StratoVirt.

## Using Ballon Devices

The balloon device is used to reclaim idle memory from a VM. It called by running the QMP command.  

**Usage:**

```shell
{"execute": "balloon", "arguments": {"value": 2147483648}}
```

**Parameters:**

- **value**: size of the guest memory to be set. The unit is byte. If the value is greater than the memory value configured during VM startup, the latter is used.

**Example:**

The memory size configured during VM startup is 4 GiB. If the idle memory of the VM queried by running the free command is greater than 2 GiB, you can run the QMP command to set the guest memory size to 2147483648 bytes.

```shell
<- {"execute": "balloon", "arguments": {"value": 2147483648}}
-> {"return": {}}
```

Query the actual memory of the VM:

```shell
<- {"execute": "query-balloon"}
-> {"return":{"actual":2147483648}}
```

## Using VM Memory Snapshots

### Introduction

A VM memory snapshot stores the device status and memory information of a VM in a snapshot file. If the VM is damaged, you can use the snapshot to restore it to the time when the snapshot was created, improving system reliability.

StratoVirt allows you to create snapshots for stopped VMs and create VMs in batches with a snapshot file as the VM template. As long as a snapshot is created after a VM is started and enters the user mode, the quick startup can skip the kernel startup and user-mode service initialization phases and complete the VM startup in milliseconds.

### Mutually Exclusive Features

Memory snapshots cannot be created or used for VMs that are configured with the following devices or use the following features:

- vhost-net device
- VFIO passthrough device
- Balloon device
- Huge page memory feature
- mem-shared feature
- memory backend file **mem-path**

### Creating a Snapshot

For StratoVirt VMs, perform the following steps to create a storage snapshot:

1. Create and start a VM.

2. Run the QMP command on the host to stop the VM.

   ```shell
   <- {"execute":"stop"}
   -> {"event":"STOP","data":{},"timestamp":{"seconds":1583908726,"microseconds":162739}}
   -> {"return":{}}
   ```

3. Confirm that the VM is stopped.

   ```shell
   <- {"execute":"query-status"}
   -> {"return":{"running":true,"singlestep":false,"status":"paused"}}
   ```

4. Run the following QMP command to create a VM snapshot in a specified absolute path, for example, **/path/to/template**:

   ```shell
   <- {"execute":"migrate", "arguments":{"uri":"file:/path/to/template"}}
   -> {"return":{}}
   ```

5. Check whether the snapshot is successfully created.

   ```shell
   <- {"execute":"query-migrate"}
   ```

   If "{"return":{"status":"completed"}}" is displayed, the snapshot is successfully created.

   If the snapshot is created successfully, the `memory` and `state` directories are generated in the specified path **/path/to/template**. The `state` file contains VM device status information, and the `memory` file contains VM memory data. The size of the `memory` file is close to the configured VM memory size.

### Querying Snapshot Status

There are five statuses in the snapshot process.

- `None`: The snapshot resource is not ready.
- `Setup`: The snapshot resource is ready. You can create a snapshot.
- `Active`: The snapshot is being created.
- `Completed`: The snapshot is created successfully.
- `Failed`: The snapshot fails to be created.

You can run the `query-migrate` QMP command on the host to query the status of the current snapshot. For example, if the VM snapshot is created successfully, the following output is displayed:

```shell
<- {"execute":"query-migrate"}
-> {"return":{"status":"completed"}}
```

### Restoring a VM

#### Precautions

- The following models support the snapshot and boot from snapshot features:
    - microvm
    - Q35 (x86_64)
    - virt (AArch64)
- When a snapshot is used for restoration, the configured devices must be the same as those used when the snapshot is created.
- If a microVM is used and the disk/NIC hot plugging-in feature is enabled before the snapshot is taken, you need to configure the hot plugged-in disks or NICs in the startup command line during restoration.

#### Restoring a VM from a Snapshot File

**Command Format**

```shell
stratovirt -incoming URI
```

**Parameters**

**URI**: snapshot path. The current version supports only the `file` type, followed by the absolute path of the snapshot file.

**Example**

Assume that the VM used for creating a snapshot is created by running the following command:

```shell
$ stratovirt \
    -machine microvm \
    -kernel /path/to/kernel \
    -smp 1 -m 1024 \
    -append "console=ttyS0 pci=off reboot=k quiet panic=1 root=/dev/vda" \
    -drive file=/path/to/rootfs,id=rootfs,readonly=off,direct=off \
    -device virtio-blk-device,drive=rootfs \
    -qmp unix:/path/to/socket,server,nowait \
    -serial stdio
```

Then, the command for restoring the VM from the snapshot (assume that the snapshot storage path is **/path/to/template**) is as follows:

```shell
$ stratovirt \
    -machine microvm \
    -kernel /path/to/kernel \
    -smp 1 -m 1024 \
    -append "console=ttyS0 pci=off reboot=k quiet panic=1 root=/dev/vda" \
    -drive file=/path/to/rootfs,id=rootfs,readonly=off,direct=off \
    -device virtio-blk-device,drive=rootfs \
    -qmp unix:/path/to/another_socket,server,nowait \
    -serial stdio \
    -incoming file:/path/to/template
```

## VM Live Migration

### Introduction

StratoVirt provides the VM live migration capability, that is, migrating a VM from one server to another without interrupting VM services.

VM live migration can be used in the following scenarios:

- When a server is overloaded, the VM live migration technology can be used to migrate VMs to another physical server for load balancing.
- When a server needs maintenance, VMs on the server can be migrated to another physical server without interrupting services.
- When a server is faulty and hardware needs to be replaced or the networking needs to be adjusted, VMs on the server can be migrated to another physical machine to prevent VM service interruption.

### Live Migration Operations

This section describes how to live migrate a VM.

#### Preparing for Live Migration

1. Log in to the host where the source VM is located as the **root** user and run the following command to start the source VM. Modify the parameters as required:

    ```shell
    ./stratovirt \
        -machine q35 \
        -kernel ./vmlinux.bin \
        -append "console=ttyS0 pci=off reboot=k quiet panic=1 root=/dev/vda" \
        -drive file=path/to/rootfs,id=rootfs,readonly=off,direct=off \
        -device virtio-blk-pci,drive=rootfs,id=rootfs,bus=pcie.0,addr=0 \
        -qmp unix:path/to/socket1,server,nowait \
        -serial stdio \
    ```

2. Log in to the host where the target VM is located as the **root** user and run the following command to start the target VM. The parameters must be consistent with those of the source VM:

    ```shell
    ./stratovirt \
        -machine q35 \
        -kernel ./vmlinux.bin \
        -append "console=ttyS0 pci=off reboot=k quiet panic=1 root=/dev/vda" \
        -drive file=path/to/rootfs,id=rootfs,readonly=off,direct=off \
        -device virtio-blk-pci,drive=rootfs,id=rootfs,bus=pcie.0,addr=0 \
        -qmp unix:path/to/socket2,server,nowait \
        -serial stdio \
        -incoming tcp:192.168.0.1:4446 \
    ```

> [!NOTE]NOTE
>
> - The parameters for starting the target VM must be consistent with those for starting the source VM:
> - To change the data transmission mode for live migration from TCP to the UNIX socket protocol, change the `-incoming tcp:192.168.0.1:4446` parameter for starting the target VM to `-incoming unix:/tmp/stratovirt-migrate.socket`. However, the UNIX socket protocol supports only live migration between different VMs on the same physical host.

#### Starting Live Migration

On the host where the source VM is located, run the following command to start the VM live migration task:

```shell
$ ncat -U path/to/socket1
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"migrate", "arguments":{"uri":"tcp:192.168.0.1:4446"}}
-> {"return":{}}
```

> [!NOTE]NOTE
>
> If the UNIX socket protocol is used for live migration transmission, change `"uri":"tcp:192.168.0.1:4446"` in the command to `"uri":"unix:/tmp/stratovirt-migrate.socket"`.

#### Finishing Live Migration

After the `QMP` command is executed, the VM live migration task starts. If no live migration error log is generated, the source VM is migrated to the target VM, and the source VM is automatically destroyed.

#### Canceling Live Migration

During live migration, the migration may take a long time or the load of the host where the target VM is located changes. In this case, you need to adjust the migration policy. StratoVirt provides the feature of canceling live migration.

To cancel live migration, log in to the host where the source VM is located and run the following `QMP` command:

```shell
$ ncat -U path/to/socket1
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"migrate_cancel"}
-> {"return":{}}
```

If the live migration task on the target VM stops and the log indicates that the live migration is canceled, the live migration task is canceled successfully.

#### Querying the Live Migration Status

A live migration task can be in the following statuses:

- **None**: Resources, such as vCPUs, memory, and devices, are not ready for live migration.
- **Setup**: Live migration resources have been prepared and live migration can be performed.
- **Active**: The VM is being live migrated.
- **Completed**: Live migration is complete.
- **Failed**: Live migration fails.

The following `QMP` command queries completed live migration tasks:

```shell
$ ncat -U path/to/socket
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"query-migrate"}
-> {"return":{"status":"completed"}}
```

### Constraints

StratoVirt supports live migration of the following standard VM boards:

- Q35 (x86_64)
- virt (AArch64)

The following devices and features do not support live migration:

- vhost-net device
- vhost-user-net device
- virtio balloon device
- vfio device
- Shared backend storage
- Shared memory (back-end memory feature)

The following command parameters for starting the source and target VMs must be the same:

- virtio-net: MAC address
- device: BDF number
- smp
- m
