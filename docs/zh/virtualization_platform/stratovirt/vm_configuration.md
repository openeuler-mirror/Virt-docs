# 虚拟机配置

## 概述

使用 StratoVirt 时，可以通过命令行参数指定虚拟机配置，也支持对接 libvirt ，通过 XML 文件配置。本章介绍命令行方式的配置方式。

> [!NOTE]说明
>
> 本文中的 /path/to/socket 为用户自定义路径下的 socket 文件。
>
> 从 openEuler 21.09 版本开始，取消了对 json 文件的支持。

## 规格说明

StratoVirt 支持启动轻量级虚拟机和标准虚拟机。

- 轻量级虚拟机使用轻量级 microVM 主板，以及 mmio 总线。
- 标准虚拟机支持标准启动，在 x86 平台使用 Q35 主板，AArch64 架构下使用 virt 主板以及 PCI 总线。

### 轻量级虚拟机

- 虚拟机 CPU 个数：[1, 254]
- 虚拟机内存大小：[128 MiB, 512 GiB]，默认内存配置256MiB
- 虚拟机磁盘个数（包括热插的磁盘）：[0, 6]
- 虚拟机网卡个数（包括热插的网卡）：[0, 2]
- 虚拟机 console 设备仅支持单路连接
- 主机 CPU 架构为 x86_64 时，最多可以配置 11 个 mmio 设备，但是除了磁盘和网卡，建议最多配置 2 个其他设备;  AArch64 平台，最多可以配置 160 个 mmio 设备，但是除了磁盘和网卡，建议最多配置 12 个其他设备。

### 标准虚拟机

- 虚拟机 CPU 个数：[1, 254]
- 虚拟机内存大小：[128  MiB, 512  GiB]，默认内存配置256MiB
- 虚拟机 console 设备仅支持单路连接
- 只支持 1 个 console 设备
- 最多支持 32 个 PCI 设备
- PCI 设备挂载的 PCI 总线 slot 取值范围： [0, 32)；function 取值范围 [0, 8)

## 最小配置

StratoVirt 能够运行的最小配置为：

- PE 格式或 bzImage 格式（仅 x86_64）的 Linux 内核镜像
- 将 rootfs 镜像设置成 virtio-blk 设备，并添加到内核参数中
- 使用 QMP 控制 StratoVirt
- 如果要使用串口登录，添加一个串口到内核启动命令行，AArch64平台标准机型为ttyAMA0，其他情况为ttyS0.

## 配置介绍

### **命令格式**

使用 cmdline 配置的命令格式如下：

**$ /path/to/stratovirt** *-[参数1] [参数选项] -[参数2] [参数选项] ...*

### **使用说明**

1. 首先，为确保可以创建 QMP 需要的 socket，可以参考如下命令清理环境：

   ```sh
   # rm [参数] [用户自定义socket文件路径]
   ```

2. 然后，运行 cmdline 命令。

   ```sh
   # /path/to/stratovirt -[参数1] [参数选项] -[参数2] [参数选项] ...
   ```

### 基本信息配置

基本配置信息如下表所示：

| 参数             | 参数选项                                                     | 说明                                                         |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -name            | *VMname*                                                     | 配置虚拟机名称（字符长度：1-255字符）                        |
| -kernel          | /path/to/vmlinux.bin                                         | 配置内核镜像                                                 |
| -append          | console=ttyS0 root=/dev/vda reboot=k panic=1 rw              | 配置内核命令行参数，轻量级虚拟机固定配置为console=ttyS0（与架构平台无关）。标准虚拟化X86_64平台默认使用console=ttyS0，AArch64平台默认使用console=ttyAMA0。在配置了virtio-console设备但是没有配置serial串口设备时，需要配置为console=hvc0（与架构平台无关） |
| -initrd          | /path/to/initrd.img                                          | 配置initrd文件                                               |
| -smp             | [cpus=]n[,maxcpus=,sockets=,dies=,clusters=,cores=,threads=] | cpus：配置cpu个数，范围[1, 254]。maxcpus：最大cpu个数，范围[1,254]。sockets：socket的个数，如果不设置它的值依赖于maxcpus；die：die的个数；cluster：cluster的个数；core：core的个数，如果不设置它的值依赖于maxcpus；thread：thread的个数，如果不设置它的值依赖于maxcpus；maxcpus=sockets *dies* clusters *cores* threads |
| -m               | 内存大小MiB、内存大小GiB，默认单位MiB                            | 配置内存大小，范围[128 MiB, 512 GiB]，默认内存配置256MiB         |
| -qmp             | unix:/path/to/socket,server,nowait                           | 配置QMP，运行前须保证socket文件不存在                |
| -D               | /path/to/logfile                                             | 配置日志文件                                                 |
| -pidfile         | /path/to/pidfile                                             | 配置pid文件，必须和-daemonize一起使用。运行前须保证pid文件不存在 |
| -disable-seccomp | NA                                                           | 关闭Seccomp，默认打开                                        |
| -daemonize       | NA                                                           | 开启进程daemon化                                             |

