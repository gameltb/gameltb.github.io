---
title: 折腾kvm虚拟机
date: 2019-12-24 09:01:34
tags: kvm
---

- 随笔可能有错误，请多包涵。
- 如果有时间可以到[repo](https://github.com/gameltb/gameltb.github.io/issues)下提 issue，谢谢。

# 起

最近想折腾一下 FPGA，需要安装 xilinx 的一套工具。这堆东西真的大。自己想了想，为了稳定性还有使用上的方便。决定将这一套工具安装到虚拟机内，虚拟机还能有快照。于是看了下 linux 下的虚拟机。  
最初想用 lxc 容器，它靠内核来隔离用户空间，性能应该是最好的了，然而使用的还是和主机相同的内核，对于一些需要在内核上安装驱动的场景就不太合适了，弃之。  
最后还是用上了“传统”的虚拟机，毕竟有时候还要用到 windows 下的一些工具。先想到的是 vmware，virtualbox 这样的，但是 vmware 要钱，virtualbox 的图形似乎不太好，看 LTT 看到 linux 下的 qemu+kvm 似乎性能不错，就找几篇 wiki 看了一下。才发现功能强大，性能损失可以到相当低的地步，内存也可以在运行时动态分配，自己火星了。想来很早之前用 android sdk 时用的模拟器性能差劲，导致对 qemu 印象不好，一直没用上这一套。用了好长一段时间双系统，切换是真的麻烦。

## 图形

对于虚拟机的图形来说，在合适的硬件上可以让显卡直通虚拟机，叫做 [PCI passthrough](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)，打游戏都不成问题。但是这样不太灵活，直通要求虚拟机独占一块显卡和显示器。不过也可以使用[ LTT 的视频](https://www.bilibili.com/video/av68273308)中用的办法来提高灵活性。  
如果使用的是 Intel 的核心显卡，你也可以使用由 Intel 开发的 iGVT-g 来获得满足轻度使用的性能，客户机可以使用未修改的 Intel 显卡驱动，兼容性也不错。  
通用一点的是 Virgil 3D，令虚拟机使用主机的渲染节点，不过 windows 的驱动似乎还不太完善，性能个人没有测试，从搜索结果来看，似乎还不是特别好。

- QEMU/Guest graphics acceleration  
  https://wiki.archlinux.org/index.php/QEMU/Guest_graphics_acceleration

## 存储

大概要感谢云计算和虚拟化的发展，qemu/kvm 在存储上是非常灵活的，既可以使用文件，也可以使用硬盘分区，还可以加上 lvm，使用跨盘跨分区的逻辑卷，这样的逻辑卷本身就有快照功能。更绝的是对于支持的 linux 客户机，使用 9p 或者新出的性能更好的 virtio-fs 可将 root 挂在在主机的文件夹中，这样不会有虚拟磁盘有可能产生的空间浪费，文件共享也更加轻松。不过这样没有快照，你可能需要手动配置 OverlayFS 来做到差不多的效果，这样的共享还可能会导致更多的漏洞。  
我自己没有使用 lvm，又没有在使用前做好功课，使用 ext4 文件系统下的虚拟磁盘文件+ovmf uefi 启动，由于现在的（2019-12）libvirt 对 uefi 启动的虚拟机快照支持不完善，导致快照使用起来有点麻烦，叹气。

## 动态内存分配

在设置好最大可以占用内存后，可以在虚拟机开机时动态调整实际占用的内存大小，要注意最大值不能在运行时调整。windows 客户机需要按照下面这个说明安装驱动。这个特性对日常桌面应用应该是提高了不少可用性。

- Windows VirtIO 驱动  
  https://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers

自动分配，启用之后感觉客户机吃内存时变卡了些。

- https://www.linux-kvm.org/page/Projects/auto-ballooning
- 使用 libvirt 时的配置方法  
  https://libvirt.org/formatdomain.html#elementsMemBalloon

## 最终想要的效果

三大操作系统程序都能跑，性能损耗不大，存储空间和内存浪费小，互相隔离又有快照，还能轻松迁移。甚至可以跑个 androidx86，，，感觉总算得到了一个个人比较满意的环境。

# 糖炒栗子

下面用我的 arch 主机搭建一个用于 xilinx 工具的 ubuntu 18.04.3 lts 虚拟机。

## 准备

安装

```shell
sudo pacman -S qemu virt-manager
```

这里我选择 virt-manager 为图形前端方便使用。  
你也可以通过命令使用 qemu 来直接运行虚拟机。

### 准备系统镜像

下载 ubuntu 之类的发行版选择一个速度快的镜像站点下载即可。

# 新建虚拟机

执行

```
virt-manager
```

添加连接 QEMU/KVM ，然后新建虚拟机，按照提示一步步来即可。在最后一步选择 在安装前自定义配置 ，在此时我们就可以做一些调整。

# 调整

一些配置需要编辑 xml 文件，请先在虚拟系统管理器的首选项中启用 XML 编辑。
我们可以删除一些用不到的设备例如数位板，这样可以提高性能。详细配置可以参考下面两个链接。

- libvirt 配置文件说明  
  https://libvirt.org/formatdomain.html
- 虚拟化调试和优化指南  
  https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html-single/virtualization_tuning_and_optimization_guide/index

## 减少磁盘空间浪费

- https://chrisirwin.ca/posts/discard-with-kvm/

## 自动调整使用的内存

- https://libvirt.org/formatdomain.html#elementsMemBalloon

## 图形

显卡类型改为 virtio ，勾选 3d acceleration。  
显示协议监听类型改为无，勾选 OpenGL。

轻度使用这样就可以了。
