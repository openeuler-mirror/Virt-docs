# 管理虚拟机

## 概述

StratoVirt可以查询虚拟机信息并对虚拟机的资源和生命周期进行管理。由于StratoVirt使用QMP管理虚拟机，所以查询虚拟机信息，也需要先连接到虚拟机。

## 查询虚拟机信息

### 简介

StratoVirt可以查询虚拟机状态、vCPU拓扑信息、vCPU上线情况等。

### 查询状态

使用query-status命令查询虚拟机的运行状态。

- 用法：

  **{ "execute": "query-status" }**

- 示例：

```shell
<- { "execute": "query-status" }
-> { "return": { "running": true,"singlestep": false,"status": "running" } 
```

### 查询拓扑

使用query-cpus命令查询所有CPU的拓扑结构。

- 用法：

**{ "execute": "query-cpus" }**

- 示例：

```shell
<- { "execute": "query-cpus" }
-> {"return":[{"CPU":0,"arch":"x86","current":true,"halted":false,"props":{"core-id":0,"socket-id":0,"thread-id":0},"qom_path":"/machine/unattached/device[0]","thread_id":8439},{"CPU":1,"arch":"x86","current":true,"halted":false,"props":{"core-id":0,"socket-id":1,"thread-id":0},"qom_path":"/machine/unattached/device[1]","thread_id":8440}]}
```

### 查询vCPU上线情况

使用query-hotpluggable-cpus命令查询所有vCPU的online/offline情况。

- 用法：

**{ "execute": "query-hotpluggable-cpus" }**

- 示例：

```shell
<- { "execute": "query-hotpluggable-cpus" }
-> {"return":[{"props":{"core-id":0,"socket-id":0,"thread-id":0},"qom-path":"/machine/unattached/device[0]","type":"host-x86-cpu","vcpus-count":1},{"props":{"core-id":0,"socket-id":1,"thread-id":0},"qom-path":"/machine/unattached/device[1]","type":"host-x86-cpu","vcpus-count":1}]}
```

其中，online的vCPU具有`qom-path`项，offline的vCPU则没有。

## 管理虚拟机生命周期

### 简介

StratoVirt可以对虚拟机进行启动、暂停、恢复、退出等生命周期进行管理。

### 创建并启动虚拟机

通过命令行参数指定虚拟机配置，创建并启动虚拟机。

- 使用命令行参数给出虚拟机配置，创建并启动虚拟机的命令如下：

```shell
$ /path/to/stratovirt -[参数1] [参数选项] -[参数2] [参数选项] ...
```

> [!NOTE]说明
>
> 轻量虚拟启动后，内部会有eth0和eth1两张网卡。这两张网卡预留用于网卡热插拔。热插的第一张网卡是eth0，热插的第二张网卡是eth1，目前只支持热插两张virtio-net网卡。

### 连接虚拟机

StratoVirt当前采用QMP管理虚拟机，暂停、恢复、退出虚拟机等操作需要通过QMP连接到虚拟机进行管理。

在主机上打开新的命令行窗口B，并使用root权限进行api-channel连接，参考命令如下：

```shell
# ncat -U /path/to/socket
```

连接建立后，会收到来自StratoVirt的问候消息，如下所示：

```shell
{"QMP":{"version":{"qemu":{"micro":1,"minor":0,"major":4},"package":""},"capabilities":[]}}
```

现在，可以在窗口B中输入QMP命令来管理虚拟机。

> [!NOTE]说明
>
> QMP提供了stop、cont、quit和query-status等来管理和查询虚拟机状态。
>
> 管理虚拟机的QMP命令均在窗口B中进行输入。符号：`<-`表示命令输入，`->`表示QMP结果返回。

### 暂停虚拟机

QMP提供了stop命令用于暂停虚拟机，即暂停虚拟机所有的vCPU。命令格式如下：

**{"execute":"stop"}**

**示例：**

使用stop暂停该虚拟机的命令和回显如下：

