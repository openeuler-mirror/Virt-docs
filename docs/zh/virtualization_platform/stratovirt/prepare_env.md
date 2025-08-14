# 准备环境

## 使用说明

- StratoVirt仅支持运行于x86_64和AArch64处理器架构下并启动相同架构的Linux虚拟机。
- 建议在 openEuler 22.03 LTS及以上 版本编译、调测和部署该版本 StratoVirt。
- StratoVirt支持以非root权限运行。

## 环境要求

运行StratoVirt需要具备如下环境：

- /dev/vhost-vsock设备（用于实现mmio）
- nmap工具
- Kernel镜像和rootfs镜像

## 准备设备和工具

- StratoVirt运行需要实现mmio设备，所以运行之前确保存在设备`/dev/vhost-vsock`

  查看该设备是否存在：

  ```shell
  $ ls /dev/vhost-vsock
  /dev/vhost-vsock
  ```

  若该设备不存在，请执行如下命令生成/dev/vhost-vsock设备。

  ```shell
  $ modprobe vhost_vsock
  ```

- 为了能够使用QMP命令，需要安装nmap工具，在配置yum源的前提下，可执行如下命令安装nmap。

  ```shell
  # yum install nmap
  ```

## 准备镜像

### 制作kernel镜像

当前版本的StratoVirt仅支持x86_64和AArch64平台的PE格式内核镜像。此格式内核映像可通过以下方法生成。

1. 获取openEuler的kernel源代码，参考命令如下：

   ```shell
   $ git clone https://gitee.com/openeuler/kernel.git
   $ cd kernel
   ```

2. 查看并切换kernel的版本到openEuler-22.03-LTS，参考命令如下：

   ```shell
   $ git checkout openEuler-22.03-LTS
   ```

3. 配置并编译Linux kernel。目前有两种方式可以生成配置文件：1. 使用推荐配置（[获取配置文件](https://gitee.com/openeuler/stratovirt/tree/master/docs/kernel_config)），将指定版本的推荐文件复制到kernel路径下并重命名为`.config`， 并执行命令`make olddefconfig`更新到最新的默认配置（否则后续编译可能有选项需要手动选择）。2. 通过以下命令进行交互，根据提示完成kernel配置，可能会提示缺少指定依赖，按照提示使用`yum install`命令进行安装。

   ```shell
   $ make menuconfig
   ```

4. 使用下面的命令制作并转换kernel镜像为PE格式，转化后的镜像为vmlinux.bin。

   ```shell
   $ make -j vmlinux && objcopy -O binary vmlinux vmlinux.bin
   ```

5. 如果想在x86平台使用bzImage格式的kernel，可以使用如下命令进行编译。

   ```shell
   $ make -j bzImage
   ```

   ​

## 制作rootfs镜像

rootfs镜像是一种文件系统镜像，在StratoVirt启动时可以装载带有init的ext4格式的镜像。下面是制作ext4 rootfs镜像的简单方法。

1. 准备一个大小合适的文件（例如在/home中创建10GiB空间大小的文件）。

   ```shell
   $ cd /home
   $ dd if=/dev/zero of=./rootfs.ext4 bs=1G count=10
   ```

2. 在此文件上创建空的ext4文件系统。

   ```shell
   $ mkfs.ext4 ./rootfs.ext4
   ```

3. 挂载文件镜像。创建/mnt/rootfs，使用root权限，将rootfs.ext4挂载到/mnt/rootfs目录。

   ```shell
   $ mkdir /mnt/rootfs
   # 返回刚刚创建文件系统的目录（如/home）
   $ cd /home
   $ sudo mount ./rootfs.ext4 /mnt/rootfs && cd /mnt/rootfs
   ```

4. 获取对应处理器架构的最新alpine-mini rootfs。

    - 对于AArch64处理器架构，从[alpine](http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/)网站获取最新alpine-mini rootfs，例如：alpine-minirootfs-3.21.0-aarch64.tar.gz ，参考命令如下：

    ```shell
    $ wget http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/alpine-minirootfs-3.21.0-aarch64.tar.gz
    $ tar -zxvf alpine-minirootfs-3.21.0-aarch64.tar.gz
    $ rm alpine-minirootfs-3.21.0-aarch64.tar.gz
    ```

    - 对于x86_64处理器架构，从[alpine](http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/)网站获取指定架构最新alpine-mini rootfs，例如：alpine-minirootfs-3.21.0-x86_64.tar.gz，参考命令如下：

    ```shell
    $ wget http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-minirootfs-3.21.0-x86_64.tar.gz
    $ tar -zxvf alpine-minirootfs-3.21.0-x86_64.tar.gz
    $ rm alpine-minirootfs-3.21.0-x86_64.tar.gz
    ```

5. 为ext4文件镜像制作一个简单的/sbin/init，参考命令如下：

   ```shell
   $ rm sbin/init; touch sbin/init && cat > sbin/init <<EOF
   #! /bin/sh
   mount -t devtmpfs dev /dev
   mount -t proc proc /proc
   mount -t sysfs sysfs /sys
   ip link set up dev lo
   
   exec /sbin/getty -n -l /bin/sh 115200 /dev/ttyS0
   poweroff -f
   EOF
   
   sudo chmod +x sbin/init
   ```

6. 卸载rootfs镜像。

   ```shell
   $ cd /home; umount /mnt/rootfs
   ```

   至此， rootfs制作成功，已可使用ext4 rootfs镜像文件rootfs.ext4，该文件在/home目录下。

## 获取标准启动所需固件

固件 (firmware) 是指设备内部保存的设备驱动程序。操作系统只有通过固件才能按照标准启动的方式进行启动。 StratoVirt 当前只支持在 x86_64 和 aarch64 架构下按照 UEFI (Unified Extensible Firmware Interface) 接口进行标准启动。

EDK II 是实现了 UEFI 标准的开源软件，StratoVirt 使用 EDK II 作为标准启动的固件。因此需要获取EDK II的固件文件。可以通过 yum 命令来进行 EDK II 固件文件的安装，具体命令如下。

x86_64 架构上运行以下命令：

```shell
$ sudo yum install -y edk2-ovmf
```

aarch64 架构上运行以下命令：

```shell
$ sudo yum install -y edk2-aarch64
```

EDK II 的固件包括两个文件：一个文件用于保存可执行代码，另一个文件用于保存启动配置信息。安装成功之后，在 x86_64 架构上，固件文件 `OVMF_CODE.fd` 与固件配置文件 `OVMF_VARS.fd` 会保存在 `/usr/share/edk2/ovmf` 路径下；在 aarch64 架构上, 固件文件 `QEMU_EFI-pflash.raw` 和固件配置文件 `vars-template-pflash.raw` 则是保存在 `/usr/share/edk2/aarch64` 路径下。
