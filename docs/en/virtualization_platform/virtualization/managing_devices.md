# Managing Devices

## Configuring a PCIe Controller for a VM

### Overview

The NIC, disk controller, and PCIe pass-through devices in a VM must be mounted to a PCIe root port. Each root port corresponds to a PCIe slot. The devices mounted to the root port support hot swap, but the root port does not support hot swap. Therefore, users need to consider the hot swap requirements and plan the maximum number of PCIe root ports reserved for the VM. Before the VM is started, the root port is statically configured.

### Configuring the PCIe Root, PCIe Root Port, and PCIe-PCI-Bridge

The VM PCIe controller is configured using the XML file. The  **model**  corresponding to PCIe root, PCIe root port, and PCIe-PCI-bridge in the XML file are  **pcie-root**,  **pcie-root-port**, and  **pcie-to-pci-bridge**, respectively.

- Simplified configuration method

    Add the following contents to the XML file of the VM. Other attributes of the controller are automatically filled by libvirt.

    ```xml
      <controller type='pci' index='0' model='pcie-root'/>
      <controller type='pci' index='1' model='pcie-root-port'/>
      <controller type='pci' index='2' model='pcie-to-pci-bridge'/>
      <controller type='pci' index='3' model='pcie-root-port'/>
      <controller type='pci' index='4' model='pcie-root-port'/>
      <controller type='pci' index='5' model='pcie-root-port'/>
    ```

    The  **pcie-root**  and  **pcie-to-pci-bridge**  occupy one  **index**  respectively. Therefore, the final  **index**  is the number of required  **root ports + 1**.

- Complete configuration method

    Add the following contents to the XML file of the VM:

    ```xml
      <controller type='pci' index='0' model='pcie-root'/>
      <controller type='pci' index='1' model='pcie-root-port'>
        <model name='pcie-root-port'/>
        <target chassis='1' port='0x8'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0' multifunction='on'/>
      </controller>
      <controller type='pci' index='2' model='pcie-to-pci-bridge'>
        <model name='pcie-pci-bridge'/>
        <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      </controller>
      <controller type='pci' index='3' model='pcie-root-port'>
        <model name='pcie-root-port'/>
        <target chassis='3' port='0x9'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
      </controller>
      <controller type='pci' index='3' model='pcie-root-port'>
    ```

    In the preceding contents:

    - The  **chassis**  and  **port**  attributes of the root port must be in ascending order. Because a PCIe-PCI-bridge is inserted in the middle, the  **chassis**  number skips  **2**, but the  **port**  numbers are still consecutive.
    - The  **address function**  of the root port ranges from  **0\*0**  to  **0\*7**.
    - A maximum of eight functions can be mounted to each slot. When the slot is full, the slot number increases.

    The complete configuration method is complex. Therefore, the simplified one is recommended.

## Managing Virtual Disks

### Overview

Virtual disk types include virtio-blk, virtio-scsi, and vhost-scsi. virtio-blk simulates a block device, and virtio-scsi and vhost-scsi simulate SCSI devices.

- virtio-blk: It can be used for common system disk and data disk. In this configuration, the virtual disk is presented as  **vd\[a-z\]**  or  **vd\[a-z\]\[a-z\]**  in the VM.
- virtio-scsi: It is recommended for common system disk and data disk. In this configuration, the virtual disk is presented as  **sd\[a-z\]**  or  **sd\[a-z\]\[a-z\]**  in the VM.
- vhost-scsi: It is recommended for the virtual disk that has high performance requirements. In this configuration, the virtual disk is presented as  **sd\[a-z\]**  or  **sd\[a-z\]\[a-z\]**  on the VM.

### Procedure

For details about how to configure a virtual disk, see **VM Configuration** > **Network Devices**. This section uses the virtio-scsi disk as an example to describe how to attach and detach a virtual disk.

- Attach a virtio-scsi disk.

    Run the  **virsh attach-device**  command to attach the virtio-scsi virtual disk.

    ```shell
     # virsh attach-device <VMInstance> <attach-device.xml>
    ```

    The preceding command can be used to attach a disk to a VM online. The disk information is specified in the  **attach-device.xml**  file. The following is an example of the  **attach-device.xml**  file:

    ```shell
    ### attach-device.xml ###
        <disk type='file' device='disk'>
          <driver name='qemu' type='qcow2' cache='none' io='native'/>
          <source file='/path/to/another/qcow2-file'/>
          <backingStore/>
          <target dev='sdb' bus='scsi'/>
          <address type='drive' controller='0' bus='0' target='1' unit='0'/>
        </disk>
    ```

    The disk attached by running the preceding commands becomes invalid after the VM is shut down and restarted. If you need to permanently attach a virtual disk to a VM, run the  **virsh attach-device**  command with the  **--config**  parameter.