### 虚拟机类型

通过-machine参数来指定启动的虚拟机的类型。

参数说明

- type：启动虚拟机的类型（轻量级虚拟化为“MicroVm”类型，标准虚拟化在x86_64平台为”q35“，在aarch64平台为”virt”）。
- dump-guest-core：进程panic时，是否dump虚拟机内存（可选配置）。
- mem-share：是否与其他进程共享内存（可选配置）。

### 磁盘配置

虚拟机磁盘配置包含以下配置项

- drive_id： 磁盘的id。
- path_on_host： 磁盘的路径。
- serial_num： 磁盘的串号（可选配置）。
- read_only： 是否只读（可选配置）。
- direct： 是否以“O_DIRECT”模式打开（可选配置）。
- iothread： 配置iothread属性（可选配置）。
- throttling.iops-total： 配置磁盘QoS，以限制磁盘的io操作（可选配置）。
- if：driver的类型，block设备为“none”（可选配置，缺省值为“none”）
- bus：设备要挂载的bus。
- addr：设备要挂载的slot和function号。
- multifunction：是否开启pci多功能。（可选配置）
- bootindex：配置启动优先级属性，如果没有设置，默认最低优先级。配置范围从0到255，数字越小，优先级越高。（可选配置，只支持标准机型）

#### 磁盘配置方式

磁盘的配置分为两步：driver的配置和block设备的配置

轻量虚拟机配置格式为：

```Conf
-drive id=drive_id,file=path_on_host[,readonly=off][,direct=off][,throttling.iops-total=200][,if=none]
-device virtio-blk-device,drive=drive_id[,iothread=iothread1][,serial=serial_num]
```

标准虚拟机配置格式为：

```Conf
-drive id=drive_id,file=path_on_host[,readonly=off][,direct=off][,throttling.iops-total=200][,if=none]
-device virtio-blk-pci,drive=drive_id,bus=pcie.0,addr=0x3.0x0[,iothread=iothread1,][serial=serial_num][,multifunction=on][,bootindex=1]
```

下面对throttling.iops-total和iothread两个配置项进行详细说明：

#### 磁盘QoS

##### 简介

QoS（Quality of Service）是服务质量的意思。在云场景中，单主机内会启动多台虚拟机，当某台虚拟机对磁盘访问压力大时，由于同主机的磁盘访问总带宽有限，这会挤占其他虚拟机的访问带宽，从而造成对其他虚拟机IO影响。为了降低影响，可以为虚拟机配置QoS属性，限制它们对磁盘访问的速率，从而降低对彼此的影响。

##### 注意事项

- 当前QoS支持配置磁盘的iops。
- iops的设定范围是[0, 1000000]，0为不限速；实际iops不会超过设定值，并且不会超过后端磁盘实际性能的上限。
- 只能限制平均iops，无法限速瞬时的突发流量。

##### 配置方式

用法：

**命令行**

```Conf
-drive xxx,throttling.iops-total=200
```

参数：

- throttling.iops-total：当配置了iops后，本磁盘在虚拟机内部的IO下发速度，不会超过此配置值。
- xxx：表示磁盘的其他设置。

#### iothread