```shell
<- {"execute":"stop"}
-> {"event":"STOP","data":{},"timestamp":{"seconds":1583908726,"microseconds":162739}}
-> {"return":{}}
```

### 恢复虚拟机

QMP提供了cont命令用于恢复处于暂停状态suspend的虚拟机，即恢复虚拟机所有vCPU的运行。命令格式如下：

**{"execute":"cont"}**

**示例：**

使用cont恢复该虚拟机的命令和回显如下：

```shell
<- {"execute":"cont"}
-> {"event":"RESUME","data":{},"timestamp":{"seconds":1583908853,"microseconds":411394}}
-> {"return":{}}
```

### 退出虚拟机

QMP提供了quit命令用于退出虚拟机，即退出StratoVirt进程。命令格式如下：

**{"execute":"quit"}**

**示例：**

```shell
<- {"execute":"quit"}
-> {"return":{}}
-> {"event":"SHUTDOWN","data":{"guest":false,"reason":"host-qmp-quit"},"timestamp":{"ds":1590563776,"microseconds":519808}}
```

## 管理虚拟机资源

### 热插拔磁盘

StratoVirt支持在虚拟机运行过程中调整磁盘数量，即在不中断业务前提下，增加或删除虚拟机磁盘。

**注意事项**

- 对于标准机型，需要虚拟机内核开启 CONFIG_HOTPLUG_PCI_PCIE=y 配置。

- 对于标准机型，目前支持热插拔设备到 Root Port 设备，Root Port 设备需要在虚拟机启动前配置。

- 不建议在虚拟机启动、关闭、内部高压力等状态下进行设备热插拔，可能会因为虚拟机内驱动没有及时响应导致虚拟机出现异常。

#### 热插磁盘

**用法：**

轻量机型：

```shell
{"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
{"execute": "device_add", "arguments": {"id": "drive-0", "driver": "virtio-blk-mmio", "addr": "0x1"}}
```

标准机型：

```shell
{"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
{"execute":"device_add", "arguments":{"id":"drive-0", "driver":"virtio-blk-pci", "drive": "drive-0", "addr":"0x0", "bus": "pcie.1"}}
```

**参数**

- 对于轻量机型，blockdev-add 中的 node-name 要和 device_add 中的 id 一致，如上都是 drive-0。

- 对于标准机型 drive 参数需要和 blockdev-add 中的 node-name 一致。

- /path/to/block 是被热插磁盘的镜像路径，不能是启动 rootfs 的磁盘镜像。

- 对于轻量机型，addr 参数从 0x0 开始与虚拟机的 vda 映射，0x1 与 vdb 映射，以此类推。为了兼容 QMP 协议，"addr" 也可以用 "lun" 代替，但是 lun=0 与客户机的 vdb 映射。对于标准机型，目前 addr 参数需要指定为 0x0。

- 对于标准机型，bus 为设备要挂载的总线名称，目前只支持热插到 Root Port 设备，需要和 Root Port 的 id 保持一致。

- 对于轻量机型，StratoVirt 支持的最大 virtio-blk 磁盘数量是6个，热插磁盘时请注意规格约束。对于标准机型，热插磁盘的数量取决于 Root Port 设备的数量。

**示例**

轻量机型：

```shell
<- {"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
-> {"return": {}}
<- {"execute": "device_add", "arguments": {"id": "drive-0", "driver": "virtio-blk-mmio", "addr": "0x1"}}
-> {"return": {}}
```

标准机型：

```shell
<- {"execute": "blockdev-add", "arguments": {"node-name": "drive-0", "file": {"driver": "file", "filename": "/path/to/block"}, "cache": {"direct": true}, "read-only": false}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"drive-0", "driver":"virtio-blk-pci", "drive": "drive-0", "addr":"0x0", "bus": "pcie.1"}}
-> {"return": {}}
```

#### 热拔磁盘

**用法：**

轻量机型：