- Detach a virtio-scsi disk.

    If a disk attached online is no longer used, run the  **virsh detach**  command to dynamically detach it.

    ```shell
     # virsh detach-device <VMInstance> <detach-device.xml>
    ```

    **detach-device.xml**  specifies the XML information of the disk to be detached, which must be the same as the XML information during dynamic attachment.

## Managing vNICs

### Overview

The vNIC types include virtio-net, vhost-net, and vhost-user. After creating a VM, you may need to attach or detach a vNIC. openEuler supports NIC hot swap, which can change the network throughput and improve system flexibility and scalability.

### Procedure

For details about how to configure a virtual NIC, see  [3.2.4.2 Network Devices](./vm_configuration.md#network-devices). This section uses the vhost-net NIC as an example to describe how to attach and detach a vNIC.

- Attach the vhost-net NIC.

    Run the  **virsh attach-device**  command to attach the vhost-net vNIC.

    ```shell
     # virsh attach-device <VMInstance> <attach-device.xml>
    ```

    The preceding command can be used to attach a vhost-net NIC to a running VM. The NIC information is specified in the  **attach-device.xml**  file. The following is an example of the  **attach-device.xml**  file:

    ```shell
    ### attach-device.xml ###
        <interface type='bridge'>
          <mac address='52:54:00:76:f2:bb'/>
          <source bridge='br0'/>
          <virtualport type='openvswitch'/>
          <model type='virtio'/>
          <driver name='vhost' queues='2'/>
        </interface>
    ```

    The vhost-net NIC attached using the preceding commands becomes invalid after the VM is shut down and restarted. If you need to permanently attach a vNIC to a VM, run the  **virsh attach-device**  command with the  **--config**  parameter.

- Detach the vhost-net NIC.

    If a NIC attached online is no longer used, run the  **virsh detach**  command to dynamically detach it.

    ```shell
     # virsh detach-device <VMInstance> <detach-device.xml>
    ```

    **detach-device.xml**  specifies the XML information of the vNIC to be detached, which must be the same as the XML information during dynamic attachment.

## Configuring a Virtual Serial Port

### Overview

In a virtualization environment, VMs and host machines need to communicate with each other to meet management and service requirements. However, in the complex network architecture of the cloud management system, services running on the management plane and VMs running on the service plane cannot communicate with each other at layer 3. As a result, service deployment and information collection are not fast enough. Therefore, a virtual serial port is required for communication between VMs and host machines. You can add serial port configuration items to the XML configuration file of a VM to implement communication between VMs and host machines.

### Procedure

The Linux VM serial port console is a pseudo terminal device connected to the host machine through the serial port of the VM. It implements interactive operations on the VM through the host machine. In this scenario, the serial port needs to be configured in the pty type. This section describes how to configure a pty serial port.

- Add the following virtual serial port configuration items under the  **devices**  node in the XML configuration file of the VM:

    ```xml
        <serial type='pty'>
        </serial>
        <console type='pty'>
          <target type='serial'/>
        </console>
    ```

- Run the  **virsh console**  command to connect to the pty serial port of the running VM.

    ```shell
    # virsh console <VMInstance>
    ```

- To ensure that no serial port message is missed, use the  **--console**  option to connect to the serial port when starting the VM.

    ```shell
    # virsh start --console <VMInstance>
    ```

## Managing Device Passthrough

The device passthrough technology enables VMs to directly access physical devices. The I/O performance of VMs can be improved in this way.

Currently, the VFIO passthrough is used. It can be classified into PCI passthrough and SR-IOV passthrough based on device type.

### PCI Passthrough

PCI passthrough directly assigns a physical PCI device on the host to a VM. The VM can directly access the device. PCI passthrough uses the VFIO device passthrough mode. The PCI passthrough configuration file in XML format for a VM is as follows:

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>   
    <driver name='vfio'/> 
    <source>
        <address domain='0x0000' bus='0x04' slot='0x10' function='0x01'/>
    </source>
    <rom bar='off'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</hostdev>
```

**Table  1**  Device configuration items for PCI passthrough

| Parameter | Description | Value |
|---|---|---|
| hostdev.source.address.domain | Domain ID of the PCI device on the host OS. | ≥ 0 |
| hostdev.source.address.bus | Bus ID of the PCI device on the host OS. | ≥ 1 |
| hostdev.source.address.slot | Device ID of the PCI device on the host OS. | ≥ 0 |
| hostdev.source.address.function | Function ID of the PCI device on the host OS. | ≥ 0 |
| hostdev.driver.name | Backend driver of PCI passthrough. This parameter is optional. | **vfio** (default value) |
| hostdev.rom | Whether the VM can access the ROM of the passthrough device. | This parameter can be set to **on** or **off**. The default value is **on**. <br> - **on**: indicates that the VM can access the ROM of the passthrough device. For example, if a VM with a passthrough NIC needs to boot from the preboot execution environment (PXE), or a VM with a passthrough Host Bus Adapter (HBA) card needs to boot from the ROM, you can set this parameter to **on**. <br> - **off**: indicates that the VM cannot access the ROM of the passthrough device. |
| hostdev.address.type | Device type displayed on the guest, which must be the same as the actual device type. | **pci** (default configuration) |
| hostdev.address.domain | Domain number of the device displayed on the guest. | 0x0000 |
| hostdev.address.bus | Bus number of the device displayed on the guest. | **0x00** (default configuration). This parameter can only be set to the bus number configured in section "Configuring a PCIe Controller for a VM." |
| hostdev.address.slot | Slot number of the device displayed on the guest. | The slot number range is \[0x03,0x1e] <br> Note:  - The first slot number 0x00 is occupied by the system, the second slot number 0x01 is occupied by the IDE controller and USB controller, and the third slot number 0x02 is occupied by the video. - The last slot number 0x1f is occupied by the pvchannel. |
| hostdev.address.function | Function number of the device displayed on the guest. | **0x0** (default configuration): The function number range is \[0x0,0x7] |

> [!NOTE]NOTE   
> VFIO passthrough is implemented by IOMMU group. Devices are divided to IOMMU groups based on access control services (ACS) on hardware. Devices in the same IOMMU group can be assigned to only one VM. If multiple functions on a PCI device belong to the same IOMMU group, they can be directly assigned to only one VM as well.  

### SR-IOV Passthrough

#### Overview

Single Root I/O Virtualization (SR-IOV) is a hardware-based virtualization solution. With the SR-IOV technology, a physical function (PF) can provide multiple virtual functions (VFs), and each VF can be directly assigned to a VM. This greatly improves hardware resource utilization and I/O performance of VMs. A typical application scenario is SR-IOV passthrough for NICs. With the SR-IOV technology, a physical NIC (PF) can function as multiple VF NICs, and then the VFs can be directly assigned to VMs.

> [!NOTE]NOTE   
>
> - SR-IOV requires the support of physical hardware. Before using SR-IOV, ensure that the hardware device to be directly assigned supports SR-IOV and the device driver on the host OS works in SR-IOV mode.  
> - The following describes how to query the NIC model:  
> In the following command output, values in the first column indicate the PCI numbers of NICs, and  **19e5:1822**  indicates the vendor ID and device ID of the NIC.  
>
> ```shell
> # lspci | grep Ether  
> 05:00.0 Ethernet controller: Device 19e5:1822 (rev 45)  
> 07:00.0 Ethernet controller: Device 19e5:1822 (rev 45)  
> 09:00.0 Ethernet controller: Device 19e5:1822 (rev 45)  
> 0b:00.0 Ethernet controller: Device 19e5:1822 (rev 45)  
> 81:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)  
> 81:00.1 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)  
> ```

#### Procedure

To configure SR-IOV passthrough for a NIC, perform the following steps:

1. Enable the SR-IOV mode for the NIC.
    1. Ensure that VF driver support provided by the NIC supplier exists on the guest OS. Otherwise, VFs in the guest OS cannot work properly.
    2. Enable the SMMU/IOMMU support in the BIOS of the host OS. The enabling method varies depending on the servers of different vendors. For details, see the help documents of the servers.
    3. Configure the host driver to enable the SR-IOV VF mode. The following uses the Hi1822 NIC as an example to describe how to enable 16 VFs.

        ```shell
        echo 16 > /sys/class/net/ethX/device/sriov_numvfs
        ```

2. Obtain the PCI BDF information of PFs and VFs.
    1. Run the following command to obtain the NIC resource list on the current board:

        ```shell
        # lspci | grep Eth
        03:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
        04:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
        05:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
        06:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family (4*25GE) (rev 45)
        7d:00.0 Ethernet controller: Huawei Technologies Co., Ltd. Device a222 (rev 20)
        7d:00.1 Ethernet controller: Huawei Technologies Co., Ltd. Device a222 (rev 20)
        7d:00.2 Ethernet controller: Huawei Technologies Co., Ltd. Device a221 (rev 20)
        7d:00.3 Ethernet controller: Huawei Technologies Co., Ltd. Device a221 (rev 20)
        ```

    2. Run the following command to view the PCI BDF information of VFs:

        ```shell
        # lspci | grep "Virtual Function"
        03:00.1 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.2 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.3 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.4 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.5 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.6 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:00.7 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:01.0 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:01.1 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        03:01.2 Ethernet controller: Huawei Technologies Co., Ltd. Hi1822 Family Virtual Function (rev 45)
        ```

    3. Select an available VF and write its configuration to the VM configuration file based on its BDF information. For example, the bus ID of the device  **03:00.1**  is  **03**, its slot ID is  **00**, and its function ID is  **1**.

3. Identify and manage the mapping between PFs and VFs.
    1. Identify VFs corresponding to a PF. The following uses PF 03.00.0 as an example:

        ```shell
        # ls -l /sys/bus/pci/devices/0000\:03\:00.0/
        ```

        The following symbolic link information is displayed. You can obtain the VF IDs (virtfnX) and PCI BDF IDs based on the information.

    2. Identify the PF corresponding to a VF. The following uses VF 03:00.1 as an example:

        ```shell
        # ls -l /sys/bus/pci/devices/0000\:03\:00.1/
        ```

        The following symbolic link information is displayed. You can obtain PCI BDF IDs of the PF based on the information.

        ```shell
        lrwxrwxrwx 1 root root       0 Mar 28 22:44 physfn -> ../0000:03:00.0
        ```

    3. Obtain names of NICs corresponding to the PFs or VFs. For example:

        ```shell
        # ls /sys/bus/pci/devices/0000:03:00.0/net
        eth0
        ```

    4. Set the MAC address, VLAN, and QoS information of VFs to ensure that the VFs are in the  **Up**  state before passthrough. The following uses VF 03:00.1 as an example. The PF is eth0 and the VF ID is  **0**.

        ```shell
        # ip link set eth0 vf 0 mac 90:E2:BA:21:XX:XX    #Sets the MAC address.
        # ifconfig eth0 up
        # ip link set eth0 vf 0 rate 100                 #Sets the VF outbound rate, in Mbit/s.
        # ip link show eth0                              #Views the MAC address, VLAN ID, and QoS information to check whether the configuration is successful.
        ```

4. Mount the SR-IOV NIC to the VM.

    When creating a VM, add the SR-IOV passthrough configuration item to the VM configuration file.

    ```xml
    <interface type='hostdev' managed='yes'> 
        <mac address='fa:16:3e:0a:xx:xx'/>
        <source> 
            <address type='pci' domain='0x0000' bus='0x06' slot='0x11' function='0x6'/>
        </source> 
        <vlan>
            <tag id='1'/>
        </vlan>
    </interface>
    ```

    **Table  1**  SR-IOV configuration options

    | Parameter                       | Description                                   | Value                                                                                                                                                                                               |
    | ------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | hostdev.managed                 | Two modes for libvirt to process PCI devices. | **no**: default value. The passthrough device is managed by the user. <br>**yes**: The passthrough device is managed by libvirt. Set this parameter to **yes** in the SR-IOV passthrough scenario. |
    | hostdev.source.address.bus      | Bus ID of the PCI device on the host OS.      | ≥ 1                                                                                                                                                                                                 |
    | hostdev.source.address.slot     | Device ID of the PCI device on the host OS.   | ≥ 0                                                                                                                                                                                                 |
    | hostdev.source.address.function | Function ID of the PCI device on the host OS. | ≥ 0                                                                                                                                                                                                 |

    > [!NOTE]NOTE   
    > Disabling the SR-IOV function:  
    > To disable the SR-IOV function after the VM is stopped and no VF is in use, run the following command:  
    > The following uses the Hi1822 NIC corresponding network interface name: eth0) as an example:  

    ```shell
    echo 0 > /sys/class/net/eth0/device/sriov_numvfs  
    ```

#### Configuring SR-IOV Passthrough for the HPRE Accelerator

The accelerator engine is a hardware acceleration solution provided by TaiShan 200 servers. The HPRE accelerator is used to accelerate SSL/TLS applications. It significantly reduces processor consumption and improves processor efficiency. 
On the Kunpeng server, you need to pass through the VFs of the HPRE accelerator on the host to the VM for internal services of the VM.

**Table 1** HPRE accelerator description

|             | Description                                                                                               |
|-------------|-----------------------------------------------------------------------------------------------------|
| Device name   | Hi1620 on-chip RSA/DH security algorithm accelerator (HPRE engine)                                  |
| Description       | Modular exponentiation, RSA key pair operation, DH calculation, and some large-number auxiliary operations (modular exponentiation, modular multiplication, modulo operation, multiplication, modular inversion, prime number test, and mutual prime test)|
| VendorID    | 0x19E5                                                                                              |
| PF DeviceID | 0xA258                                                                                              |
| VF DeviceID | 0xA259                                                                                              |
| Maximum number of VFs | An HPRE PF supports a maximum of 63 VFs.                                                                      |

> [!NOTE]NOTE  
> When a VM is using a VF device, the driver on the host cannot be uninstalled, and the accelerator does not support hot swap. 
> VF operation (If **VFNUMS** is **0**, the VF is disabled, and **hpre_num** is used to identify a specific accelerator device): 
>
> ```shell
> echo $VFNUMS > /sys/class/uacce/hisi_hpre-$hpre_num/device/sriov_numvfs
> ```

### vDPA Passthrough

#### Overview

vDPA passthrough connects a device on a host to the vDPA framework, uses the vhost-vdpa driver to present a character device, and configures the character device for VMs to use. vDPA passthrough drives can serve as system or data drives for VMs and support hot expansion of data drives.

vDPA passthrough provides the similar I/O performance as VFIO passthrough, provides flexibility of VirtIO devices, and supports live migration of vDPA passthrough devices.

With the SR-IOV solution, vDPA passthrough can virtualize a physical NIC (PF) into multiple NICs (VFs), and then connect the VFs to the vDPA framework for VMs to use.

#### Procedure

To configure vDPA passthrough, perform the following steps as user **root**:

1. Create and configure VFs. For details, see steps 1 to 3 in SR-IOV passthrough. The following uses **virtio-net** devices as an example (**08:00.6** and **08:00.7** are PFs, and the others are created VFs):

    ```shell
    # lspci | grep -i Eth | grep Virtio
    08:00.6 Ethernet controller: Virtio: Virtio network device
    08:00.7 Ethernet controller: Virtio: Virtio network device
    08:01.1 Ethernet controller: Virtio: Virtio network device
    08:01.2 Ethernet controller: Virtio: Virtio network device
    08:01.3 Ethernet controller: Virtio: Virtio network device
    08:01.4 Ethernet controller: Virtio: Virtio network device
    08:01.5 Ethernet controller: Virtio: Virtio network device
    08:01.6 Ethernet controller: Virtio: Virtio network device
    08:01.7 Ethernet controller: Virtio: Virtio network device
    08:02.0 Ethernet controller: Virtio: Virtio network device
    08:02.1 Ethernet controller: Virtio: Virtio network device
    08:02.2 Ethernet controller: Virtio: Virtio network device
    ```

2. Unbind the VF drivers and bind the vDPA driver of the hardware vendor.

    ```shell
    echo 0000:08:01.1 > /sys/bus/pci/devices/0000\:08\:01.1/driver/unbind
    echo 0000:08:01.2 > /sys/bus/pci/devices/0000\:08\:01.2/driver/unbind
    echo 0000:08:01.3 > /sys/bus/pci/devices/0000\:08\:01.3/driver/unbind
    echo 0000:08:01.4 > /sys/bus/pci/devices/0000\:08\:01.4/driver/unbind
    echo 0000:08:01.5 > /sys/bus/pci/devices/0000\:08\:01.5/driver/unbind
    echo -n "1af4 1000" > /sys/bus/pci/drivers/vender_vdpa/new_id
    ```

3. After vDPA devices are bound, you can run the `vdpa` command to query the list of devices managed by vDPA.

    ```shell
    # vdpa mgmtdev show
    pci/0000:08:01.1:
        supported_classes net
    pci/0000:08:01.2:
        supported_classes net
    pci/0000:08:01.3:
        supported_classes net
    pci/0000:08:01.4:
        supported_classes net
    pci/0000:08:01.5:
        supported_classes net
    ```

4. After the vDPA devices are created, create the vhost-vDPA devices.

    ```shell
    vdpa dev add name vdpa0 mgmtdev pci/0000:08:01.1
    vdpa dev add name vdpa1 mgmtdev pci/0000:08:01.2
    vdpa dev add name vdpa2 mgmtdev pci/0000:08:01.3
    vdpa dev add name vdpa3 mgmtdev pci/0000:08:01.4
    vdpa dev add name vdpa4 mgmtdev pci/0000:08:01.5
    ```

5. After the vhost-vDPA devices are created, you can run the `vdpa` command to query the vDPA device list or run the `libvirt` command to query the vhost-vDPA device information.

    ```shell
    # vdpa dev show
    vdpa0: type network mgmtdev pci/0000:08:01.1 vendor_id 6900 max_vqs 3 max_vq_size 256
    vdpa1: type network mgmtdev pci/0000:08:01.2 vendor_id 6900 max_vqs 3 max_vq_size 256
    vdpa2: type network mgmtdev pci/0000:08:01.3 vendor_id 6900 max_vqs 3 max_vq_size 256
    vdpa3: type network mgmtdev pci/0000:08:01.4 vendor_id 6900 max_vqs 3 max_vq_size 256
    vdpa4: type network mgmtdev pci/0000:08:01.5 vendor_id 6900 max_vqs 3 max_vq_size 256

    # virsh nodedev-list vdpa
    vdpa_vdpa0
    vdpa_vdpa1
    vdpa_vdpa2
    vdpa_vdpa3
    vdpa_vdpa4

    # virsh nodedev-dumpxml vdpa_vdpa0
    <device>
        <name>vdpa_vdpa0</name>
        <path>/sys/devices/pci0000:00/0000:00:0c.0/0000:08:01.1/vdpa0</path>
        <parent>pci_0000_08_01_1</parent>
        <driver>
        <name>vhost_vdpa</name>
        </driver>
        <capability type='vdpa'>
        <chardev>/dev/vhost-vdpa-0</chardev>
        </capability>
    </device>
    ```

6. Mount a vDPA device to the VM.

    When creating a VM, add the item for the vDPA passthrough device to the VM configuration file:

    ```xml
    <devices>
       <hostdev mode='subsystem' type='vdpa'>
           <source dev='/dev/vhost-vdpa-0'/>
       </hostdev>
    </devices>
    ```

    **Table 4** vDPA configuration description

    | Parameter          | Description                                          | Value             |
    | ------------------ | ---------------------------------------------------- | ----------------- |
    | hostdev.source.dev | Path of the vhost-vDPA character device on the host. | /dev/vhost-vdpa-x |

    > [!NOTE]NOTE  
    > The procedures of creating and configuring VFs and binding the vDPA drivers vary with the design of hardware vendors. Follow the procedure of the corresponding vendor.

## Managing VM USB

To facilitate the use of USB devices such as USB key devices and USB mass storage devices on VMs, openEuler provides the USB device passthrough function. Through USB passthrough and hot-swappable interfaces, you can configure USB passthrough devices for VMs, or hot swap USB devices when VMs are running.

### Configuring USB Controllers

#### Overview

A USB controller is a virtual controller that provides specific USB functions for USB devices on VMs. To use USB devices on a VM, you must configure USB controllers for the VM. Currently, openEuler supports the following types of USB controllers:

- Universal host controller interface (UHCI): also called the USB 1.1 host controller specification.
- Enhanced host controller interface (EHCI): also called the USB 2.0 host controller specification.
- Extensible host controller interface (xHCI): also called the USB 3.0 host controller specification.

#### Precautions

- The host server must have USB controller hardware and modules that support USB 1.1, USB 2.0, and USB 3.0 specifications.
- You need to configure USB controllers for the VM by following the order of USB 1.1, USB 2.0, and USB 3.0.
- An xHCI controller has eight ports and can be mounted with a maximum of four USB 3.0 devices and four USB 2.0 devices. An EHCI controller has six ports and can be mounted with a maximum of six USB 2.0 devices. A UHCI controller has two ports and can be mounted with a maximum of two USB 1.1 devices.
- On each VM, only one USB controller of the same type can be configured.
- USB controllers cannot be hot swapped.
- If the USB 3.0 driver is not installed on a VM, the xHCI controller may not be identified. For details about how to download and install the USB 3.0 driver, refer to the official description provided by the corresponding OS distributor.
- To ensure the compatibility of the OS, set the bus ID of the USB controller to  **0**  when configuring a USB tablet for the VM. The tablet is mounted to the USB 1.1 controller by default.

#### Configuration Methods

The following describes the configuration items of USB controllers for a VM. You are advised to configure USB 1.1, USB 2.0, and USB 3.0 to ensure the VM is compatible with three types of devices.

The configuration item of the USB 1.1 controller (UHCI) in the XML configuration file is as follows:

```xml
<controller type='usb' index='0' model='piix3-uhci'>
</controller>
```

The configuration item of the USB 2.0 controller (EHCI) in the XML configuration file is as follows:

```xml
<controller type='usb' index='1' model='ehci'>
</controller>
```

The configuration item of the USB 3.0 controller (xHCI) in the XML configuration file is as follows:

```xml
<controller type='usb' index='2' model='nec-xhci'>
</controller>
```

### Configuring a USB Passthrough Device

#### Overview

After USB controllers are configured for a VM, a physical USB device on the host can be mounted to the VM through device passthrough for the VM to use. In the virtualization scenario, in addition to static configuration, hot swapping the USB device is supported. That is, the USB device can be mounted or unmounted when the VM is running.

#### Precautions

- A USB device can be assigned to only one VM.
- A VM with a USB passthrough device does not support live migration.
- VM creation fails if no USB passthrough devices exist in the VM configuration file.
- Forcibly hot removing a USB storage device that is performing read or write operation may damage files in the USB storage device.

#### Configuration Description

The following describes the configuration items of a USB device for a VM.

Description of the USB device in the XML configuration file:

```xml
<hostdev mode='subsystem' type='usb' managed='yes'>
    <source>        
        <address bus='m' device='n'/>
    </source>
    <address type='usb' bus='x' port='y'/>