iothread配置细节见[iothread配置](#iothread)

### 网卡配置

虚拟机网卡的配置包含以下配置项：

- id：唯一的设备 id。
- tap：指定 tap 设备。
- ifname：host 上的 tap 设备名。
- mac：设置虚拟机 mac 地址（可选配置）。
- iothread：配置磁盘的 iothread 属性（可选配置）。网卡 iothread 配置详见 [iothread配置](#iothread) 。

#### 配置方式

> [!NOTE]说明
>
> 使用网络前请先使用如下命令配置好 host 网桥和 tap 设备。
>
> ```sh
> # brctl addbr qbr0
> # ip tuntap add tap0 mode tap
> # brctl addif qbr0 tap0
> # ifconfig qbr0 up; ifconfig tap0 up
> # ifconfig qbr0 192.168.0.1
> ```

1. 配置 virtio-net（本文中 [] 表示可选参数）

   轻量级虚拟机：

   ```Conf
   -netdev tap,id=netdevid,ifname=host_dev_name[,vhostfd=2]
   -device virtio-net-device,netdev=netdevid,id=netid[,iothread=iothread1,mac=12:34:56:78:9A:BC]
   ```

   标准虚拟机：

   ```Conf
   -netdev tap,id=netdevid,ifname=host_dev_name[,vhostfd=2]
   -device virtio-net-pci,netdev=netdevid,id=netid,bus=pcie.0,addr=0x2.0x0[,multifunction=on,iothread=iothread1,mac=12:34:56:78:9A:BC]
   ```

2. 配置  vhost-net

   轻量级虚拟机：

   ```Conf
   -netdev tap,id=netdevid,ifname=host_dev_name,vhost=on[,vhostfd=2]
   -device virtio-net-device,netdev=netdevid,id=netid[,iothread=iothread1,mac=12:34:56:78:9A:BC]
   ```

   标准虚拟机：

   ```Conf
   -netdev tap,id=netdevid,ifname=host_dev_name,vhost=on[,vhostfd=2]
   -device virtio-net-pci,netdev=netdevid,id=netid,bus=pcie.0,addr=0x2.0x0[,multifunction=on,iothread=iothread1,mac=12:34:56:78:9A:BC]
   ```

### chardev 配置

将来自 Guest 的 I/O 重定向到宿主机的 chardev。chardev 后端的类型可以是：stdio、pty、socket 和 file。其中 file 仅支持输出时设置。配置项：

- id：唯一的设备 id。
- backend：重定向的类型。
- path：设备重定向文件路径。仅 socket 和 file 类型的设备需要此参数。
- server：将 chardev 作为服务器运行。仅 socket 类型的设备需要此参数。
- nowait：预期状态为断开连接。仅 socket 类型的设备需要此参数。

使用 chardev 时，会创建并使用 console 文件，所以启动 stratovirt 之前，请确保 console 文件不存在。

#### 配置方式

```Conf
-chardev backend,id=chardev_id[,path=path,server,nowait]
```

### 串口配置

串口是虚拟机的设备，用于主机和虚拟机之间传送数据。使用串口时，kernel 命令行中配置 console=ttyS0 ，在 AArch64 平台上标准启动时，配置 console=ttyAMA0 。配置项：

- chardev：重定向的 chardev 设备
- backend、path、server、nowait：这些参数的含义与 chardev 中的相同。

#### 配置方式

```Conf
-serial chardev:chardev_id
```

或者：

```Conf
-chardev backend[,path=path,server,nowait]
```

### console 设备配置

virtio-console 是通用的串口设备，用于主机和虚拟机之间传送数据。当只配 console 并通过 console 进行 I/O 操作时，kernel 启动参数中配置 console=hvc0。console 设备有如下配置项：

- id： 设备的 id。
- path：virtio console 文件路径。
- socket：以 socket 的方式重定向。
- chardev：重定向的 chardev 设备。

#### 配置方式

console 配置分为三步：首先指定 virtio-serial，然后创建字符设备，最后创建 virtconsole 设备。

轻量级虚拟机：

```Conf
-device virtio-serial-device[,id=virtio-serial0]
-chardev socket,path=socket_path,id=virtioconsole1,server,nowait
-device virtconsole,chardev=virtioconsole1,id=console_id
```

标准虚拟机：

```Conf
-device virtio-serial-pci,bus=pcie.0,addr=0x1.0x0[,multifunction=on,id=virtio-serial0]
-chardev socket,path=socket_path,id=virtioconsole1,server,nowait
-device virtconsole,chardev=virtioconsole1,id=console_id
```

### vsock 设备配置

vsock 也是主机和虚拟机之间通信的设备，类似于 console，但具有更好的性能。配置项：

- id： 唯一的设备 id。
- guest_cid： 唯一的 context id 。

#### 配置方式

轻量级虚拟机：

```Conf
-device vhost-vsock-device,id=vsock_id,guest-cid=3
```

标准虚拟机：

```Conf
-device vhost-vsock-pci,id=vsock_id,guest-cid=3,bus=pcie.0,addr=0x1.0x0[,multifunction=on]
```

### 内存大页配置

#### 概述

StratoVirt 支持为虚拟机配置内存大页，相比传统的 4KiB 内存分页模式，大页内存可以有效减少 TLB Miss 次数和缺页中断次数，能够显著提升内存密集型业务性能。

#### 注意事项

- 指定的大页挂载的目录，必须是绝对路径。
- 仅支持在启动时配置。
- 仅支持静态大页。
- 使用大页前， 在Host上需要配置好大页。
- 使用大页特性， 指定虚拟机内存规格必须是**大页页面大小的整数倍**。

#### 互斥特性

- 内存大页和 ballon 特性互斥，同时配置时，balloon 特性无效。

#### 配置方式

##### 配置Host上大页

###### 挂载

将大页文件系统挂载到指定目录上，其中 `/path/to/hugepages`为用户自定义的空目录。

```sh
# mount -t hugetlbfs hugetlbfs /path/to/hugepages
```

###### 设置大页数目

- 设置静态大页数目, `num`为指定的大页数目

  ```sh
  # sysctl vm.nr_hugepages=num
  ```

- 查询大页统计信息

  ```sh
  # cat /proc/meminfo | grep Hugepages
  ```

  如果需要查看其他页面大小的大页统计信息， 可以查看 `/sys/kernel/mm/hugepages/hugepages-*/`目录下相关信息。

> [!NOTE]说明
>
> 请根据大页使用情况，配置StratoVirt内存规格和大页。如果大页资源不足，虚拟机会启动失败。

#### 启动StratoVirt时添加大页配置

- 命令行

  ```Conf
  -mem-path /page/to/hugepages
  ```

  其中 `/page/to/hugepages`为大页文件系统挂载的目录，仅支持绝对路径。

> [!NOTE]说明
>
> **典型配置：**指定StratoVirt命令行中的mem-path项为：**大页文件系统挂载的目录**。 推荐使用典型配置使用StratoVirt大页特性。

### 配置iothread

#### 简介

当StratoVirt启动了带iothread配置的虚拟机后，会在主机上启动独立于主线程的单独线程，这些单独线程可以用来处理设备的IO请求，一方面提升设备的IO性能，另一方面降低对管理面消息处理的影响。

#### 注意事项

- 支持配置最多8个iothread线程
- 支持磁盘和网卡配置iothread属性
- iothread线程会占用主机CPU资源，在虚拟机内部大IO压力情况下，单个iothread占用的CPU资源取决于磁盘的访问速度，例如普通的SATA盘会占用20%以内CPU资源。

#### 创建iothread线程

**命令行：**

```shell
-object iothread,id=iothread1 -object iothread,id=iothread2
```

参数：

- id：用于标识此iothread线程，该id可以被设置到磁盘或网卡的iothread属性。当启动参数配置了iothread线程信息，虚拟机启动后会在主机上启动相应id名的线程。

#### 配置磁盘或网卡的iothread属性

**命令行配置**

轻量虚拟机：

磁盘

```Conf
-device virtio-blk-device xxx,iothread=iothread1
```

网卡

```Conf
-device virtio-net-device xxx,iothread=iothread2
```

标准虚拟机：

磁盘

```Conf
-device virtio-blk-pci xxx,iothread=iothread1
```

网卡

```Conf
-device virtio-net-pci xxx,iothread=iothread2
```

参数：

1. iothread：设置成 iothread 线程的 id，指明处理本设备 I/O 的线程。
2. xxx: 表示磁盘或者网卡的其他配置

### 配置balloon设备

#### 简介

在虚拟机运行过程中,由虚拟机里的balloon驱动来动态占用或释放内存,从而动态改变这台虚拟机当前可用内存，达到内存弹性的效果。

#### 注意事项

- 启用balloon前须确保guest和host的页面大小相同。
- guest内核须开启balloon特性支持。
- 开启内存弹性时，有可能造成虚拟机内部轻微卡顿、内存性能下降。

#### 互斥特性

- 大页内存互斥。
- 在x86下，由于中断数量有限，所以balloon设备和其他virtio的数量（默认使用6个block设备，2个net设备和1个串口设备）总和不得超过11个。

#### 规格

- 每个VM只能配置1个balloon设备。

#### 配置方式

轻量级虚拟机：

```Conf
-device virtio-balloon-device[,deflate-on-oom=true|false][,free-page-reporting=true|false]
```

标准虚拟机：

```Conf
-device virtio-balloon-pci,bus=pcie.0,addr=0x4.0x0[,deflate-on-oom=true|false][,free-page-reporting=true|false][,multifunction=on|off]
```

[!NOTE]说明

1. deflate-on-oom的取值为bool类型，表示是否开启auto deflate特性。开启时，如果balloon已经回收部分内存，当guest需要内存时，balloon设备会自动放气，归还内存给guest。不开启则不会自动归还。
2. free-page-reporting的取值为bool类型，表示是否开启free page reporting特性。开启时，如果guest内核向balloon设备发送了free pages，balloon将释放free pages所占用的内存。不开启则guest内核不会向balloon设备发送free pages。
3. 使用qmp命令回收虚拟机内存时，应确保回收后虚拟机仍然有足够的内存来保持最基本的运行。否则可能会出现一些操作超时，以及导致虚拟机内部无法申请到空闲内存等现象。
4. 如果虚拟机内部开启内存大页，balloon不能回收大页占用内存。

> deflate-on-oom=false时，当Guest中内存不足时，balloon不会自动放气并归还内存，可能会引起Guest内部OOM，进程被Kill，甚至虚拟机无法正常运行。

### 配置RNG设备

#### 简介

Virtio RNG是半虚拟化的随机数生成器设备，用于为guest提供硬件随机数生成能力。

#### 配置方式

Virtio RNG可配置为Virtio mmio设备或者virtio PCI设备，Virtio RNG配置为Virtio mmio设备时，命令行参数如下：

```Conf
-object rng-random,id=objrng0,filename=/path/to/random_file
-device virtio-rng-device,rng=objrng0,max-bytes=1234,period=1000
```

Virtio RNG配置为Virtio PCI设备时，命令行参数如下：

```Conf
-object rng-random,id=objrng0,filename=/path/to/random_file
-device virtio-rng-pci,rng=objrng0,max-bytes=1234,period=1000,bus=pcie.0,addr=0x1.0x0,id=rng-id[,multifunction=on]
```

参数：

- filename：在host上用于生成随机数的字符设备路径，例如/dev/random；
- period：限制随机数字符速率的定时周期，单位为毫秒；
- max-bytes：在period时间内字符设备生成随机数的最大字节数；
- bus：Virtio RNG设备挂载的总线名称；
- addr：Virtio RNG设备地址，参数格式为addr=[slot].[function]，分别表示设备的slot号和function号，均使用十六进制表示，其中Virtio RNG设备的function号为0x0。

#### 注意事项

- 如不配置period和max-bytes，则不对随机数字符读取速率进行限制；
- 如配置限速，则max-bytes/period\*1000的设定范围为[64, 1000000000]，建议不应设置过小，以防获取随机数字符速率过慢；
- 只能限制平均随机数字符数，无法限制瞬间的突发流量；
- guest如需使用Virtio RNG设备，guest内核需要使能配置：CONFIG_HW_RANDOM=y，CONFIG_HW_RANDOM_VIA=y，CONFIG_HW_RANDOM_VIRTIO=y；
- 用户在配置Virtio RNG设备时，请检查熵池是否足够，以免引起虚拟机卡顿问题，例如配置字符设备路径为/dev/random，当前熵池大小可通过/proc/sys/kernel/random/entropy_avail查看，熵池满时的大小为4096，通常应该大于1000。

### 配置VNC

#### 简介

用户可以通过VNC客户端登录虚拟机，输入鼠标键盘事件，并通过VNC显示的桌面完成与远程虚拟机系统的交互。

#### 注意事项

- 当前只有标准虚拟机支持VNC特性。
- 目前只支持RFB3.3-3.8版本客户端连接。
- 目前只支持单个客户端连接，暂不支持多个客户端同时连接。多个客户端连接会返回连接失败。
- 目前仅支持在ARM环境上使用。

#### 互斥特性

- VNC特性暂不支持热迁移。

#### 规格

- 每个虚拟机只支持配置一个VNC Server。

#### 配置方式

标准虚拟机:

```shell
-vnc 0.0.0.0:11
```

[!NOTE]说明

1. 图像渲染用到`pixman`库，需要在虚拟机运行环境中安装`pixman.rpm`和`pixman-devel.rpm`两个包。
2. 鼠标键盘输入需要配置一个`USB`控制器，以及鼠标键盘设备。
3. 需要配置一个显示设备，如`virtio-gpu`、`ramfb`。

### 配置 USB 键盘和 USB 鼠标

#### 简介

StratoVirt 支持配置 USB 键盘和 USB 鼠标，用户可以通过 VNC 远程连接虚拟机，通过 USB 键盘鼠标对虚拟机进行图形化操作。USB 设备需要挂载在 USB 控制器上，因此需要提前在命令行里配置 USB 控制器。

#### 注意事项

- 当前只有标准虚拟机支持 USB 键盘鼠标。

#### 互斥特性

- USB 键盘鼠标暂不支持热迁移。

#### 规格

- 每个 VM 只能配置 1 个 USB 控制器
- 每个 VM 只能配置 1 个 USB 键盘
- 每个 VM 只能配置 1 个 USB 鼠标

#### 配置方式

USB 控制器在启动 StratoVirt 时命令行配置：

```Conf
-device nec-usb-xhci,id=xhci,bus=pcie.0,addr=0xa.0x0
```

参数：

- id：唯一的设备 id。
- bus：设备要挂载的 bus。
- addr：设备要挂载的 slot 和 function 号。

注意需要合理配置设备的 bus 和 addr 参数，不能和其他配置的 PCI 设备冲突，否则可能会导致虚拟机启动失败。

USB 键盘在启动 StratoVirt 时命令行配置：

```Conf
-device usb-bkd,id=kbd
```

参数：

- id：唯一的设备 id。

USB 鼠标在启动 StratoVirt 时命令行配置：

```Conf
-device usb-tablet,id=tablet
```

参数：

- id：唯一的设备 id。

### 配置virtio-gpu设备

#### 简介

标准虚拟机可支持配置virtio-gpu显卡用于显示。

#### 注意事项

- 目前仅支持2D。
- max_hostmem（即在host侧可占用内存）建议不小于256MiB，否则影响分辨率配置。
- max_outputs（即支持的屏幕数量）配置不可大于16。
- 不支持热迁移。

#### 规格

- 每个VM只能配置1个virtio-gpu设备。

#### 配置方式

标准虚拟机：

```Conf
-device virtio-gpu-pci,id=XX,bus=pcie.0,addr=0x2.0x0[,max_outputs=XX][,edid=true|false][,xres=XX][,yres=XX][,max_hostmem=XX]
```

参数：

1. max_outputs：当前显卡需要支持的屏幕数量，建议配置为1，最大值不超过16。
2. edid：当前显卡是否支持edid，建议配置为true，虚拟机内核会检查显卡是否支持edid。
3. xres/yres：登录窗口的横向/纵向大小。
4. max_hostmem: 显卡最大可占用host侧内存, 以Byte为单位。

## 配置示例

### 轻量级虚拟机

此处给出创建一个轻量级虚拟机的最小配置示例。

1. 登录主机，删除 socket 文件，确保可以创建 QMP。

   ```sh
   # rm -f /tmp/stratovirt.socket
   ```

2. 运行 StratoVirt 。

   ```sh
   # /path/to/stratovirt \
       -kernel /path/to/vmlinux.bin \
       -append console=ttyS0 root=/dev/vda rw reboot=k panic=1 \
       -drive file=/home/rootfs.ext4,id=rootfs,readonly=false \
       -device virtio-blk-device,drive=rootfs \
       -qmp unix:/tmp/stratovirt.socket,server,nowait \
       -serial stdio
   ```

   运行成功后，将根据指定的配置参数创建并启动虚拟机。

### 标准虚拟机

此处给出在 ARM 平台创建一个标准虚拟机的最小配置示例。

1. 删除 socket 文件，确保可以创建 QMP 。

   ```sh
   # rm -f /tmp/stratovirt.socket
   ```

2. 运行 StratoVirt 。

   ```sh
   # /path/to/stratovirt \
       -kernel /path/to/vmlinux.bin \
       -append console=ttyAMA0 root=/dev/vda rw reboot=k panic=1 \
       -drive file=/path/to/edk2/code_storage_file,if=pflash,unit=0[,readonly=true] \
       -drive file=/path/to/edk2/data_storage_file,if=pflash,unit=1, \
       -drive file=/home/rootfs.ext4,id=rootfs,readonly=false \
       -device virtio-blk-device,drive=rootfs,bus=pcie.0,addr=0x1 \
       -qmp unix:/tmp/stratovirt.socket,server,nowait \
       -serial stdio
   ```