```shell
{"execute": "device_del", "arguments": {"id":"drive-0"}}
```

标准机型：

```shell
{"execute": "device_del", "arguments": {"id":"drive-0"}}
{"execute": "blockdev-del", "arguments": {"node-name": "drive-0"}}
```

**参数：**

- id 为热拔磁盘的 ID 号。
- node-name 为磁盘后端名称。

**示例**

轻量机型：

```shell
<- {"execute": "device_del", "arguments": {"id": "drive-0"}}
-> {"event":"DEVICE_DELETED","data":{"device":"drive-0","path":"drive-0"},"timestamp":{"seconds":1598513162,"microseconds":367129}}
-> {"return": {}}
```

标准机型：

```shell
<- {"execute": "device_del", "arguments": {"id":"drive-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"drive-0","path":"drive-0"},"timestamp":{"seconds":1598513162,"microseconds":367129}}
<- {"execute": "blockdev-del", "arguments": {"node-name": "drive-0"}}
-> {"return": {}}
```

当收到 DEVICE_DELETED 事件时，表示设备在 StratoVirt 侧被移除。

### 热插拔网卡

StratoVirt支持在虚拟机运行过程中调整网卡数量，即在不中断业务前提下，给虚拟机增加或删除网卡。

**注意事项**

- 对于标准机型，需要虚拟机内核开启 CONFIG_HOTPLUG_PCI_PCIE=y 配置。

- 对于标准机型，目前支持热插拔设备到 Root Port 设备，Root Port 设备需要在虚拟机启动前配置。

- 不建议在虚拟机启动、关闭、内部高压力等状态下进行设备热插拔，可能会因为虚拟机内驱动没有及时响应导致虚拟机出现异常。

#### 热插网卡

**准备工作（需要使用root权限）**

1. 创建并启用Linux网桥，例如网桥名为 qbr0 的参考命令如下：

    ```shell
    # brctl addbr qbr0
    # ifconfig qbr0 up
    ```

2. 创建并启用 tap 设备，例如设备名为 tap0 的参考命令如下：

    ```shell
    # ip tuntap add tap0 mode tap
    # ifconfig tap0 up
    ```

3. 添加 tap 设备到网桥：

    ```shell
    # brctl addif qbr0 tap0
    ```

**用法**

轻量机型：

```shell
{"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
{"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-mmio", "addr":"0x0"}}
```

标准机型：

```shell
{"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
{"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-pci", "addr":"0x0", "netdev": "net-0", "bus": "pcie.1"}}
```

**参数**

- 对于轻量机型，netdev_add 中的 id 应该和 device_add 中的 id 一致，ifname 是后端的 tap 设备名称。

- 对于标准机型，netdev 参数需要和 netdev_add 中的 id 一致。

- 对于轻量机型，addr 参数从 0x0 开始与虚拟机的 eth0 映射，0x1 和虚拟机的 eth1 映射。对于标准机型，目前 addr 参数需要指定为 0x0。

- 对于标准机型，bus 为设备要挂载的总线名称，目前只支持热插到 Root Port 设备，需要和 Root Port 的 id 保持一致。

- 对于轻量机型，由于 StratoVirt 支持的最大 virtio-net 数量为2个，热插网卡时请注意规格约束。对于标准机型，热插磁盘的数量取决于 Root Port 设备的数量。

**示例**

轻量机型:

```shell
<- {"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-mmio", "addr":"0x0"}} 
-> {"return": {}}
```

其中，addr:0x0,对应虚拟机内部的eth0。

标准机型：

```shell
<- {"execute":"netdev_add", "arguments":{"id":"net-0", "ifname":"tap0"}}
-> {"return": {}}
<- {"execute":"device_add", "arguments":{"id":"net-0", "driver":"virtio-net-pci", "addr":"0x0", "netdev": "net-0", "bus": "pcie.1"}}
-> {"return": {}}
```

#### 热拔网卡

