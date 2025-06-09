# StratoVirt-VFIO使用

## 管理设备直通

虚拟化平台使用设备直通，可以使虚拟机直接使用相关硬件设备，提升虚拟机性能。本章介绍 StratoVirt 支持的设备直通特性。

### 前提条件

使用设备直通，主机需要满足如下条件：

1. 开启 IOMMU 功能

   IOMMU 全称是 Input/Output Memory Management Unit，该技术可以让 PCI/PCIe 设备的资源直接分配给虚拟机。

   在主机上执行如下命令，查看 IOMMU 是否已经开启。

   ```shell
   # dmesg | grep iommu
   ```

   若已开启，回显如下：

   ```shell
   iommu: Default domain type: Translated
   hibmc-drm 0000:0a:00.0: Adding to iommu group 0
   ehci-pci 0000:7a:01.0: Adding to iommu group 1
   ehci-pci 0000:ba:01.0: Adding to iommu group 2
   ohci-pci 0000:7a:00.0: Adding to iommu group 3
   ohci-pci 0000:ba:00.0: Adding to iommu group 4
   xhci_hcd 0000:7a:02.0: Adding to iommu group 5
   ......
   ```

   若未开启，则没有回显或只显示如下信息：

   ```shell
   iommu: Default domain type: Translated
   ```

   开启IOMMU：

   1. 为linux内核增加启动参数: `intel_iommu=on iommu=pt`;

      ```shell
      vim /boot/grub2/grub.cfg
      linux /vmlinuz-5.15.0+ root=/dev/mapper/openeuler-root ro resume=/dev/mapper/openeuler-swap rd.lvm.lv=openeuler/root rd.lvm.lv=openeuler/swap crashkernel=512M intel_iommu=on iommu=pt
      ```

   2. 重启Host OS;

2. 加载 vfio-pci 内核模块

   ```shell
   # modprobe vfio-pci
   # lsmod | grep vfio_pci
   ```

   成功加载 vfio-pci 模块，则回显如下：

   ```shell
   vfio_pci              327680  0
   vfio_virqfd           327680  1 vfio_pci
   vfio                  327680  2 vfio_iommu_type1,vfio_pci
   ```

3. 将 PCI 设备从主机解绑，重新绑定到 vfio-pci 驱动

   假设使用 VFIO 直通 Hi1822 网卡设备，首先查看网卡设备对应的 PCI 设备信息：

   ```shell
   # lspci -v | grep "Eth" | grep 1822
   03:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
   04:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
   05:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
   06:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
   ```

   选择其中 bus 号 03，slot 号 00，function 号 0 的设备，即上述的 03:00.0。然后将该 PCI 设备从主机上解绑

   ```shell
   # echo 0000:03:00.0 > /sys/bus/pci/devices/0000:03:00.0/driver/unbind
   ```

   最后将该 PCI 设备重新绑定到 vfio-pci 驱动。

   ```shell
   lspci -ns 0000:03:00.0 |awk -F':| ' '{print $5" "$6}' > /sys/bus/pci/drivers/vfio-pci/new_id
   ```

   将网卡绑定到 vfio-pci 驱动后，在主机上无法查询到对应网卡信息，只能查询到对应的 PCI 设备信息。

### VFIO 设备直通

#### 简介

VFIO(Virtual Function I/O) 是内核提供的一种用户态设备驱动方案。VFIO 驱动可以安全地把设备 I/O，中断，DMA 等能力呈现给用户空间。StratoVirt 虚拟化平台使用 VFIO 设备直通方案后，在虚拟机可以极大限度地提升 I/O 性能。

#### 使用 VFIO 直通

StratoVirt 支持 libvirt 管理，可以使用 XML 文件配置虚拟机。以下内容介绍通过修改虚拟机 XML 文件的方式，使用 VFIO 设备直通功能。

一、修改 XML 文件

1. 在主机上执行如下命令，查询 CPU 架构信息

   ```shell
   # uname -m
   ```

