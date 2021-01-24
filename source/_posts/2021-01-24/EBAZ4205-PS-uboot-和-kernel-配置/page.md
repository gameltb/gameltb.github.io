---
title: EBAZ4205 PS uboot 和 kernel 配置
date: 2021-01-24 13:04:49
tags: zynq uboot linux
---

&emsp;&emsp;记录一下 EBAZ4205 uboot 和 kernel 的配置过程。

## 准备

&emsp;&emsp;使用的 vivado 版本为 2020.2。  
&emsp;&emsp;首先将工具链的路径 export 到 PATH

```sh
export PATH=/tools/Xilinx/Vitis/2020.2/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin/:$PATH
export PATH=/tools/Xilinx/Vitis/2020.2/bin:$PATH
```

&emsp;&emsp;clone 以下 repo 到合适的地方。

- uboot https://github.com/Xilinx/u-boot-xlnx
- 内核 https://github.com/Xilinx/linux-xlnx

&emsp;&emsp;我们基本无需改动这些代码，只需要写好设备树就可以了。  
&emsp;&emsp;创建 arch/arm/dts/zynq-EBAZ4205.dts，定义好设备。  
&emsp;&emsp;这里有一份[我的](#dts)，参考[link](https://hhuysqt.github.io/zynq2/)。  
&emsp;&emsp;打开 arch/arm/dts/Makefile ，将我们的 zynq-EBAZ4205.dts 照葫芦画瓢加在 zynq-zc702.dtb 旁边。

```sh
make xilinx_zynq_virt_defconfig
DEVICE_TREE=zynq-EBAZ4205 make -j8
```

- 去掉 xilinx_zynq_virt_defconfig 里的 CONFIG_ENV_IS_IN_SPI_FLASH=y

&emsp;&emsp;linux-kernel 的设备树和 uboot 相同，放在 arch/arm/boot/dts/zynq-EBAZ4205.dts ，和上面差不多改下 arch/arm/boot/dts/Makefile 就可以了。

```sh
make xilinx_zynq_defconfig
make uImage UIMAGE_LOADADDR=0x8000
make dtbs
```

&emsp;&emsp; 得到 arch/arm/boot/uImage 和 arch/arm/boot/dts/zynq-EBAZ4205.dtb 这两个我们启动需要的文件。

## JTAG 启动

&emsp;&emsp;不想老拔插 sd 卡只好用 jtag 启动了，顺便还可以调试。  
&emsp;&emsp;我们可以用 xsct 来将 uboot.elf 文件通过 jtag 下载到板子的内存里，不过直接执行是有问题的，通过 emio 连接的 phy 会找不到。需要先执行 fsbl 里的初始化部分。  
&emsp;&emsp;这里使用 vivado 导出的 xsa 文件包含的 ps7_init.tcl 脚本通过 jtag 来初始化。正常独立启动时是由编译出的 fsbl 来做的。

```tcl
connect
targets -set -nocase -filter {name =~"APU*"}
rst -system
after 3000
fpga -file ../vitis/EBAZ4205_wrapper/export/EBAZ4205_wrapper/hw/EBAZ4205_wrapper.bit
loadhw -hw ../vitis/EBAZ4205_wrapper/export/EBAZ4205_wrapper/hw/EBAZ4205_wrapper.xsa -mem-ranges [list {0x40000000 0xbfffffff}] -regs
configparams force-mem-access 1
source ../vitis/EBAZ4205_wrapper/export/EBAZ4205_wrapper/hw/ps7_init.tcl
ps7_init
ps7_post_config
targets -set -nocase -filter {name =~ "*A9*#0"}
dow ../petalinux/u-boot-xlnx/u-boot.elf
configparams force-mem-access 0
con
```

&emsp;&emsp;调试时在 vitis 新建一个调试配置，将脚本指定到上面那个文件就可以了，你可能需要修改一下文件的路径，如果需要在入口停住则去掉最后的 con。 uboot 会进行一次重定位，因此使用断点需要在配置里用带符号的文件 u-boot 设置好两个地址。

## tftp kernel

&emsp;&emsp;rootfs 从 [link](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842473/Build+and+Modify+a+Rootfs) 下载

```sh
mkimage -A arm -T ramdisk -C gzip -d arm_ramdisk.image.gz uramdisk.image.gz
```

&emsp;&emsp;tftp 服务器有很多实现这里我使用 dnsmasq 附带的。编辑/etc/dnsmasq.conf

```conf
listen-address=192.168.1.1
enable-tftp
tftp-root=/srv/tftp
```

&emsp;&emsp;配置上面这几行就可以。将 uImage，zynq-EBAZ4205.dtb 还有 rootfs 放到/srv/tftp 下，启动

```sh
systemctl start dnsmasq
```

&emsp;&emsp;通过 uart 连接板子执行

```sh
setenv ipaddr 192.168.1.10
setenv gatewayip 192.168.1.1
setenv netmask 255.255.255.0
setenv serverip 192.168.1.1

tftpboot 0 zynq-EBAZ4205.dtb
tftpboot 8000 uImage
tftpboot 1000000 uramdisk.image.gz
bootm 8000 1000000 0
```

&emsp;&emsp;不出意外就能进入 shell 了。

- repo 地址 https://github.com/gameltb/EBAZ4205

## dts

```dts
// SPDX-License-Identifier: GPL-2.0+
/*
 *  Copyright (C) 2011 - 2015 Xilinx
 *  Copyright (C) 2012 National Instruments Corp.
 */
/dts-v1/;
#include "zynq-7000.dtsi"

/ {
    model = "EBAZ4205 board";
    compatible = "xlnx,zynq-EBAZ4205", "xlnx,zynq-7000";

    aliases {
        ethernet0 = &gem0;
        serial0 = &uart1;
        mmc0 = &sdhci0;
    };

    memory@0 {
        device_type = "memory";
        reg = <0x0 0x10000000>;
    };

    chosen {
        bootargs = "";
        stdout-path = "serial0:115200n8";
    };

    gpio-keys {
        compatible = "gpio-keys";
        autorepeat;

        s2 {
            label = "s2";
            gpios = <&gpio0 20 1>;
            linux,code = <102>;
            wakeup-source;
            autorepeat;
        };

        s3 {
            label = "s3";
            gpios = <&gpio0 32 1>;
            linux,code = <116>;
            wakeup-source;
            autorepeat;
        };
    };
};

&clkc {
    ps-clk-frequency = <33333333>;
};

&gem0 {
    status = "okay";
    phy-mode = "rgmii-id";
    phy-handle = <&ethernet_phy>;

    ethernet_phy: ethernet-phy@0 {
        reg = <0>;
        device_type = "ethernet-phy";
    };
};

&smcc {
    status = "okay";
};

&nand0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_nand0_default>;
    partition@0 {
        label = "nand-fsbl-uboot";
        reg = <0x0 0x300000>;
    };
    partition@1 {
        label = "nand-linux";
        reg = <0x300000 0x500000>;
    };
    partition@2 {
        label = "nand-device-tree";
        reg = <0x800000 0x20000>;
    };
    partition@3 {
        label = "nand-rootfs";
        reg = <0x820000 0xa00000>;
    };
    partition@4 {
        label = "nand-jffs2";
        reg = <0x1220000 0x1000000>;
    };
    partition@5 {
        label = "nand-bitstream";
        reg = <0x2220000 0x800000>;
    };
    partition@6 {
        label = "nand-allrootfs";
        reg = <0x2a20000 0x4000000>;
    };
    partition@7 {
        label = "nand-release";
        reg = <0x6a20000 0x13e0000>;
    };
    partition@8 {
        label = "nand-reserve";
        reg = <0x7e00000 0x200000>;
    };
};

&pinctrl0 {
    pinctrl_nand0_default: nand0-default {
        mux {
            groups = "smc0_nand8_grp";
            function = "smc0_nand";
        };

        conf {
            groups = "smc0_nand8_grp";
            bias-pull-up;
        };
    };

    pinctrl_uart1_default: uart1-default {
        mux {
            groups = "uart1_4_grp";
            function = "uart1";
        };

        conf {
            groups = "uart1_4_grp";
            slew-rate = <0>;
            io-standard = <3>;
        };
    };
};

&sdhci0 {
    u-boot,dm-pre-reloc;
    status = "okay";
};

&uart1 {
    u-boot,dm-pre-reloc;
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_uart1_default>;
};
```