**用法**

轻量机型：

```shell
{"execute": "device_del", "arguments": {"id": "net-0"}}
```

标准机型：

```shell
{"execute": "device_del", "arguments": {"id":"net-0"}}
{"execute": "netdev_del", "arguments": {"id": "net-0"}}
```

**参数**

- id：网卡的ID号，例如 net-0。

- netdev_del 中的 id 是网卡后端的名称。

**示例**

轻量机型：

```shell
<- {"execute": "device_del", "arguments": {"id": "net-0"}}
-> {"event":"DEVICE_DELETED","data":{"device":"net-0","path":"net-0"},"timestamp":{"seconds":1598513339,"microseconds":97310}}
-> {"return": {}}
```

标准机型：

```shell
<- {"execute": "device_del", "arguments": {"id":"net-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"net-0","path":"net-0"},"timestamp":{"seconds":1598513339,"microseconds":97310}}
<- {"execute": "netdev_del", "arguments": {"id": "net-0"}}
-> {"return": {}}
```

当收到 DEVICE_DELETED 事件时，表示设备在 StratoVirt 侧被移除。

### 热插拔直通设备

StratoVirt 标准机型支持在虚拟机运行过程中调整直通设备数量，即在不中断业务前提下，给虚拟机增加或删除直通设备。

**注意事项**

- 需要虚拟机内核开启 CONFIG_HOTPLUG_PCI_PCIE=y 配置。

- 目前支持热插拔设备到 Root Port 设备，Root Port 设备需要在虚拟机启动前配置。

- 不建议在虚拟机启动、关闭、内部高压力等状态下进行设备热插拔，可能会因为虚拟机内驱动没有及时响应导致虚拟机出现异常。

#### 热插直通设备

**用法**

```shell
{"execute":"device_add", "arguments":{"id":"vfio-0", "driver":"vfio-pci", "bus": "pcie.1", "addr":"0x0", "host": "0000:1a:00.3"}}
```

**参数**

- id 为热插设备的 ID 号。

- bus 为设备要挂载的总线名称。

- addr 为设备要挂载的 slot 和 function 号，目前 addr 参数需要指定为 0x0。

- host 为直通设备在主机上的 domain 号, bus 号, slot 号和 function 号。

**示例**

```shell
<- {"execute":"device_add", "arguments":{"id":"vfio-0", "driver":"vfio-pci", "bus": "pcie.1", "addr":"0x0", "host": "0000:1a:00.3"}}
-> {"return": {}}
```

#### 热拔直通设备

**用法**

```shell
{"execute": "device_del", "arguments": {"id": "vfio-0"}}
```

**参数**

- id 为热拔设备的 ID 号。在热插设备时指定。

**示例**

```shell
<- {"execute": "device_del", "arguments": {"id": "vfio-0"}}
-> {"return": {}}
-> {"event":"DEVICE_DELETED","data":{"device":"vfio-0","path":"vfio-0"},"timestamp":{"seconds":1614310541,"microseconds":554250}}
```

当收到 DEVICE_DELETED 事件时，表示设备在 StratoVirt 侧被移除。

## Ballon设备使用

使用balloon设备可以从虚拟机回收空闲的内存。Balloon通过qmp命令来调用。qmp命令使用如下：

**用法：**

```shell
{"execute": "balloon", "arguments": {"value": 2147483648‬}}
```

**参数：**

- value： 想要设置的guest内存大小值，单位为字节。如果该值大于虚拟机启动时配置的内存值，则以启动时配置的内存值为准。

**示例：**

启动时配置的内存大小为4GiB，在虚拟机内部通过free命令查询虚拟机空闲内存大于2GiB，那么可以通过qmp命令设置guest内存大小为2147483648字节。

```shell
<- {"execute": "balloon", "arguments": {"value": 2147483648‬}}
-> {"return": {}}
```

查询虚拟机的当前实际内存：