2. aarch64 和 x86_64 架构分别[下载](https://gitee.com/openeuler/stratovirt/tree/master/docs) StratoVirt 自带的 XML 文件 stratovirt_aarch64.xml 或 stratovirtvirt_x86.xml，并存放到任一目录，例如 /home：

   ```shell
   # cp stratovirt/docs/stratovirt_$arch.xml /home
   ```

3. 根据实际需求，修改XML文件中的VFIO配置。 bus，slot，function 为上述绑定到 vfio-pci 驱动的 PCI 设备。相关配置如下：

   ```shell
   <!-- vfio -->
   <hostdev mode='subsystem' type='pci' managed='yes'>
   <driver name='vfio'/>
   <source>
       <address domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
   </source>
   </hostdev>
   ```

   上例中，设备类型为 PCI 设备，managed='yes' 表示 libvirt 将把 PCI 设备从主机解绑，并重新绑定到 vfio-pci 驱动。source 项配置了需要作为 VFIO 直通设备的 domain，bus，slot，function 信息。

二、使用 libvirt 命令行创建并登录虚拟机

```shell
# virsh create stratovirt_$arch.xml
# virsh list --all
Id  Name   State
--------------------
1  StratoVirt  running
# virsh console 1
```

三、在虚拟机内查看并使用 VFIO 直通网卡。

1. 配置前查看网卡信息

   ```shell
   # ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
   2: enp1s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 72:b8:51:9d:d1:27 brd ff:ff:ff:ff:ff:ff
   ```

2. 动态配置网卡的 IP 地址

   ```shell
   # dhclient
   ```

3. 查询 IP 是否配置成功

   ```shell
   # ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
     valid_lft forever preferred_lft forever
   2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 72:b8:51:9d:d1:27 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.3/16 brd 192.168.255.255 scope global dynamic enp1s0
     valid_lft 86453sec preferred_lft 86453sec
   ```

   如上回显可知，成功分配了 IP 地址 192.168.1.3，虚拟机可以直接使用配置的网卡

   说明：使用的直通网卡如果没有连接物理网络，将获取不到网络信息。

#### 解绑 VFIO 驱动

如果需要将直通给虚拟机使用的网卡解除绑定，可以登录主机，执行如下命令，将网卡设备重新绑定到主机上。其中，hinic是对应网卡设备驱动的类型：

```shell
# echo 0000:03:00.0 > /sys/bus/pci/drivers/vfio-pci/unbind
# echo 0000:03:00.0 > /sys/bus/pci/drivers/hinic/bind
```

说明：绑定 VFIO 驱动前，可以再主机上执行 ethtool -i enp0 命令，获取网卡设备驱动类型。enp0 为对应网卡名称。

### SR-IOV 直通

#### 简介

使用 VFIO 设备直通时，虚拟机能直接访问硬件，但每个设备只能被一个虚拟机独占。SR-IOV 直通技术支持将一个 PF(Physical Function) 虚拟出多个 VF (Virtual Function)，并直通给不同虚拟机，解决了设备直通的独占问题，增加可用的设备。

#### 操作步骤

1. 创建多个 VF：

   sriov_numvfs 文件用于描述 SR-IOV 提供的 VF 个数，存放在 /sys/bus/pci/devices/domain\:bus\:slot.function/ 路径下，例如上述例子中的 bus 号 03，slot 号 00，function 号 0 的设备，可以使用如下命令创建4个 VF：

   ```shell
   # echo 4 > /sys/bus/pci/devices/0000\:03\:00.0/sriov_numvfs
   ```

2. 确认 VF 设备创建成功

   ```shell
   # lspci -v | grep "Eth" | grep 1822
   ```

   回显如下，说明成功创建了4个 VF 03:00.1、03:00.2、03:00.3、03:00.4：

   ```shell
   03:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
   03:00.1 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
   03:00.2 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
   03:00.3 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
   03:00.4 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
   ```

3. 上述创建的 VF 设备均可以直通给虚拟机，使用 SR-IOV 设备的方法与普通 PCI 设备的直通方法相同。
