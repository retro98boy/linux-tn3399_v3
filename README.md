# U-Boot的编译

参考[lanseyujie](https://github.com/lanseyujie/tn3399_v3)的教程

### 编译ATF

去[这里](https://github.com/ARM-software/arm-trusted-firmware/tags)下载最新版的ATF，解压并cd进入，然后执行make CROSS_COMPILE=aarch64-linux-gnu- PLAT=rk3399编译，最后设置下环境变量：export BL31=path_to_your_bl31.elf供下面编译U-Boot时使用

### 编译主线U-Boot

使用主线U-Boot，去U-Boot的Github [tag界面](https://github.com/u-boot/u-boot/tags)下载v2022.04版U-Boot，解压，打上本项目中提供的U-Boot源码patch，接着执行make tn3399_v3-rk3399_defconfig&&make CROSS_COMPILE=aarch64-linux-gnu- -j12完成编译，目录下u-boot-rockchip.bin就是我们需要的，将其烧录到img镜像或者tf卡的32k偏移处即可。**注意使用dd烧录到img镜像时，要加上conv=notrunc选项**

# kernel的编译

在kernel.org下载kernel源码。将TN3399_V3的mainline dts添加到arch/arm64/boot/dts/rockchip/下，并修改Makefile模仿添加TN3399_V3的选项，最后编译即可。内核的编译config可以选择默认的defconfig，也可以从Armbian rockchip64镜像中提取config。

如果要使用BSP内核，推荐[mrfixit2001](https://github.com/mrfixit2001/rockchip-kernel)维护的版本，但rk3399的主线内核似乎已经很完善了，包括GPU VOP等都有主线化的开源驱动了。连batocera项目34版本都转而使用主线kernel了。

# 发行版的构建

Ubuntu可以从[这里](http://mirrors.ustc.edu.cn/ubuntu-cdimage/ubuntu-base/releases/)下载rootfs，然后解压对其进行自定义，例如添加用户名，修改主机名、时区，预装安装软件，修改配置文件等等。最后将之前编译的内核镜像和模块放到rootfs中，选择一种引导方式，如extlinux或者U-Boot script即可。再使用dd命令创建空白镜像，进行分区，放入rootfs，刻录U-Boot blob，可引导的系统镜像就做好了
使用x86主机运行arm的rootfs需要安装qemu-user-static，如果是ArchLinux，还需要安装binfmt，不像Ubuntu在安装qemu-user-static会自动安装binfmt。同时建议使用systemd-nspawn来替代chroot来挂载rootfs
这种构建系统镜像的方法每次当内核更新时，就只能再重新编译来更新内核，所以我更愿意在已有的系统镜像上进行修改，例如下载rockpro64的Manjaro系统，使用losetup挂载img，替换dtb，做些修改，例如使用vim代替nano，删除ntfs-3g使用ntfs3，卸载rockpro64的U-Boot包，最后将TN3399_V3的U-Boot刻录到img上，就可以一直yay -Syyu享受最新的内核了

**修改img镜像的大致操作过程：**

### 已有img系统镜像，需要修改

```
sudo losetup -f                                                                                                ─╯
# 输出/dev/loop0，说明/dev/loop0可以使用
sudo losetup path_to_your_img
# 将/dev/loop0和img关联起来
sudo partprobe /dev/loop0
# 刷新下/dev/loop0(img)的分区表
sudo mount /dev/loop0p2 /mnt && sudo mount /dev/loop0p1 /mnt/boot
# 不一定所以的分区表都一样，按需修改
sudo systemd-nspawn -D /mnt -M tmp
# chroo到img中的rootfs中，需要安装systemd-container qemu-user-static binfmt
# 对rootfs做些自定义，比如换源，添加预装软件等
# 退出systemd-nspawn，方式是按住 ctrl ，接着连按 ] 三次
sudo umount -R /mnt
# 卸载所有已挂载的img分区
sudo losetup -d /dev/loop0
# 取消/dev/loop0和img的关联
sudo dd if=path_to_uboot of=path_to_your_img bs=1k seek=32 conv=notrunc
# 给img烧录U-Boot，使img bootable，不同SoC厂商的偏移地址不一样，这里是rockchip SoC的
# 这样我们就完成了对img的编辑
```

### 自己创建系统img镜像

```
sudo dd if=/dev/zero of=~/myimg.img bs=1M count=2048 status=progress
# 创建空白文件，大小自己定
sudo parted ~/myimg.img mklabel gpt
# 给img创建gpt分区表
sudo cfdisk ~/myimg.img
# 给img分区，也可以使用fdisk parted等分区工具，分什么分区由自己定，记得第一个分区起始位置靠后些，前面给U-Boot留下空间
# 最后参考上面的操作，使用losetup挂载img,下载Ubuntu Ports的rootfs复制到img里，做些自定义，比较重要的有passwd sudo timezone locales hosts fstab
```

# WIFI

AP6255需要固件才能正常运行，将主线内核使用的一些固件放到/lib/firmware/brcm下。~~BSP内核的文件放到/lib/firmware下，BSP内核还需要正确配置Kconfig，可以参考rk3399_linux_ap6xxx_defconfig~~  [mrfixit2001](https://github.com/mrfixit2001/rockchip-kernel)维护的BSP内核已经修复了defconfig配置错误导致WIFI不能用的问题