```shell
<- {"execute": "query-balloon"}
-> {"return":{"actual":2147483648}}
```

## 虚拟机内存快照

### 简介

虚拟机内存快照是指将虚拟机的设备状态和内存信息保存在快照文件中。当虚拟机系统损坏时，可以使用内存快照将虚拟机恢复到快照对应时间点，从而提升系统的可靠性。

StratoVirt 支持对处于暂停状态（suspend）的虚拟机制作快照，并且支持虚拟机以快照文件为虚拟机模板批量创建新的虚拟机。只要制作快照的时间点在虚拟机启动完成并进入用户态之后，快速启动就能够跳过内核启动阶段和用户态服务初始化阶段，在毫秒级完成虚拟机启动。

### 互斥特性

虚拟机配置了如下设备或使用了如下特性时，不能制作和使用内存快照：

- vhost-net 设备
- vfio 直通设备
- balloon 设备
- 大页内存
- mem-shared 特性
- 配置了内存后端文件 mem-path

### 制作快照

针对 StratoVirt 虚拟机，可以参考如下步骤制作存储快照：

1. 创建并启动虚拟机。

2. 在 Host 上执行 QMP 命令暂停虚拟机：

   ```shell
   <- {"execute":"stop"}
   -> {"event":"STOP","data":{},"timestamp":{"seconds":1583908726,"microseconds":162739}}
   -> {"return":{}}

   ```

3. 确认虚拟机处于暂停状态：

   ```shell
   <- {"execute":"query-status"}
   -> {"return":{"running":true,"singlestep":false,"status":"paused"}}

   ```

4. 执行如下 QMP 命令，在任一指定的绝对路径下创建虚拟机快照，例如 /path/to/template 路径，参考命令如下：

   ```shell
   <- {"execute":"migrate", "arguments":{"uri":"file:/path/to/template"}}
   -> {"return":{}}

   ```

5. 确认快照是否创建成功。

   ```shell
   <- {"execute":"query-migrate"}

   ```

   如果回显 {"return":{"status":"completed"}} ，说明快照创建成功。

   快照创建成功，会在指定路径 /path/to/template 生成 memory 和 state 两个目录。`state`文件包含虚拟机设备状态的信息，`memory`文件包含虚拟机内存的数据信息，memory 文件大小接近配置的虚拟机内存。

### 查询快照状态

当前在整个快照过程中，存在5种状态：

- `None`: 快照资源没有准备完成
- `Setup`: 快照资源准备完成，可以进行快照
- `Active`: 处于制作快照状态中
- `Completed`: 快照制作成功
- `Failed`: 快照制作失败

可以通过在 Host 执行`query-migrate`qmp 命令查询当前快照的状态，如当虚拟机快照制作成功时查询：

```shell
<- {"execute":"query-migrate"}
-> {"return":{"status":"completed"}}
```

### 恢复虚拟机

#### 注意事项

- 快照以及从快照启动特性支持的机型包括：
    - microvm
    - q35（x86_64）
    - virt（aarch64平台）
- 在使用快照恢复时，配置的设备必须与制作快照时保持一致
- 当使用 microvm 机型，并且在快照前使用了磁盘/网卡的热插特性，在恢复时需要将热插的磁盘/网卡配置进启动命令行

#### 从快照文件中恢复虚拟机

**命令格式**

```shell
stratovirt -incoming URI

```

**参数说明**

URI：快照的路径，当前版本只支持 `file` 类型，后加上快照文件的绝对路径

**示例**

假设制作快照所使用的虚拟机是通过以下命令创建的：

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

那么，使用快照恢复虚拟机的参考命令如下（此处假设快照存放的路径为 /path/to/template ）:

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

## 虚拟机热迁移

### 简介

StratoVirt 提供了虚拟机热迁移能力，也就是在虚机业务不中断的情况下，将虚拟机从一台服务器迁移到另一台服务器。 

下列情形，可以使用虚拟机热迁移： 