</hostdev>
```

- **<address bus='***m_**'device='***n_**'/\>**:  *m_  indicates the USB bus address on the host, and  _n*  indicates the device ID.
- **<address type='usb'bus='***x_**'port='***y_**'\>**: indicates that the USB device is to be mounted to the USB controller specified on the VM.  *x_  indicates the controller ID, which corresponds to the index number of the USB controller configured on the VM.  _y*  indicates the port address. When configuring a USB passthrough device, you need to set this parameter to ensure that the controller to which the device is mounted is as expected.

#### Configuration Methods

To configure USB passthrough, perform the following steps:

1. Configure USB controllers for the VM. For details, see  [Configuring USB Controllers](#configuring-usb-controllers).
2. Query information about the USB device on the host.

    Run the  **lsusb**  command (the  **usbutils**  software package needs to be installed) to query the USB device information on the host, including the bus address, device address, device vendor ID, device ID, and product description. For example:

    ```shell
    # lsusb
    ```

    ```text
    Bus 008 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 007 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 004 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 006 Device 002: ID 0bda:0411 Realtek Semiconductor Corp. 
    Bus 006 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 005 Device 003: ID 136b:0003 STEC 
    Bus 005 Device 002: ID 0bda:5411 Realtek Semiconductor Corp. 
    Bus 005 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 001 Device 003: ID 12d1:0003 Huawei Technologies Co., Ltd. 
    Bus 001 Device 002: ID 0bda:5411 Realtek Semiconductor Corp. 
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 003 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    ```

3. Prepare the XML description file of the USB device. Before hot removing the device, ensure that the USB device is not in use. Otherwise, data may be lost.
4. Run the hot swapping commands.

    Take a VM whose name is  **openEulerVM**  as an example. The corresponding configuration file is  **usb.xml**.

    - Hot adding of the USB device takes effect only for the current running VM. After the VM is restarted, hot add the USB device again.

        ```shell
        # virsh attach-device openEulerVM usb.xml --live
        ```

    - Complete persistency configurations for hot adding of the USB device. After the VM is restarted, the USB device is automatically assigned to the VM.

        ```shell
        # virsh attach-device openEulerVM usb.xml --config
        ```

    - Hot removing of the USB device takes effect only for the current running VM. After the VM is restarted, the USB device with persistency configurations is automatically assigned to the VM.

        ```shell
        # virsh detach-device openEulerVM usb.xml --live
        ```

    - Complete persistency configurations for hot removing of the USB device.

        ```shell
        # virsh detach-device openEulerVM usb.xml --config
        ```

## Storing Snapshots

### Overview

The VM system may be damaged due to virus damage, system file deletion by mistake, or incorrect formatting. As a result, the system cannot be started. To quickly restore a damaged system, openEuler provides the storage snapshot function. openEuler can create a snapshot that records the VM status at specific time points without informing users (usually within a few seconds). The snapshot can be used to restore the VM to the status when the snapshots were taken. For example, a damaged system can be quickly restored with the help of snapshots, which improves system reliability.

> [!NOTE]NOTE   
> Currently, storage snapshots can be QCOW2 and RAW images only. Block devices are not supported.  

### Procedure

To create VM storage snapshots, perform the following steps:

1. Log in to the host and run the  **virsh domblklist**  command to query the disk used by the VM.

    ```shell
    # virsh domblklist openEulerVM
      Target   Source
     ---------------------------------------------
      vda      /mnt/openEuler-image.qcow2
    ```

2. Run the following command to create the VM disk snapshot  **openEuler-snapshot1.qcow2**:

    ```shell
    # virsh snapshot-create-as --domain openEulerVM --disk-only --diskspec vda,snapshot=external,file=/mnt/openEuler-snapshot1.qcow2 --atomic
    Domain snapshot 1582605802 created
    ```

3. Run the following command to query disk snapshots:

    ```shell
    # virsh snapshot-list openEulerVM
     Name         Creation Time               State
    ---------------------------------------------------------
     1582605802   2020-02-25 12:43:22 +0800   disk-snapshot
    ```

## Configuring Disk I/O Suspension

### Introduction

#### Overview

When a storage fault occurs (for example, the storage link is disconnected), the I/O error of the physical disk is sent to the VM front end through the virtualization layer. The VM receives the I/O error. As a result, the user file system in the VM may change to the read-only state. In this case, the VM needs to be restarted or the user needs to manually restore the file system, which brings extra workload to users.

In this case, the virtualization platform provides the disk I/O suspension capability. That is, when the storage device is faulty, the VM I/Os are suspended when being delivered to the host. During the suspension period, no I/O error is returned to the VM. In this way, the file system in the VM does not change to the read-only state due to I/O errors. Instead, the file system is suspended. At the same time, the VM backend retries I/Os based on the specified suspension interval. If the storage fault is rectified within the suspension period, the suspended I/Os can be flushed to disks, and the internal file system of the VM automatically recovers without restarting the VM. If the storage fault is not rectified within the suspension period, an error is reported to the VM and the user is notified.

#### Application Scenarios

A cloud disk that may cause storage plane link disconnection is used as the backend of the virtual disk.

#### Precautions and Restrictions

- Only virtio-blk or virtio-scsi virtual disks support disk I/O suspension.

- Generally, the backend of the virtual disk where the disk I/Os are suspended is the cloud disk whose storage plane link may be disconnected.

- Disk I/O suspension can be enabled for read and write I/O errors separately. The retry interval and timeout interval for read and write I/O errors of the same disk are the same.

- The disk I/O suspension retry interval does not include the actual read/write I/O overhead on the host. That is, the actual interval between two I/O retries is greater than the configured I/O error retry interval.

- Disk I/O suspension cannot distinguish the specific type of I/O errors (such as storage link disconnection, bad sector, or reservation conflict). As long as the hardware returns an I/O error, suspension is performed.

- When disk I/O suspension occurs, the internal I/Os of the VM are not returned, the system commands for accessing the disk, such as fdisk, are suspended, and the services that depend on the returned commands are suspended.

- When disk I/O suspension occurs, the I/Os cannot be flushed to the disk. As a result, the VM may not be gracefully shut down. In this case, you need to forcibly shut down the VM.

- When disk I/O suspension occurs, the disk data cannot be read. As a result, the VM cannot be restarted. In this case, you need to forcibly stop the VM, wait until the storage fault is rectified, and then restart the VM.

- After a storage fault occurs, the following problems cannot be solved although disk I/O suspension occurs:

    1. Failed to execute advanced storage features.

        Advanced features include: virtual disk hot swap, virtual disk creation, VM startup, VM shutdown, VM forcible shutdown, VM hibernation, VM wakeup, VM storage live migration, VM storage live migration cancellation, VM storage snapshot creation, VM storage snapshot combination, VM disk capacity query, online disk capacity expansion, virtual CD-ROM drive insertion, and CD-ROM drive ejection from the VM.

    2. Failed to execute the VM lifecycle.

- When a live migration is initiated for a VM configured with disk I/O suspension, the disk I/O suspension configuration must be the same as that of the source host in the XML configuration of the destination disk.

### Disk I/O Suspension Configuration

#### Using the QEMU CLI

To enable disk I/O suspension, configure `werror=retry` `rerror=retry` on the virtual disk device. To configure the retry policy, configure `retry_interval` and `retry_timeout`. `retry_interval` indicates the I/O retry interval. The value ranges from 0 to `MAX_LONG`, in milliseconds. If this parameter is not set, the default value 1000 ms is used. `retry_timeout` indicates the I/O retry timeout interval. The value ranges from 0 to `MAX_LONG`, in milliseconds. The value **0** indicates that no timeout occurs. If this parameter is not set, the default value **0** is used.

The disk I/O suspension configuration of the virtio-blk disk is as follows:

```shell
-drive file=/path/to/your/storage,format=raw,if=none,id=drive-virtio-disk0,cache=none,aio=native \
-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x6,\
drive=drive-virtio-disk0,id=virtio-disk0,write-cache=on,\
werror=retry,rerror=retry,retry_interval=2000,retry_timeout=10000
```

The disk I/O suspension configuration of the virtio-scsi disk is as follows:

```shell
-drive file=/path/to/your/storage,format=raw,if=none,id=drive-scsi0-0-0-0,cache=none,aio=native \
-device scsi-hd,bus=scsi0.0,channel=0,scsi-id=0,lun=0,\
device_id=drive-scsi0-0-0-0,drive=drive-scsi0-0-0-0,id=scsi0-0-0-0,write-cache=on,\
werror=retry,rerror=retry,retry_interval=2000,retry_timeout=10000
```

#### Using an XML Configuration File

To enable disk I/O suspension, configure `error_policy='retry'` `rerror_policy='retry'` in the disk XML configuration. Configure `retry_interval` and `retry_timeout`. retry_interval indicates the I/O retry interval. The value ranges from 0 to `MAX_LONG`, in milliseconds. If this parameter is not set, the default value 1000 ms is used. retry_timeout indicates the I/O retry timeout interval. The value ranges from 0 to `MAX_LONG`, in milliseconds. The value **0** indicates that no timeout occurs. If this parameter is not set, the default value **0** is used.

The disk I/O suspension XML configuration of the virtio-blk disk is as follows:

```xml
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native' error_policy='retry' rerror_policy='retry' retry_interval='2000' retry_timeout='10000'/>
  <source dev='/path/to/your/storage'/>
  <target dev='vdb' bus='virtio'/>
  <backingStore/>
</disk>
```

The disk I/O suspension XML configuration of the virtio-scsi disk is as follows:

```xml
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native' error_policy='retry' rerror_policy='retry' retry_interval='2000' retry_timeout='10000'/>
  <source dev='/path/to/your/storage'/>
  <target dev='sdb' bus='scsi'/>
  <backingStore/>
  <address type='drive' controller='0' bus='0' target='0' unit='0'/>
</disk>
```
