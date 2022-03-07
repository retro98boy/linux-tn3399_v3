# U-Boot的编译
参考[lanseyujie](https://github.com/lanseyujie/tn3399_v3)的教程

首先去[这里](https://github.com/ARM-software/arm-trusted-firmware/tags)下载最新版的ATF，解压并cd进入，然后执行make CROSS_COMPILE=aarch64-linux-gnu- PLAT=rk3399编译，最后设置下环境变量：export BL31=path_to_your_bl31.elf

选择使用主线U-Boot，去U-Boot的Github [tag界面](https://github.com/u-boot/u-boot/tags)下载v2022.01版U-Boot，解压，cd进入
将arch/arm/dts/rk3399-firefly.dts的内容替换成TN3399_V3的主线dts，接着执行make firefly-rk3399_defconfig&&make CROSS_COMPILE=aarch64-linux-gnu- -j12完成编译，目录下u-boot-rockchip.bin就是我们需要的，将其烧录到img镜像或者tf卡的32k偏移处即可，注意使用dd烧录到img镜像时，要加上conv=notrunc
之所以选择在firefly-rk3399上修改而不是其他板子，是因为使用了和TN3399_V3相同的ddr3，可以从arch/arm/dts/rk3399-firefly-u-boot.dtsi中看出，如果使用其他lpddr板子的模板，会导致U-Boot初始化内存失败，除非添加rockchip的miniloader bin，那样就不算真正意义上的纯主线了

# kernel的编译
只需将TN3399_V3的dts添加到arch/arm64/boot/dts/rockchip/下，并修改Makefile添加TN3399_V3的选项即可。BSP内核推荐[mrfixit2001](https://github.com/mrfixit2001/rockchip-kernel)维护的版本

# 发行版的构建
Ubuntu可以从[这里](http://mirrors.ustc.edu.cn/ubuntu-cdimage/ubuntu-base/releases/)下载rootfs，然后解压对其进行自定义，例如添加用户名，修改主机名时区，预装安装软件，修改配置文件等等。最后将之前编译的内核镜像和模块放到rootfs中，选择一种引导方式，如extlinux或者U-Boot script即可。再使用dd命令创建空白镜像，进行分区，放入rootfs，刻录U-Boot blob，可引导的系统镜像就做好了
使用x86主机运行arm的rootfs需要安装qemu-user-static，如果是ArchLinux，还需要安装binfmt，不像Ubuntu在安装qemu-user-static会自动安装binfmt。同时建议使用systemd-nspawn来替代chroot来挂载rootfs
这种构建系统镜像的方法每次当内核更新时，就只能再重新编译来更新内核，所以我更愿意在已有的系统镜像上进行修改，例如下载rockpro64的Manjaro系统，使用losetup挂载img，替换dtb，做些修改，例如使用vim代替nano，删除ntfs-3g使用ntfs3，卸载rockpro64的U-Boot包，最后将TN3399_V3的U-Boot刻录到img上，就可以一直yay -Syyu享受最新的内核了

# wifi
AP6255需要固件才能正常运行，将主线内核使用的一些固件放到/lib/firmware/brcm下，BSP内核的文件放到/lib/firmware下，BSP内核还需要正确配置Kconfig，可以参考rk3399_linux_ap6xxx_defconfig