- 当服务器负载过重时，可以使用虚拟机热迁移技术，将虚拟机迁移到另一台物理服务器上，达到负载均衡的目的。
- 如果需要维护服务器，该服务器上的虚拟机可以在不中断业务的情形下，迁移到另一台物理服务器上。
- 服务器出现故障，需要更换硬件或者调整组网时，为了避免虚拟机业务中断，可以将运行的虚拟机迁移到另一台物理机上。

### 热迁移操作

此处介绍热迁移虚拟机的操作方法，供用户参考。 

**准备热迁移**

1.使用 `root` 帐号，登录源端虚拟机所在的主机，执行如下命令（命令行参数，请根据实际情况修改），启动源端虚拟机。

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

2.使用 `root` 帐号，登录目的端虚拟机所在的主机，执行如下命令（命令行参数需要和启动源端虚拟机保持一致），启动目的端虚拟机。

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

> [!NOTE]说明
>
> - 目的端虚拟机的启动命令行参数需要与源端虚拟机命令行保持一致。
> - 如果需要将热迁移数据传输模式从 `TCP` 网络协议改为 `UNIX socket` 通信协议， 
    只需要将目的端虚拟机的命令行 `-incoming tcp:192.168.0.1:4446`，改为 `-incoming unix:/tmp/stratovirt-migrate.socket`。但 `UNIX socket` 协议只支持单物理主机的不同虚拟机之间热迁移。

**开始热迁移**

在源端虚拟机所在的主机，执行如下命令，启动虚拟机热迁移任务。

```shell
$ ncat -U path/to/socket1
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"migrate", "arguments":{"uri":"tcp:192.168.0.1:4446"}}
-> {"return":{}}
```

> [!NOTE]说明
>
> 如果热迁移传输协议为 `UNIX socket` 通信协议，只需要将 QMP 命令中的 `"uri":"tcp:192.168.0.1:4446"`，改为 `"uri":"unix:/tmp/stratovirt-migrate.socket"`。

**结束热迁移**

当执行上述迁移 `QMP` 命令后，虚拟机热迁移任务就开始执行。如果没有热迁移错误日志，则源端的虚拟机就迁移到了目的端，源端虚拟机会自动销毁。

### 取消热迁移

在热迁移过程中，可能出现迁移时间较长，或目的端虚拟机所在的主机负载发生变化，需要调整迁移策略。StratoVirt 提供了取消热迁移操作的特性。

取消热迁移的操作如下：
登录源端虚拟机所在的主机，执行如下 `QMP` 命令：

```shell
$ ncat -U path/to/socket1
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"migrate_cancel"}
-> {"return":{}}
```

如果目的端虚拟机退出热迁移任务，并在日志提示取消热迁移，表示热迁移任务取消成功。

### 查询热迁移状态

热迁移存在如下几种状态：

- `None`: 热迁移 vCPU，内存，设备等资源没有准备完成
- `Setup`: 热迁移资源准备完成，可以进行热迁移
- `Active`: 处于制作热迁移过程中
- `Completed`: 热迁移完成
- `Failed`: 热迁移失败

以下 `QMP` 命令表示查询当前热迁移处于完成状态：

```shell
$ ncat -U path/to/socket
-> {"QMP":{"version":{"StratoVirt":{"micro":1,"minor":0,"major":0},"package":""},"capabilities":[]}}
<- {"execute":"query-migrate"}
-> {"return":{"status":"completed"}}
```

### 约束与限制

StratoVirt 只支持标准虚机主板热迁移：

- q35 (x86_64平台)
- virt (aarch64平台)

以下设备和特性不支持热迁移：

- vhost-net 设备
- vhost-user-net 设备
- virtio balloon 设备
- vfio 设备
- 共享后端存储
- 共享内存，后端内存特性

以下启动源端和目的端虚拟机命令行参数必须保持一致：

- virtio-net: MAC 地址
- device: BDF 号
- smp
- m
