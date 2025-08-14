# 安装虚拟化组件

本章介绍在openEuler中安装虚拟化组件的方法。

## 最低硬件要求

在openEuler系统中安装虚拟化组件，最低硬件要求：

- AArch64处理器架构：ARMv8以上并且支持虚拟化扩展
- x86\_64处理器架构：支持VT-x
- 2核CPU
- 4GB的内存
- 16GB可用磁盘空间

## 安装虚拟化核心组件

### 安装方法

#### 前提条件

- 已经配置yum源。配置方式请参见《openEuler 25.03 管理员指南》。
- 安装操作需要root用户权限。

#### 安装步骤

1. 安装QEMU组件。

    ``` shell
    # yum install -y qemu
    ```

    > [!WARNING]注意
    >
    > QEMU组件默认以用户qemu和用户组qemu运行，如果您不了解Linux用户组、用户的权限管理，后续创建和启动虚拟机时可能会遇到权限不足问题，以下有两种解决方法：  
    > 第一种方法：修改QEMU配置文件。使用以下命令打开QEMU配置文件，`sudo vim /etc/libvirt/qemu.conf`，找到以下两个字段，`user = "root"`和`group = "root"`，取消注释（即删除前面的`#`号），保存并退出。  
    > 第二种方法：修改虚拟机文件的所有者。首先需要保证用户qemu拥有访问存放虚拟机文件的文件夹权限，使用以下命令修改文件的所有者，`sudo chown qemu:qemu xxx.qcow2`，修改所有需要读写的虚拟机文件即可。  

2. 安装libvirt组件。

    ``` shell
    # yum install -y libvirt
    ```

3. 启动libvirtd服务。

    ``` shell
    # systemctl start libvirtd
    ```

> [!NOTE]说明
>
> KVM模块已经集成在openEuler内核中，因此不需要单独安装。  

### 验证安装是否成功

1. 查看内核是否支持KVM虚拟化，即查看/dev/kvm和/sys/module/kvm文件是否存在，命令和回显如下：

    ``` shell
    $ ls /dev/kvm
    /dev/kvm
    ```

    ``` shell
    $ ls /sys/module/kvm
    parameters  uevent
    ```

    若上述文件存在，说明内核支持KVM虚拟化。若上述文件不存在，则说明系统内核编译时未开启KVM虚拟化，需要更换支持KVM虚拟化的Linux内核。

2. 确认QEMU是否安装成功。若安装成功则可以看到QEMU软件包信息，命令和回显如下：

    ``` shell
    $ rpm -qi qemu
    Name        : qemu
    Epoch       : 2
    Version     : 4.0.1
    Release     : 10
    Architecture: aarch64
    Install Date: Wed 24 Jul 2019 04:04:47 PM CST
    Group       : Unspecified
    Size        : 16869484
    License     : GPLv2 and BSD and MIT and CC-BY
    Signature   : (none)
    Source RPM  : qemu-4.0.0-1.src.rpm
    Build Date  : Wed 24 Jul 2019 04:03:52 PM CST
    Build Host  : localhost
    Relocations : (not relocatable)
    URL         : http://www.qemu.org
    Summary     : QEMU is a generic and open source machine emulator and virtualizer
    Description :
    QEMU is a generic and open source processor emulator which achieves a good
    emulation speed by using dynamic translation. QEMU has two operating modes:
    
     * Full system emulation. In this mode, QEMU emulates a full system (for
       example a PC), including a processor and various peripherals. It can be
       used to launch different Operating Systems without rebooting the PC or
       to debug system code.
     * User mode emulation. In this mode, QEMU can launch Linux processes compiled
       for one CPU on another CPU.
    
    As QEMU requires no host kernel patches to run, it is safe and easy to use.
    ```

3. 确认libvirt是否安装成功。若安装成功则可以看到libvirt软件包信息，命令和回显如下：

    ``` shell
    $ rpm -qi libvirt
    Name        : libvirt
    Version     : 5.5.0
    Release     : 1
    Architecture: aarch64
    Install Date: Tue 30 Jul 2019 04:56:21 PM CST
    Group       : Unspecified
    Size        : 0
    License     : LGPLv2+
    Signature   : (none)
    Source RPM  : libvirt-5.5.0-1.src.rpm
    Build Date  : Mon 29 Jul 2019 08:14:57 PM CST
    Build Host  : 71e8c1ce149f
    Relocations : (not relocatable)
    URL         : https://libvirt.org/
    Summary     : Library providing a simple virtualization API
    Description :
    Libvirt is a C toolkit to interact with the virtualization capabilities
    of recent versions of Linux (and other OSes). The main package includes
    the libvirtd server exporting the virtualization support.
    ```

4. 查看libvirt服务是否启动成功。若服务处于“Active”状态，说明服务启动成功，可以正常使用libvirt提供的virsh命令行工具，命令和回显如下：

    ``` shell
    $ systemctl status libvirtd
    ● libvirtd.service - Virtualization daemon
       Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
       Active: active (running) since Tue 2019-08-06 09:36:01 CST; 5h 12min ago
         Docs: man:libvirtd(8)
               https://libvirt.org
     Main PID: 40754 (libvirtd)
        Tasks: 20 (limit: 32768)
       Memory: 198.6M
       CGroup: /system.slice/libvirtd.service
               ─40754 /usr/sbin/libvirtd
    
    ```
